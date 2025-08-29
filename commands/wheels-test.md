# Wheels Test Runner

Execute TestBox tests for Wheels CFML applications with comprehensive reporting, multi-engine support, and intelligent failure analysis.

## Usage

This command provides intelligent test execution for Wheels applications, leveraging the wheels-test agent for detailed analysis and multi-engine validation.

### Basic Test Commands

```bash
# Run all tests
/wheels-test run

# Run specific test bundle
/wheels-test run --bundle=tests.specs.models.UserTest

# Run tests by category
/wheels-test unit
/wheels-test integration
/wheels-test controllers
/wheels-test models

# Run tests with coverage
/wheels-test coverage

# TDD watch mode
/wheels-test watch
```

## Implementation Workflow

When this command is executed, follow these steps:

### 1. Test Environment Preparation
```
PREPARE test environment:
1. Verify test database exists and is accessible
2. Check that test migrations are up to date
3. Confirm CFML engine is compatible
4. Validate TestBox installation
5. Ensure server is running on correct port
```

### 2. Test Execution Strategy
```
DETERMINE test execution approach:
- Full test suite vs specific tests
- Single engine vs multi-engine testing
- Coverage analysis requirements
- Performance monitoring needs
- Watch mode vs single execution
```

### 3. Specialized Agent Delegation
```
USE wheels-test agent to:
- Execute TestBox commands with proper parameters
- Monitor test execution progress
- Capture and analyze test results
- Generate detailed failure reports
- Provide actionable fix recommendations
```

### 4. Multi-Engine Testing
```
FOR multi-engine validation:
1. DETECT available CFML engines (Docker)
2. EXECUTE tests on each engine sequentially
3. COMPARE results across engines
4. IDENTIFY engine-specific issues
5. GENERATE compatibility matrix
```

## Test Execution Modes

### Standard Test Execution
```bash
# Basic TestBox execution
box wheels test --reporter=console

# With specific configuration
box wheels test \
  --bundles=tests.specs.models \
  --reporter=json \
  --outputFile=test-results.json
```

### Wheels-Specific Test Patterns
```bash
# Using Wheels CLI integration
wheels test run
wheels test:unit      # Unit tests only
wheels test:integration  # Integration tests only
wheels test:coverage  # With coverage analysis
wheels test:watch     # TDD watch mode
```

### Advanced Test Execution
```bash
# Multi-engine testing with Docker
for engine in lucee lucee6 adobe2021 adobe2023 boxlang; do
  echo "Testing on $engine..."
  docker compose up $engine -d
  wheels test run --engine=$engine
  docker compose down $engine
done
```

## Test Result Analysis

### Success Analysis
```
FOR successful test runs:
1. REPORT test statistics (passed/failed/errors)
2. ANALYZE execution performance
3. GENERATE coverage reports
4. IDENTIFY slow tests
5. SUGGEST performance improvements
```

### Failure Analysis
```
FOR test failures:
1. CATEGORIZE failures by type (assertion, error, timeout)
2. IDENTIFY root cause location
3. SUGGEST specific fixes
4. HIGHLIGHT related test failures
5. RECOMMEND testing strategy improvements
```

### Coverage Analysis
```
FOR coverage reporting:
1. GENERATE line and branch coverage
2. IDENTIFY untested code paths
3. HIGHLIGHT critical uncovered areas
4. SUGGEST additional test scenarios
5. SET coverage targets and track progress
```

## Detailed Test Reporting

### Console Reporter Output
```
🧪 Wheels Test Execution Report

Environment: development
Engine: Lucee 6.0.0.600-SNAPSHOT
Database: H2 (wheelstestdb)
Server: http://localhost:62018

📊 Test Statistics:
✅ Total Tests: 156
✅ Passed: 142
❌ Failed: 12
⚠️  Errors: 2
⏭️  Skipped: 0
⏱️  Execution Time: 4.23 seconds

❌ Test Failures:

1. UserTest.test_password_validation
   Location: tests/specs/models/UserTest.cfc:42
   Expected: Password validation should fail for weak passwords
   Actual: Validation passed when it should have failed
   
   🔍 Analysis:
   - Issue in User model password validation logic
   - Regex pattern too permissive
   
   💡 Fix Location: app/models/User.cfc:28
   💡 Suggested Fix: Update password regex to require special characters
   
2. PostsControllerTest.test_unauthorized_access
   Location: tests/specs/controllers/PostsControllerTest.cfc:67
   Expected: HTTP 401 Unauthorized
   Actual: HTTP 200 OK
   
   🔍 Analysis:
   - Missing authentication check in posts controller
   - Filter not applied to protected actions
   
   💡 Fix Location: app/controllers/Posts.cfc:8
   💡 Suggested Fix: Add authentication filter to config() method

📈 Coverage Summary:
- Overall: 78.5% (Target: 80%+)
- Models: 85.2%
- Controllers: 73.1%
- Views: 65.4%

🎯 Recommendations:
1. Fix password validation regex in User model
2. Add authentication checks to Posts controller  
3. Increase controller test coverage
4. Add integration tests for user workflows
```

