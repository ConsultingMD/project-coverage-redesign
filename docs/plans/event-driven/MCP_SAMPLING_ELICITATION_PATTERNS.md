# MCP Sampling & Elicitation Patterns for Digital Twin

## Executive Summary

This document describes how the Digital Twin MCP implementation can leverage **sampling** (intelligent resource selection) and **elicitation** (guided information gathering) to create more effective AI agent interactions and improve member care outcomes.

---

## Core Concepts

### Sampling in MCP Context
**Sampling** refers to intelligently selecting which resources to include in context based on relevance, priority, and token limits.

### Elicitation in MCP Context
**Elicitation** refers to systematically gathering information through the `chat` tool, using structured prompts and progressive refinement.

---

## 1. Resource Sampling Patterns

### Problem: Context Window Limitations

AI agents have limited context windows. A member might have thousands of resources, but we can only include the most relevant ones.

### Solution: Intelligent Sampling via Annotations

The MCP Resources specification supports annotations for priority and audience:

```json
{
  "uri": "mcp://twins/member/M123/clinical/medications",
  "name": "Current Medications",
  "annotations": {
    "priority": 0.95,  // Very high priority
    "audience": ["assistant"],
    "lastModified": "2025-01-15T10:00:00Z",
    "relevance": {
      "conditions": ["diabetes", "hypertension"],
      "recency_weight": 0.8
    }
  }
}
```

### Sampling Strategies

#### 1. Priority-Based Sampling

```typescript
// Digital Twin SDK implementation
class ResourceSampler {
  async sampleForContext(
    memberId: string,
    maxTokens: number = 8000
  ): Promise<Resource[]> {
    // Get all available resources
    const allResources = await this.listResources(memberId);
    
    // Sort by priority (1.0 = most important)
    const sorted = allResources.sort((a, b) => 
      (b.annotations?.priority || 0) - (a.annotations?.priority || 0)
    );
    
    // Sample until token limit
    const sampled: Resource[] = [];
    let tokenCount = 0;
    
    for (const resource of sorted) {
      const estimatedTokens = this.estimateTokens(resource);
      if (tokenCount + estimatedTokens > maxTokens) break;
      
      sampled.push(resource);
      tokenCount += estimatedTokens;
    }
    
    return sampled;
  }
}
```

#### 2. Recency-Weighted Sampling

```typescript
class RecencySampler {
  sampleRecent(resources: Resource[], options: {
    recencyWeight: number,  // 0-1, how much to weight recency
    maxAge: Duration,       // Ignore older than this
    targetCount: number     // How many to sample
  }): Resource[] {
    const now = new Date();
    
    return resources
      .filter(r => {
        const age = now - new Date(r.annotations?.lastModified);
        return age < options.maxAge;
      })
      .map(r => ({
        ...r,
        score: this.calculateScore(r, now, options.recencyWeight)
      }))
      .sort((a, b) => b.score - a.score)
      .slice(0, options.targetCount);
  }
  
  private calculateScore(resource: Resource, now: Date, recencyWeight: number): number {
    const priority = resource.annotations?.priority || 0.5;
    const age = now - new Date(resource.annotations?.lastModified);
    const recencyScore = 1 / (1 + age.days());  // Decay function
    
    return (priority * (1 - recencyWeight)) + (recencyScore * recencyWeight);
  }
}
```

#### 3. Relevance-Based Sampling

```typescript
class RelevanceSampler {
  async sampleByRelevance(
    memberId: string,
    context: {
      currentConditions: string[],
      recentSymptoms: string[],
      upcomingAppointments: string[]
    }
  ): Promise<Resource[]> {
    const resources = await this.listResources(memberId);
    
    return resources
      .map(r => ({
        ...r,
        relevanceScore: this.calculateRelevance(r, context)
      }))
      .filter(r => r.relevanceScore > 0.3)  // Threshold
      .sort((a, b) => b.relevanceScore - a.relevanceScore);
  }
  
  private calculateRelevance(resource: Resource, context: any): number {
    let score = 0;
    
    // Check condition overlap
    const resourceConditions = resource.annotations?.relevance?.conditions || [];
    const conditionOverlap = this.jaccardSimilarity(
      resourceConditions,
      context.currentConditions
    );
    score += conditionOverlap * 0.5;
    
    // Check temporal relevance
    if (resource.uri.includes('/appointments')) {
      const hasUpcoming = context.upcomingAppointments.length > 0;
      score += hasUpcoming ? 0.3 : 0;
    }
    
    // Check symptom relevance
    if (resource.uri.includes('/clinical')) {
      const symptomMatch = this.checkSymptomRelevance(
        resource,
        context.recentSymptoms
      );
      score += symptomMatch * 0.2;
    }
    
    return Math.min(1.0, score);
  }
}
```

