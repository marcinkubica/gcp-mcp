# Testing Recommendations - GCP MCP Server

## **Current State Analysis**

**Project Overview:**
- Model Context Protocol server for Google Cloud Platform interactions
- 711 lines of TypeScript code in single file (`index.ts`)
- **No existing tests or testing infrastructure**
- Complex GCP API integrations with multiple Google Cloud services
- Dynamic code execution via VM contexts
- High-complexity authentication and project management logic

---

## **Testing Framework Setup**

### **Recommended Stack:**
- **Jest** - Primary testing framework
- **ts-jest** - TypeScript preprocessor
- **@google-cloud/storage** - For GCP service mocking
- **google-auth-library** - Authentication mocking
- **vm** module mocking for code execution testing

### **Installation:**
```bash
npm install --save-dev jest @types/jest ts-jest @types/node
npm install --save-dev @google-cloud/bigquery @google-cloud/storage # For mocking
```

### **Configuration Files:**

**jest.config.js:**
```javascript
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/', '<rootDir>/__tests__'],
  testMatch: ['**/__tests__/**/*.test.ts', '**/*.test.ts'],
  collectCoverageFrom: [
    'index.ts',
    '!**/*.d.ts'
  ],
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  setupFilesAfterEnv: ['<rootDir>/__tests__/setup.ts'],
  testTimeout: 30000 // GCP API calls may be slow
};
```

**package.json additions:**
```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:integration": "jest --testPathPattern=integration",
    "test:unit": "jest --testPathPattern=unit",
    "test:ci": "jest --ci --coverage --watchAll=false"
  }
}
```

---

## **Priority Testing Areas**

### **1. Core Utility Functions (High Priority)**

#### **`retry()` Function**
**Location:** Lines 266-278
**Risk Level:** HIGH - Critical for API reliability

**Test Cases:**
```typescript
describe('retry function', () => {
  it('should succeed on first attempt')
  it('should retry failed functions up to 3 times')
  it('should return result on eventual success')
  it('should throw error after max retries')
  it('should handle different error types')
  it('should maintain proper delay between retries')
});
```

#### **`wrapUserCode()` Function**
**Location:** Lines 658-675
**Risk Level:** CRITICAL - Executes arbitrary code

**Test Cases:**
```typescript
describe('wrapUserCode', () => {
  it('should wrap user code with proper context')
  it('should include all required GCP client imports')
  it('should handle async/await code')
  it('should preserve TypeScript syntax')
  it('should include error handling wrapper')
});
```

### **2. Authentication & Project Management (Critical Priority)**

#### **`initializeAuth()` Function**
**Location:** Lines 280-291
**Risk Level:** CRITICAL - Authentication foundation

**Test Cases:**
```typescript
describe('initializeAuth', () => {
  beforeEach(() => {
    jest.mock('google-auth-library');
  });

  it('should initialize GoogleAuth successfully')
  it('should handle authentication failures')
  it('should set up proper scopes')
  it('should handle credential file not found')
  it('should handle default credentials')
});
```

#### **`selectProject()` Function**
**Location:** Lines 293-304
**Risk Level:** HIGH - Project context management

**Test Cases:**
```typescript
describe('selectProject', () => {
  it('should select valid project successfully')
  it('should update global project state')
  it('should set default region when not provided')
  it('should handle invalid project IDs')
  it('should validate project access permissions')
});
```

### **3. GCP Service Integrations (High Priority)**

#### **BigQuery Integration**
**Test Cases:**
```typescript
describe('BigQuery integration', () => {
  beforeEach(() => {
    jest.mock('@google-cloud/bigquery');
  });

  it('should initialize BigQuery client correctly')
  it('should handle dataset operations')
  it('should handle query execution')
  it('should manage authentication properly')
  it('should handle BigQuery errors gracefully')
});
```

#### **Compute Engine Integration**
**Test Cases:**
```typescript
describe('Compute Engine integration', () => {
  beforeEach(() => {
    jest.mock('@google-cloud/compute');
  });

  it('should list instances correctly')
  it('should handle instance operations')
  it('should manage pagination')
  it('should handle API errors')
});
```