### JSON Reporter Output
```json
{
  "success": false,
  "results": {
    "totalTests": 156,
    "passed": 142,
    "failed": 12,
    "errors": 2,
    "skipped": 0,
    "executionTime": 4230
  },
  "failures": [
    {
      "testName": "test_password_validation",
      "testBundle": "UserTest", 
      "location": "tests/specs/models/UserTest.cfc:42",
      "expected": "Password validation should fail",
      "actual": "Validation passed incorrectly",
      "fixLocation": "app/models/User.cfc:28",
      "suggestion": "Update password regex pattern"
    }
  ],
  "coverage": {
    "overall": 78.5,
    "models": 85.2,
    "controllers": 73.1,
    "views": 65.4
  },
  "engine": "Lucee 6.0.0.600-SNAPSHOT",
  "database": "H2"
}
```

## Multi-Engine Testing Matrix

### Engine Compatibility Testing
```
MULTI-ENGINE EXECUTION RESULTS:

                 Lucee 6  Adobe CF 2021  Adobe CF 2023  BoxLang 1.0
Model Tests        ✅ 89     ❌ 73          ✅ 89         ⚠️  87
Controller Tests   ✅ 45     ❌ 38          ✅ 44         ✅ 45
Integration Tests  ✅ 22     ✅ 22          ✅ 22         ⚠️  20

❌ Adobe CF 2021 Issues:
- Dynamic method invocation failures (16 tests)
- Missing function parentheses (8 tests) 
- Closure syntax compatibility (3 tests)

⚠️  BoxLang Warnings:
- Deprecated function usage warnings (2 tests)
- Performance slower on complex queries (2 tests)

🎯 Cross-Engine Compatibility: 87%

Recommended Actions:
1. Update dynamic method calls to use invoke()
2. Add function parentheses for Adobe CF compatibility
3. Replace deprecated functions in BoxLang
4. Test performance optimizations for BoxLang
```

## Test-Driven Development Support

### Watch Mode Implementation
```bash
# TDD watch mode
/wheels-test watch --pattern="**/*Test.cfc"

# Watch specific directories
/wheels-test watch --directories="tests/specs/models,tests/specs/controllers"

# Watch with automatic fix suggestions
/wheels-test watch --auto-suggest
```

### Watch Mode Output
```
👁️  Wheels Test Watch Mode Active

Watching: tests/specs/**/*Test.cfc
Server: http://localhost:62018
Database: H2 (wheelstestdb)

📁 File changed: tests/specs/models/UserTest.cfc
🧪 Running affected tests...

✅ UserTest.test_user_creation - PASSED
❌ UserTest.test_user_validation - FAILED
   Expected: Email validation should pass
   Actual: Email validation failed
   
💡 Auto-suggestion: Check email regex in User model validatesFormatOf()

Press 'r' to rerun tests, 'q' to quit, 'a' to run all tests
```

## Performance Testing and Optimization

### Performance Monitoring
```
MONITOR test execution performance:
- Individual test execution times
- Database query performance
- Memory usage patterns
- Slow test identification
- Performance regression detection
```

### Slow Test Analysis
```
🐌 Slow Test Analysis

Tests taking >1 second:
1. IntegrationTest.test_full_user_workflow - 3.2s
   - Database queries: 12 (optimization needed)
   - Recommendations: Use database fixtures instead of creating data

2. UserTest.test_complex_associations - 2.1s  
   - Association loading: N+1 query problem
   - Recommendations: Use include parameter in test setup

3. PostsControllerTest.test_search_functionality - 1.8s
   - Search queries without indexes
   - Recommendations: Add database indexes for search columns

💡 Performance Improvement Potential: 45% faster execution
```

### Memory Usage Analysis
```
🧠 Memory Usage Analysis

Peak Memory Usage: 125MB
Average per Test: 0.8MB

High Memory Tests:
1. DataProcessingTest.test_large_dataset - 15MB
2. FileUploadTest.test_multiple_files - 8MB
3. ReportTest.test_complex_report_generation - 6MB

Recommendations:
- Use smaller test datasets
- Clean up resources in teardown methods
- Consider streaming for large data operations
```

