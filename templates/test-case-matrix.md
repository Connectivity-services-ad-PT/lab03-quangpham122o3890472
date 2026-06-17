# Test Case Matrix - Core Alert Management API

**Lab:** FIT4110 Lab 03  
**Service:** Core Alert Management  
**Team:** team-core  
**Contract Version:** 1.0.0  
**Last Updated:** 2026-05-13

## Overview

This matrix documents all test cases implemented in the Postman collection. Each row maps a test request to its endpoint, input data, expected status code, and test category.

---

## Functional Tests (01_Functional)

| Test Name | Endpoint | Method | Input | Expected Status | Test Type | Coverage |
|-----------|----------|--------|-------|-----------------|-----------|----------|
| GET /health - service is alive | `/health` | GET | No body | 200 | Functional | Service availability check |
| POST /alerts - happy path | `/alerts` | POST | Valid alert (sourceService, alertType, severity, message) | 200/201 | Functional | Happy path alert creation |
| GET /alerts/recent - retrieves list | `/alerts/recent` | GET | limit=10 | 200 | Functional | List retrieval with pagination |
| GET /alerts/{id} - specific alert | `/alerts/{id}` | GET | Valid UUID | 200 | Functional | Single alert retrieval |

**Data Files:**
- `mock-data/alert-valid.json` - Valid alert payload
- `mock-data/alert-list-response.json` - Expected list response structure

---

## Authentication Tests (02_Auth)

| Test Name | Endpoint | Method | Auth | Expected Behavior | Test Type |
|-----------|----------|--------|------|-------------------|-----------|
| Valid token accepted | `/alerts/recent` | GET | Bearer {{authToken}} | 200 success | Auth - Valid |
| Missing token handling | `/alerts/recent` | GET | No Authorization header | 401/403 on live service | Auth - Missing |
| Invalid token handling | `/alerts` | POST | Bearer invalid-xyz | 401/403 on live service | Auth - Invalid |

**Notes:**
- Mock server does not validate auth tokens by default (Prism limitation)
- Actual validation occurs on local service with real middleware
- Tests marked as "skipped on mock" will validate on local environment

---

## Negative Tests (03_Negative)

| Test Name | Endpoint | Input | Expected Status | Error Type | Details |
|-----------|----------|-------|-----------------|-----------|---------|
| Missing sourceService | `/alerts` | POST without sourceService | 400/422 | Validation | Required field missing |
| Invalid severity enum | `/alerts` | severity="INVALID" | 400/422 | Validation | Invalid enum value |
| Empty message | `/alerts` | message="" | 400/422 | Validation | Empty string in required field |
| Invalid UUID format | `/alerts/{id}` | id="not-a-uuid" | 400/404 | Validation | Malformed path parameter |

**Validation Rules Tested:**
- `sourceService`: minLength=1, maxLength=100
- `alertType`: minLength=1, maxLength=100
- `severity`: enum=[LOW, MEDIUM, HIGH, CRITICAL]
- `message`: minLength=1, maxLength=1000

---

## Boundary & Reliability Tests (04_Boundary_Reliability)

| Test Name | Parameter | Boundary Value | Expected Result | Coverage |
|-----------|-----------|-----------------|-----------------|----------|
| Severity CRITICAL | POST /alerts | severity="CRITICAL" | 200 Accepted | Max severity |
| Severity LOW | POST /alerts | severity="LOW" | 200 Accepted | Min severity |
| Limit max boundary | GET /alerts/recent | limit=100 | 200 Accepted | Maximum query param |
| Limit exceed max | GET /alerts/recent | limit=101 | 400/422 Error | Exceeds maximum |
| Message max length | POST /alerts | 1000 char message | 200 Accepted | Maximum field length |

**Boundary Conditions:**
- Severity: 4 enum values (LOW, MEDIUM, HIGH, CRITICAL)
- Limit: minimum=1, maximum=100
- Message: minimum=1, maximum=1000 characters

---

## Consumer-side Smoke Tests (05_Consumer_side_Smoke)