#### **Cloud Storage Integration**
**Test Cases:**
```typescript
describe('Cloud Storage integration', () => {
  beforeEach(() => {
    jest.mock('@google-cloud/storage');
  });

  it('should list buckets correctly')
  it('should handle file operations')
  it('should manage permissions')
  it('should handle storage errors')
});
```

### **4. Tool Handlers (High Priority)**

#### **Code Execution Tool**
**Location:** Lines 400-500 (estimated)
**Risk Level:** CRITICAL - Executes arbitrary TypeScript code

**Test Cases:**
```typescript
describe('run-gcp-code tool', () => {
  it('should execute valid TypeScript code')
  it('should handle code compilation errors')
  it('should handle runtime errors')
  it('should enforce timeout limits')
  it('should sandbox code execution properly')
  it('should return structured results')
  it('should handle async code correctly')
  it('should prevent malicious code execution')
});
```

#### **Project Management Tools**
**Test Cases:**
```typescript
describe('project management tools', () => {
  describe('list-projects', () => {
    it('should list accessible projects')
    it('should handle authentication errors')
    it('should format project data correctly')
  });

  describe('select-project', () => {
    it('should select valid projects')
    it('should reject invalid projects')
    it('should update server state')
  });
});
```

#### **Billing Tools**
**Test Cases:**
```typescript
describe('billing tools', () => {
  beforeEach(() => {
    jest.mock('@google-cloud/billing');
    jest.mock('@google-cloud/billing-budgets');
  });

  describe('get-billing-info', () => {
    it('should retrieve billing information')
    it('should handle projects without billing')
    it('should format billing data correctly')
  });

  describe('get-cost-forecast', () => {
    it('should generate cost forecasts')
    it('should handle different time periods')
    it('should handle insufficient data scenarios')
  });
});
```

---

## **Mocking Strategy**

### **Google Cloud Services:**
```typescript
// __tests__/mocks/gcp-services.ts
export const mockBigQuery = {
  query: jest.fn(),
  dataset: jest.fn(),
  createDataset: jest.fn()
};

export const mockStorage = {
  getBuckets: jest.fn(),
  bucket: jest.fn()
};

export const mockCompute = {
  getInstances: jest.fn(),
  getInstance: jest.fn()
};
```

### **Authentication Mock:**
```typescript
// __tests__/mocks/auth.ts
export const mockGoogleAuth = {
  getClient: jest.fn(),
  getAccessToken: jest.fn(),
  getProjectId: jest.fn()
};
```

### **VM Context Mock:**
```typescript
// __tests__/mocks/vm.ts
export const mockVM = {
  createContext: jest.fn(),
  runInContext: jest.fn()
};
```

---

## **Test File Structure**

```
__tests__/
├── setup.ts                          # Global test setup
├── mocks/
│   ├── gcp-services.ts               # GCP service mocks
│   ├── auth.ts                       # Authentication mocks
│   └── vm.ts                         # VM execution mocks
├── unit/
│   ├── retry.test.ts                 # Retry logic tests
│   ├── code-execution.test.ts        # Code execution tests
│   ├── auth.test.ts                  # Authentication tests
│   ├── project-management.test.ts    # Project selection tests
│   └── tool-handlers.test.ts         # Individual tool tests
├── integration/
│   ├── gcp-services.test.ts          # GCP API integration tests
│   ├── end-to-end.test.ts           # Full workflow tests
│   └── authentication.test.ts        # Auth flow tests
├── security/
│   ├── code-injection.test.ts        # Code injection prevention
│   ├── permission-escalation.test.ts # Security boundary tests
│   └── data-leakage.test.ts         # Sensitive data handling
└── fixtures/
    ├── sample-code.ts                # Sample TypeScript code
    ├── gcp-responses.ts              # Mock GCP API responses
    └── test-projects.ts              # Test project configurations
```

---

## **Security Testing (Critical)**

### **Code Injection Prevention:**
```typescript
describe('Security - Code Injection', () => {
  it('should prevent malicious imports')
  it('should block file system access attempts')
  it('should prevent network access outside GCP')
  it('should block process manipulation')
  it('should prevent environment variable access')
  it('should timeout infinite loops')
  it('should limit memory usage')
});
```