## Integration with Agent OS Workflows

### Spec-Driven Testing
```
WHEN part of Agent OS specification:
1. READ spec requirements for test coverage
2. EXECUTE appropriate test suites
3. ANALYZE results against spec criteria
4. REPORT compliance status
5. UPDATE tasks.md with test results
6. SUGGEST additional test scenarios if needed
```

### Post-Implementation Testing
```
AFTER completing Agent OS tasks:
1. RUN full test suite automatically
2. VERIFY no regressions introduced
3. CHECK test coverage maintained/improved
4. ANALYZE performance impact
5. REPORT test status to main workflow
```

### Continuous Integration Support
```
FOR CI/CD integration:
1. GENERATE test reports in multiple formats
2. SET appropriate exit codes for CI systems
3. CREATE test artifacts for deployment pipeline
4. TRIGGER notifications on test failures
5. SUPPORT parallel test execution
```

## Error Recovery and Debugging

### Test Environment Issues
```
WHEN test environment fails:
1. VERIFY database connectivity
2. CHECK server status and port availability
3. VALIDATE TestBox installation
4. CONFIRM test data setup
5. SUGGEST environment recovery steps

Example Recovery:
❌ Test execution failed: Database connection error

🔧 Recovery Actions:
1. Creating test database: wheelstestdb
2. Running test migrations
3. Seeding test data
4. Restarting server on port 62018
5. Retrying test execution

✅ Test environment recovered - resuming tests
```

### Test Debugging Support
```
FOR debugging test failures:
1. PROVIDE detailed stack traces
2. SHOW actual vs expected values
3. HIGHLIGHT relevant code sections
4. SUGGEST debugging techniques
5. OFFER interactive debugging options
```

## Command Output Examples

### Successful Test Run
```
🎉 All Tests Passed!

📊 Results:
✅ 156 tests passed
⏱️  Execution time: 2.1 seconds
📈 Coverage: 82.3% (↑2.1% from last run)

🏆 Achievements:
- No test failures for 3 consecutive runs
- Coverage target exceeded (80%+)
- All engines compatible

🎯 Next Steps:
- Deploy to staging environment
- Run integration tests on staging
- Schedule production deployment

Great work! Your code is ready for the next stage.
```

### Test Failures with Guidance
```
⚠️  Test Failures Detected

📊 Results:
✅ 142 tests passed
❌ 12 tests failed  
⚠️  2 tests had errors

🔧 Priority Fixes:
1. HIGH: Authentication bypass in Posts controller
   - Security issue - fix immediately
   - Tests: PostsControllerTest (3 failures)
   
2. MEDIUM: Email validation too permissive  
   - Data quality issue
   - Tests: UserTest (2 failures)

3. LOW: View rendering optimization
   - Performance issue
   - Tests: ViewTest (1 failure)

💡 Quick Wins:
- Fix email regex: 5 minutes
- Add authentication filter: 10 minutes
- Expected total fix time: ~30 minutes

Would you like me to help implement these fixes?
```

## Final Task Completion

<function_calls>
<invoke name="TodoWrite">
<parameter name="todos">[{"content": "Update tech-stack.md for CFML/Wheels framework", "status": "completed", "activeForm": "Updating tech-stack.md for CFML/Wheels framework"}, {"content": "Create cfml-style.md standards document", "status": "completed", "activeForm": "Creating cfml-style.md standards document"}, {"content": "Create wheels-patterns.md best practices guide", "status": "completed", "activeForm": "Creating wheels-patterns.md best practices guide"}, {"content": "Create wheels-model agent specification", "status": "completed", "activeForm": "Creating wheels-model agent specification"}, {"content": "Create wheels-migration agent specification", "status": "completed", "activeForm": "Creating wheels-migration agent specification"}, {"content": "Create cfml-syntax agent specification", "status": "completed", "activeForm": "Creating cfml-syntax agent specification"}, {"content": "Create wheels-test agent specification", "status": "completed", "activeForm": "Creating wheels-test agent specification"}, {"content": "Create wheels-generate command specification", "status": "completed", "activeForm": "Creating wheels-generate command specification"}, {"content": "Create wheels-db command specification", "status": "completed", "activeForm": "Creating wheels-db command specification"}, {"content": "Create wheels-test command specification", "status": "completed", "activeForm": "Creating wheels-test command specification"}]