---

## 2. Information Elicitation Patterns

### The Chat Tool as Elicitation Interface

The Digital Twin's `chat` tool can systematically elicit information:

```typescript
interface ElicitationSession {
  sessionId: string;
  memberId: string;
  goal: string;  // What we're trying to learn
  strategy: ElicitationStrategy;
  state: ElicitationState;
}

enum ElicitationStrategy {
  PROGRESSIVE_REFINEMENT,  // Start broad, narrow down
  CHECKLIST_COMPLETION,    // Complete a form/checklist
  SYMPTOM_EXPLORATION,     // Explore symptoms systematically
  PREFERENCE_LEARNING      // Learn member preferences
}
```

### Elicitation Strategies

#### 1. Progressive Refinement Pattern

```typescript
class ProgressiveRefinementElicitor {
  async elicit(session: ElicitationSession): Promise<ElicitationResult> {
    // Start with broad question
    const initial = await this.chat({
      message: "How have you been feeling this week?",
      intents: [{
        name: "gather.general.wellbeing",
        expectedResponses: ["good", "okay", "not well", "pain", "tired"]
      }]
    });
    
    // Narrow based on response
    if (initial.sentiment === 'negative' || initial.keywords.includes('pain')) {
      const followup = await this.chat({
        message: "I'm sorry to hear that. Can you tell me more about the pain? Where is it located?",
        intents: [{
          name: "gather.pain.location",
          expectedResponses: ["head", "chest", "back", "joints", "abdomen"]
        }]
      });
      
      // Further refinement
      if (followup.entities.bodyPart) {
        const detailed = await this.chat({
          message: `On a scale of 1-10, how would you rate your ${followup.entities.bodyPart} pain?`,
          intents: [{
            name: "gather.pain.severity",
            expectedType: "number",
            validation: { min: 1, max: 10 }
          }]
        });
        
        // Store structured data
        await this.updateResource({
          uri: `mcp://twins/member/${session.memberId}/clinical/symptoms`,
          data: {
            timestamp: new Date(),
            symptom: "pain",
            location: followup.entities.bodyPart,
            severity: detailed.value,
            elicitedVia: session.sessionId
          }
        });
      }
    }
    
    return this.completeElicitation(session);
  }
}
```

#### 2. Checklist Completion Pattern

```typescript
class ChecklistElicitor {
  async completePHQ9(memberId: string): Promise<PHQ9Result> {
    const questions = [
      "Over the last 2 weeks, how often have you had little interest or pleasure in doing things?",
      "Over the last 2 weeks, how often have you felt down, depressed, or hopeless?",
      "Over the last 2 weeks, how often have you had trouble falling asleep, staying asleep, or sleeping too much?",
      // ... rest of PHQ-9 questions
    ];
    
    const responses: number[] = [];
    
    for (const [index, question] of questions.entries()) {
      const response = await this.chat({
        message: question,
        intents: [{
          name: `phq9.question.${index + 1}`,
          expectedResponses: [
            "Not at all (0)",
            "Several days (1)", 
            "More than half the days (2)",
            "Nearly every day (3)"
          ],
          requireSelection: true
        }]
      });
      
      responses.push(response.score);
      
      // Adaptive - if score is high, add crisis resources
      if (response.score >= 2 && index <= 1) {
        await this.addContext({
          uri: "mcp://twins/member/crisis-resources",
          priority: 1.0
        });
      }
    }
    
    const totalScore = responses.reduce((a, b) => a + b, 0);
    
    // Store assessment
    await this.createResource({
      uri: `mcp://twins/member/${memberId}/assessments/phq9/${Date.now()}`,
      data: {
        responses,
        totalScore,
        severity: this.calculateSeverity(totalScore),
        completedAt: new Date(),
        nextDue: this.calculateNextDue(totalScore)
      }
    });
    
    return { responses, totalScore };
  }
}
```

#### 3. Symptom Exploration Pattern

```typescript
class SymptomExplorer {
  async explore(memberId: string, initialSymptom: string): Promise<SymptomProfile> {
    const profile: SymptomProfile = {
      primary: initialSymptom,
      characteristics: {},
      associatedSymptoms: [],
      timeline: {}
    };
    
    // Use OPQRST framework for systematic exploration
    const opqrst = {
      onset: await this.elicitOnset(initialSymptom),
      provocation: await this.elicitProvocation(initialSymptom),
      quality: await this.elicitQuality(initialSymptom),
      region: await this.elicitRegion(initialSymptom),
      severity: await this.elicitSeverity(initialSymptom),
      time: await this.elicitTiming(initialSymptom)
    };
    
    // Check for red flags based on symptom + characteristics
    const redFlags = await this.checkRedFlags(initialSymptom, opqrst);
    
    if (redFlags.length > 0) {
      // Immediately escalate
      await this.escalate({
        memberId,
        symptom: initialSymptom,
        redFlags,
        urgency: 'immediate'
      });
    }
    
    // Store structured symptom data
    await this.updateResource({
      uri: `mcp://twins/member/${memberId}/clinical/current-symptoms`,
      data: {
        ...profile,
        opqrst,
        elicitedAt: new Date(),
        redFlags
      }
    });
    
    return profile;
  }
  