### **Authentication Security:**
```typescript
describe('Security - Authentication', () => {
  it('should not leak credentials in error messages')
  it('should validate project access permissions')
  it('should handle expired tokens gracefully')
  it('should prevent credential hijacking')
});
```

### **Data Protection:**
```typescript
describe('Security - Data Protection', () => {
  it('should not log sensitive data')
  it('should sanitize error outputs')
  it('should prevent unauthorized resource access')
  it('should validate resource permissions')
});
```

---

## **Performance Testing**

### **API Call Performance:**
```typescript
describe('Performance', () => {
  it('should handle concurrent API calls efficiently')
  it('should implement proper pagination')
  it('should respect API rate limits')
  it('should timeout slow operations')
  it('should handle large result sets')
});
```

### **Code Execution Performance:**
```typescript
describe('Code Execution Performance', () => {
  it('should limit execution time')
  it('should limit memory usage')
  it('should handle resource cleanup')
  it('should prevent resource exhaustion')
});
```

---

## **Integration Testing with GCP Emulators**

### **Setup GCP Emulators:**
```bash
# Install emulators
npm install -g @google-cloud/firestore-emulator
npm install -g @google-cloud/storage-emulator
```

### **Integration Test Configuration:**
```typescript
describe('GCP Integration Tests', () => {
  beforeAll(async () => {
    // Start emulators
    process.env.FIRESTORE_EMULATOR_HOST = 'localhost:8080';
    process.env.STORAGE_EMULATOR_HOST = 'localhost:9199';
  });

  it('should work with Cloud Storage emulator')
  it('should work with Firestore emulator')
  it('should handle emulator connectivity issues')
});
```

---

## **Error Scenario Testing**

### **Network and Connectivity:**
```typescript
describe('Error Scenarios', () => {
  it('should handle network timeouts')
  it('should handle API quota exceeded')
  it('should handle service unavailable')
  it('should handle authentication failures')
  it('should handle invalid project IDs')
  it('should handle permission denied errors')
});
```

---

## **Coverage Goals**

- **Target Coverage:** 85%+ overall (complex GCP integrations)
- **Critical Functions:** 95%+ coverage
  - Authentication: 95%
  - Code execution: 100%
  - Retry logic: 100%
  - Security boundaries: 100%
- **GCP Service Integrations:** 80%+ (heavy mocking required)

---

## **CI/CD Integration**

### **GitHub Actions with GCP Testing:**
```yaml
name: Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      gcp-emulator:
        image: gcr.io/google.com/cloudsdktool/cloud-sdk:emulators
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npm run test:unit
      - run: npm run test:integration
      - run: npm run test:coverage
```

---

## **Implementation Timeline**

### **Phase 1 (Week 1):** Foundation & Critical Functions
- Setup Jest and mocking infrastructure
- Test retry logic and utility functions
- Test code execution wrapper (security critical)

### **Phase 2 (Week 2):** Authentication & Project Management
- Mock Google Auth library
- Test authentication flows
- Test project selection and state management

### **Phase 3 (Week 3):** GCP Service Integration
- Mock all GCP client libraries
- Test individual service integrations
- Test tool handlers with mocked services

### **Phase 4 (Week 4):** Security & Integration
- Comprehensive security testing
- Integration tests with emulators
- Performance testing
- CI/CD pipeline setup

### **Phase 5 (Week 5):** Error Handling & Edge Cases
- Network failure scenarios
- API error handling
- Resource limit testing
- Documentation and cleanup

---

## **Risk Mitigation**

**Critical Security Risks:**
1. **Arbitrary code execution** - Requires comprehensive sandboxing tests
2. **GCP credential exposure** - Must prevent credential leakage
3. **Privilege escalation** - Validate project permissions thoroughly
4. **Resource abuse** - Implement and test resource limits

**Operational Risks:**
1. **API quota exhaustion** - Test rate limiting and retry logic
2. **Authentication token expiry** - Test token refresh mechanisms
3. **Network connectivity issues** - Test offline/degraded scenarios
4. **Large data handling** - Test memory and performance limits

**Testing Environment Requirements:**
- Isolated GCP test projects
- Emulator setup for offline testing
- Credential management for CI/CD
- Resource cleanup automation 
