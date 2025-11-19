# Member ID Flow Trace Research

**Research Date**: 2025-01-27
**Researcher**: AI Agent
**Status**: ✅ Validated
**Source**: Extracted from `ACT-2861_MEMBER_ID_TRACE.md`

## Summary

Backwards trace of member ID flow from Stedi Gateway through 8 layers to data sources. Identifies root cause of invalid placeholder values (`<no value>`) being sent to Stedi, causing thousands of HTTP 400 errors per day.

## Methodology

- **Tools Used**: Code analysis, backwards trace from error point
- **Repositories Analyzed**:
  - `realtime-eligibility` (Stedi Gateway → RTE Service → HTTP Gateway)
  - `member-sponsorship` (RTE Service → Request Builder → Field Processor)
- **Issue**: ACT-2861

## Findings

### Finding 1: Complete Request Chain (8 Layers)

**Flow Path** (backwards from Stedi Gateway):

1. **Stedi Gateway** (`realtime-eligibility/app/gateway/stedi/eligibility.go:12-20`)
   - Receives `Eligibility270Request` with `InsuranceMemberId` field
   - PR #547 adds validation here (defense-in-depth)

2. **RTE Eligibility Service** (`realtime-eligibility/app/service/eligibility/delegating/delegating.go:30-36`)
   - Passes request through without modification

3. **RTE HTTP Gateway** (`realtime-eligibility/app/gateway/rte/http.go:47-54`)
   - Marshals and forwards request

4. **HTTP Handler** (`realtime-eligibility/app/handler/http/check_eligibility.go:65-70`)
   - Parses request from HTTP body

5. **Member-Sponsorship RTE Service** (`member-sponsorship/app/service/channel/rte/rte.go:81-92`)
   - Constructs 270 request from template using verification fields

6. **Request Builder** (`member-sponsorship/app/service/channel/rte/request/request_builder.go:103-119`)
   - Converts verification fields to template map

7. **Field Processor** (`member-sponsorship/app/service/channel/rte/request/field_processor.go:130-146`) ⚠️ **CRITICAL**
   - Only strips spaces, **does NOT validate** placeholder values
   - Invalid values like `<no value>`, `<nil>`, `null` pass through

8. **Verification Fields Sources**:
   - **Enrollment Service** (`member-sponsorship/app/service/enrollment/service.go:162-163`)
   - **Member Information Service** (`member-sponsorship/app/service/memberinformation/athena_subscriber.go:174-177`)
   - **Population Enrichment** (`member-sponsorship/app/service/population/enrichment.go:188-191`)

### Finding 2: Root Cause - Missing Validation

**Critical Issue**: `VerificationFieldsToRTESafeMap` function only strips spaces but does NOT validate for placeholder values.

**Code Reference**:
```go
// File: member-sponsorship/app/service/channel/rte/request/field_processor.go:130-146
func VerificationFieldsToRTESafeMap(
    ctx context.Context,
    verificationFields []model.VerificationField,
) map[string]interface{} {
    fieldMap := make(map[string]interface{})
    for _, field := range verificationFields {
        switch field.CanonicalName {
        case model.VerificationFieldCanonicalMemberInsuranceID,
            model.VerificationFieldCanonicalSubInsuranceMemberID:
            // strip out all spaces
            fieldMap[field.Name] = strings.ReplaceAll(field.Value, " ", "")
        default:
            fieldMap[field.Name] = strings.Trim(field.Value, " ")
        }
    }
    return fieldMap
}
```

**Problem**: If `field.Value` contains `<no value>`, `<nil>`, or `null`, it gets inserted directly into the template map and rendered into the final 270 request.

### Finding 3: Where Placeholder Values Originate

1. **Template Systems**: Go templates insert placeholder values when values are missing
2. **Database/API Sources**: Member information records or API responses contain placeholder values
3. **Data Transformation**: Missing values converted to placeholder strings during transformation

### Finding 4: Why It Reaches Stedi

1. **No Validation in Field Processor**: Only strips spaces
2. **No Validation in Request Builder**: Doesn't validate before rendering template
3. **Template Rendering**: Directly inserts field values without validation
4. **RTE Service Validation**: Only validates after receiving request (PR #547), but invalid value already sent

## Validation

### Code Location Verification
- ✅ Verified: `realtime-eligibility/app/gateway/stedi/eligibility.go:12-20` (CheckEligibility function exists)
- ✅ Verified: `member-sponsorship/app/service/channel/rte/request/field_processor.go:130-146` (VerificationFieldsToRTESafeMap function exists)
- ✅ Verified: `member-sponsorship/app/service/channel/rte/request/request_builder.go:103-119` (BuildRequest function exists)
- ✅ Verified: `member-sponsorship/app/service/enrollment/service.go:162-163` (enrollment service reference)
- ✅ Verified: `member-sponsorship/app/service/memberinformation/athena_subscriber.go:174-177` (athena subscriber reference)

### Surveyor Validation
```bash
# Verify member-sponsorship → realtime-eligibility call chain
surveyor trace-rpc --target realtime-eligibility --workspaces $IH_HOME

# Expected: Should find member-sponsorship as caller
# Status: ⚠️ Needs verification
```

## Citations

- **Related JIRA Issue**: ACT-2861
- **Related PR**: PR #547 (adds validation in Stedi Gateway)
- **Related Research**:
  - `../timeouts/rte-timeout-cascade.md` (timeout analysis)
  - `rte-dependency-chains.md` (dependency analysis)

## Notes

- This research identifies the **primary fix location** (field processor) vs. defense-in-depth (RTE service validation)
- PR #547 provides defense-in-depth but doesn't prevent invalid values from being sent
- Root cause fix should be in field processor to catch invalid values before template rendering