  private async elicitOnset(symptom: string): Promise<OnsetInfo> {
    const response = await this.chat({
      message: `When did your ${symptom} start? Was it sudden or gradual?`,
      intents: [{
        name: "symptom.onset",
        extractEntities: ["timeframe", "onset_type"]
      }]
    });
    
    return {
      timeframe: response.entities.timeframe,
      type: response.entities.onset_type,  // 'sudden' | 'gradual'
      raw: response.text
    };
  }
}
```

---

## 3. Adaptive Sampling Based on Elicitation

### Dynamic Context Adjustment

As we elicit information, we can dynamically adjust what resources to sample:

```typescript
class AdaptiveSampler {
  private currentContext: Resource[] = [];
  private elicitationHistory: ElicitationEvent[] = [];
  
  async adjustContext(
    memberId: string,
    elicitationResult: ElicitationResult
  ): Promise<void> {
    // Analyze what we learned
    const topics = this.extractTopics(elicitationResult);
    const concerns = this.extractConcerns(elicitationResult);
    
    // Remove less relevant resources
    this.currentContext = this.currentContext.filter(r => {
      const stillRelevant = this.checkRelevance(r, topics, concerns);
      return stillRelevant || r.annotations?.priority > 0.8;
    });
    
    // Add newly relevant resources
    const newlyRelevant = await this.findRelevantResources(
      memberId,
      topics,
      concerns
    );
    
    for (const resource of newlyRelevant) {
      if (!this.currentContext.find(r => r.uri === resource.uri)) {
        this.currentContext.push(resource);
        
        // Notify about context change
        await this.notifyContextChange({
          added: resource,
          reason: 'elicitation_relevance',
          topics
        });
      }
    }
    
    // Re-sort by updated relevance
    this.currentContext.sort((a, b) => {
      const aScore = this.scoreRelevance(a, topics, concerns);
      const bScore = this.scoreRelevance(b, topics, concerns);
      return bScore - aScore;
    });
  }
}
```

---

## 4. Practical Use Cases

### Use Case 1: Emergency Triage

```typescript
class EmergencyTriageElicitor {
  async triage(memberId: string): Promise<TriageResult> {
    // Sample only critical resources
    const criticalResources = await this.sampler.sample({
      memberId,
      filter: {
        priority: { min: 0.9 },
        types: ['allergies', 'medications', 'conditions', 'recent-vitals']
      },
      maxCount: 10
    });
    
    // Rapid elicitation
    const symptoms = await this.rapidSymptomScreen();
    
    // Adaptive sampling based on symptoms
    if (symptoms.includes('chest pain')) {
      // Add cardiac-related resources
      const cardiacResources = await this.sampler.sample({
        memberId,
        filter: { 
          tags: ['cardiac', 'heart'],
          recency: '30d'
        }
      });
      criticalResources.push(...cardiacResources);
    }
    
    return this.calculateTriage(symptoms, criticalResources);
  }
}
```

### Use Case 2: Medication Adherence Check

```typescript
class MedicationAdherenceElicitor {
  async checkAdherence(memberId: string): Promise<AdherenceReport> {
    // Sample medication resources
    const meds = await this.sampler.sample({
      memberId,
      filter: {
        types: ['medications'],
        status: 'active'
      }
    });
    
    const adherenceData: Map<string, AdherenceInfo> = new Map();
    
    for (const med of meds) {
      // Elicit adherence for each medication
      const response = await this.chat({
        message: `Are you taking your ${med.name} as prescribed (${med.instructions})?`,
        intents: [{
          name: 'medication.adherence.check',
          medicationId: med.id,
          expectedResponses: ['yes', 'no', 'sometimes', 'forgot']
        }]
      });
      
      if (response.value !== 'yes') {
        // Drill down on barriers
        const barriers = await this.elicitBarriers(med);
        adherenceData.set(med.id, {
          adherent: false,
          barriers,
          interventionNeeded: true
        });
      }
    }
    
    return this.generateAdherenceReport(adherenceData);
  }
}
```

### Use Case 3: Care Gap Identification

```typescript
class CareGapElicitor {
  async identifyGaps(memberId: string): Promise<CareGap[]> {
    // Sample care plan and history
    const carePlan = await this.getResource(`mcp://twins/member/${memberId}/care/plan`);
    const recentVisits = await this.sampler.sample({
      memberId,
      filter: {
        types: ['encounters'],
        recency: '6m'
      }
    });
    
    const gaps: CareGap[] = [];
    
    // Check preventive care
    for (const screening of carePlan.recommendedScreenings) {
      const completed = await this.chat({
        message: `Have you completed your ${screening.name} in the last ${screening.interval}?`,
        intents: [{
          name: 'screening.completion.check',
          screeningType: screening.type
        }]
      });
      
      if (!completed.value) {
        gaps.push({
          type: 'preventive',
          screening: screening.name,
          overdue: true,
          priority: screening.priority
        });
        
        // Elicit barriers
        const barriers = await this.chat({
          message: `What's preventing you from getting your ${screening.name}?`,
          intents: [{
            name: 'barrier.identification',
            expectedCategories: ['time', 'cost', 'access', 'fear', 'awareness']
          }]
        });
        
        gaps[gaps.length - 1].barriers = barriers.categories;
      }
    }
    
    return gaps;
  }
}
```

---

## 5. Implementation Patterns

### Sampling Configuration

```yaml
# Digital Twin sampling configuration
sampling:
  default_strategy: priority_weighted
  max_context_tokens: 8000
  
  strategies:
    emergency:
      priority_threshold: 0.9
      types: [allergies, medications, conditions]
      max_age: 7d
      
    routine_visit:
      priority_threshold: 0.5
      types: [all]
      max_age: 90d
      recency_weight: 0.3
      
    mental_health:
      priority_threshold: 0.6
      types: [assessments, medications, care_team]
      tags: [behavioral_health, mental_health]
      
    chronic_care:
      priority_threshold: 0.7
      types: [conditions, medications, labs, vitals]
      max_age: 30d
      recency_weight: 0.5