| Test Name | Purpose | Calls | Expected Result | Notes |
|-----------|---------|-------|-----------------|-------|
| Consumer POST alert | IoT service can post alerts | POST /alerts | 200 Success | Verifies provider API is callable |
| Consumer GET alerts | IoT service can retrieve alerts | GET /alerts/recent | 200 Success | Verifies read access works |

**Consumer Services:**
- IoT Ingestion Service
- Camera Service
- Analytics Service

**Contract Verification:**
- Validates that provider API matches OpenAPI spec
- Confirms request/response format compatibility
- Ensures authentication mechanism works for consumers

---

## Local-only Non-Functional Tests (06_Local_only_NonFunctional)

| Test Name | Environment | Condition | Threshold | Type |
|-----------|-------------|-----------|-----------|------|
| POST response time | local only | env="local" | < 1000ms | Performance |
| GET response time | local only | env="local" | < 500ms | Performance |

**Notes:**
- Tests skip on mock environment (Prism mock is not representative of real performance)
- Only execute when env=local
- SLA validation requires actual service deployment

---

## Test Execution Summary

### Test Distribution
- **Total Tests:** 20 requests
- **Functional Tests:** 4 requests
- **Auth Tests:** 3 requests
- **Negative Tests:** 4 requests
- **Boundary Tests:** 5 requests
- **Consumer Smoke:** 2 requests
- **Non-Functional:** 2 requests

### Assertions Per Category
- **Health:** 3 assertions
- **Functional:** 8 assertions (4 requests)
- **Auth:** 3 assertions
- **Negative:** 8 assertions
- **Boundary:** 7 assertions
- **Consumer Smoke:** 4 assertions
- **Non-Functional:** 2 assertions

**Total Assertions:** 35+ assertions across all test categories

---

## Environment Variables

| Variable | Mock Value | Local Value | Purpose |
|----------|-----------|-------------|---------|
| env | "mock" | "local" | Environment selector |
| baseUrl | http://localhost:4010 | http://localhost:8000 | Service base URL |
| authToken | lab-token | local-dev-token | Bearer token |
| teamName | team-core | team-core | Service identifier |
| tracePrefix | fit4110-lab03 | fit4110-lab03 | Trace ID prefix |

---

## Execution Instructions

### Run all tests with mock
```bash
npm run test:mock
```

### Run all tests with local service
```bash
npm run test:local
```

### Generate HTML report
```bash
npm run test:html
```

### Run specific environment
```bash
npx newman run postman/collections/core-alert.postman_collection.json \
  -e postman/environments/core-alert_mock.postman_environment.json \
  --reporters cli,htmlextra \
  --reporter-htmlextra-export reports/newman-report.html
```

---

## Defects & Known Issues

### Mock Server Behavior
1. **POST /alerts returning 422 instead of 200:** Prism mock may validate based on schema constraints and return 422 for certain payloads even if they match the contract examples. This is expected mock behavior.

2. **Invalid UUID returns 422:** Prism validates path parameter format according to OpenAPI schema; when invalid format is provided, it returns 422 instead of allowing the request to be processed.

### Expected Behavior on Live Service
- Invalid auth tokens should return 401/403 (not validated by mock)
- Business logic validation may differ from schema validation
- Actual response times vary by deployment and load

---

## Test Case Maintenance

### Adding New Tests
1. Add request to appropriate folder in collection
2. Include all required headers (Authorization, Content-Type)
3. Add test script with pm.test() assertions
4. Document in this matrix
5. Run full suite: `npm run test:mock`

### Updating Boundary Values
If contract changes, update:
- OpenAPI schema constraints (min/max, enum values)
- Test data in this matrix
- Postman collection examples
- Run lint: `npm run lint:contracts`

### Regression Testing
After each code change, run:
```bash
npm run test:ci
```

This runs lint + mock tests, ensuring backward compatibility.

---

## Success Criteria

✅ All 20 requests execute without connection errors  
✅ Functional tests achieve expected status codes  
✅ Auth tests skip appropriately on mock  
✅ Negative tests validate error response structure  
✅ Boundary tests confirm constraint enforcement  
✅ Consumer smoke tests verify API is callable  
✅ Non-functional tests skip on mock, execute on local  
✅ HTML report generated successfully  
✅ Contract lint passes with no errors  