```

### Elicitation Scripts

```typescript
// Reusable elicitation scripts
const ELICITATION_SCRIPTS = {
  pain_assessment: [
    { question: "Where is your pain located?", type: "location" },
    { question: "How would you rate it from 1-10?", type: "scale" },
    { question: "When did it start?", type: "temporal" },
    { question: "What makes it better or worse?", type: "modifying_factors" }
  ],
  
  medication_review: [
    { question: "Are you taking all your medications?", type: "boolean" },
    { question: "Any side effects?", type: "open", followup: true },
    { question: "Any concerns about your medications?", type: "open" }
  ],
  
  depression_screen: [
    // PHQ-2 initial screen
    { question: "Over the past 2 weeks, have you felt little interest in doing things?", type: "frequency" },
    { question: "Over the past 2 weeks, have you felt down or hopeless?", type: "frequency" }
  ]
};
```

---

## 6. Metrics & Optimization

### Sampling Effectiveness Metrics

```typescript
interface SamplingMetrics {
  coverage: number;          // % of relevant resources included
  precision: number;         // % of included resources that were used
  tokenEfficiency: number;   // Useful tokens / total tokens
  latency: number;          // Time to sample
}

class SamplingOptimizer {
  async optimize(
    historicalSessions: Session[]
  ): Promise<OptimizedStrategy> {
    // Analyze which resources were actually used
    const usagePatterns = this.analyzeUsage(historicalSessions);
    
    // Adjust priorities based on actual usage
    const updatedPriorities = this.calculatePriorities(usagePatterns);
    
    // Test different sampling strategies
    const strategies = [
      'priority_only',
      'recency_weighted',
      'relevance_based',
      'hybrid'
    ];
    
    const results = await Promise.all(
      strategies.map(s => this.testStrategy(s, historicalSessions))
    );
    
    return this.selectBestStrategy(results);
  }
}
```

### Elicitation Quality Metrics

```typescript
interface ElicitationMetrics {
  completionRate: number;     // % of required info gathered
  questionEfficiency: number; // Info gained per question
  memberSatisfaction: number; // Self-reported or inferred
  clinicalValue: number;      // Utility for care decisions
}
```

---

## Summary

Sampling and elicitation in the Digital Twin MCP pattern enable:

1. **Intelligent Context Management**: Sample the most relevant resources based on priority, recency, and relevance
2. **Systematic Information Gathering**: Use structured elicitation to gather complete, accurate information
3. **Adaptive Interactions**: Dynamically adjust context based on what we learn
4. **Clinical Decision Support**: Combine sampling and elicitation for better care decisions
5. **Personalized Experiences**: Learn member preferences and communication styles over time

These patterns make AI agents more effective by ensuring they have the right information at the right time, gathered through natural, efficient conversations.
