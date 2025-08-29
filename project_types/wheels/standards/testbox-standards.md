# TestBox Testing Standards

## Context

Wheels CFML framework testing standards using TestBox BDD framework for Agent OS projects.

<conditional-block context-check="testbox-standards">
IF this TestBox Standards section already read in current context:
  SKIP: Re-reading this section
  NOTE: "Using TestBox standards already in context"
ELSE:
  READ: The following TestBox testing standards

## Test Generation Commands

### Model Tests
```bash
# Basic model test
wheels generate test model User

# Model test with associations
wheels generate test model Post --with="User,Category"
```

### Controller Tests
```bash
# Basic controller test
wheels generate test controller Users

# Controller test with CRUD actions
wheels generate test controller Users --crud

# API controller test
wheels generate test controller api.Users --format=json
```

### Test Execution Commands
```bash
# Run all tests
box wheels test

# Run specific test directory
box wheels test --directory=tests/specs/unit
box wheels test --directory=tests/specs/integration
box wheels test --directory=tests/specs/functional

# Run single test file
box wheels test --bundles=tests.specs.unit.models.UserTest

# Watch mode for TDD
box testbox watch

# Coverage reporting
box wheels test --coverage
box wheels test --coverage --coverageReporter=html
```

## Test Structure Standards

### Directory Organization
```
tests/
├── BaseSpec.cfc              - Wheels-aware base test class
├── runner.cfm                 - Web-based test runner
├── specs/
│   ├── unit/                  - Isolated unit tests
│   │   ├── models/            - Model tests
│   │   ├── controllers/       - Controller unit tests
│   │   └── helpers/           - Helper function tests
│   ├── integration/           - Integration tests
│   │   ├── api/               - API integration tests
│   │   └── workflows/         - Multi-component workflows
│   └── functional/            - End-to-end tests
├── fixtures/                  - Test data files
└── support/
    ├── factories/             - Test data factories
    └── TestHelpers.cfc        - Common test utilities
```

### Naming Conventions
- Test files: `ModelNameTest.cfc`, `ControllerNameTest.cfc`
- Test methods: descriptive with `it("should do something", () => {})`
- Factory methods: `userFactory()`, `postFactory()`, `categoryFactory()`
- Test data: Use prefixes like `test_`, `integration_test_`

## BDD Structure Standards

### Basic Test Structure
```cfml
component extends="testbox.system.BaseSpec" {
  
  function run() {
    
    describe("Component Name", () => {
      
      beforeEach(() => {
        // Setup before each test
      });
      
      afterEach(() => {
        // Cleanup after each test
      });
      
      it("should behave as expected", () => {
        // Test logic with expectations
        expect(actual).toBe(expected);
      });
      
    });
  }
}
```

### Model Test Structure
```cfml
component extends="testbox.system.BaseSpec" {
  
  function run() {
    
    describe("User Model", () => {
      
      beforeEach(() => {
        variables.user = model("User").new({
          name: "Test User",
          email: "test@example.com"
        });
      });
      
      describe("validations", () => {
        
        it("should validate required fields", () => {
          var emptyUser = model("User").new();
          expect(emptyUser.valid()).toBeFalse();
          expect(emptyUser.errorsOn("name")).toHaveLength(1);
        });
        
        it("should validate email format", () => {
          variables.user.email = "invalid";
          expect(variables.user.valid()).toBeFalse();
          expect(variables.user.errorsOn("email")).toHaveLength(1);
        });
        
      });
      
      describe("associations", () => {
        
        it("should have many posts", () => {
          expect(variables.user).toHaveKey("posts");
          expect(IsArray(variables.user.posts())).toBeTrue();
        });
        
      });
      
    });
  }
}
```

### Controller Test Structure
```cfml
component extends="WheelsTestBase" {
  
  function run() {
    
    describe("Users Controller", () => {
      
      beforeEach(() => {
        variables.testUser = createTestModel("User", {
          name: "Test User",
          email: "test@example.com"
        });
      });
      
      describe("GET index", () => {
        
        it("should display users list", () => {
          var result = processTestRequest(route="users", method="GET");
          
          expect(result).toHaveKey("users");
          expect(result.users.recordCount).toBeGTE(1);
        });
        
      });
      
      describe("POST create", () => {
        
        it("should create user with valid data", () => {
          var userData = {
            user: {
              name: "New User",
              email: "new@example.com"
            }
          };
          
          var result = processTestRequest(
            route="users", 
            method="POST", 
            params=userData
          );
          
          expect(result.user.hasErrors()).toBeFalse();
          expect(result.redirectedTo).toContain("users");
        });
        
        it("should show errors with invalid data", () => {
          var invalidData = {
            user: {
              name: "",
              email: "invalid"
            }
          };
          
          var result = processTestRequest(
            route="users", 
            method="POST", 
            params=invalidData
          );
          
          expect(result.user.hasErrors()).toBeTrue();
          expect(result.template).toBe("new");
        });
        
      });
      
    });
  }
}
```

## Test Data Standards

### Factory Pattern
Create consistent test data using factory methods:

```cfml
// tests/support/factories/UserFactory.cfc
component {
  
  function create(struct attributes = {}) {
    var defaults = {
      name: "Test User #randRange(1, 9999)#",
      email: "test#randRange(1, 9999)#@example.com",
      password: "password123",
      status: "active"
    };
    
    defaults.putAll(attributes);
    
    return model("User").create(defaults);
  }
  
  function build(struct attributes = {}) {
    var defaults = {
      name: "Test User #randRange(1, 9999)#",
      email: "test#randRange(1, 9999)#@example.com",
      password: "password123",
      status: "active"
    };
    
    defaults.putAll(attributes);
    
    return model("User").new(defaults);
  }
}
```

### Using Factories in Tests
```cfml
describe("User Model", () => {
  
  beforeEach(() => {
    // Create saved user
    variables.savedUser = create("user", {role: "admin"});
    
    // Build unsaved user
    variables.newUser = build("user");
    
    // Create multiple users
    variables.users = createList("user", 5);
  });
  
});
```

## Assertion Standards

### Model Assertions
```cfml
// Validation assertions
expect(model.valid()).toBeTrue();
expect(model.hasErrors()).toBeFalse();
expect(model.errorsOn("field")).toHaveLength(0);

// Data assertions
expect(model.save()).toBeTrue();
expect(model.key()).toBeNumeric();
expect(model.field).toBe("expected value");

// Association assertions
expect(model.association()).toBeInstanceOf("Model");
expect(model.associations()).toHaveLength(3);
```

### Controller Assertions
```cfml
// Response assertions
expect(result.status).toBe(200);
expect(result.template).toBe("index");
expect(result.redirectedTo).toContain("success");

// Data assertions
expect(result).toHaveKey("users");
expect(result.users.recordCount).toBeGTE(1);

// Error assertions
expect(result.model.hasErrors()).toBeTrue();
expect(flash.error).toContain("validation");
```

### Custom Assertion Helpers
```cfml
// tests/support/TestHelpers.cfc
component {
  
  function assertHasErrors(required any model, string property = "") {
    expect(arguments.model.hasErrors()).toBeTrue("Model should have validation errors");
    
    if (Len(arguments.property)) {
      expect(arguments.model.errorsOn(arguments.property)).toHaveLength_GT(0, 
             "Should have errors on property: #arguments.property#");
    }
  }
  
  function assertRedirectsTo(required any result, required string path) {
    expect(arguments.result).toHaveKey("redirectedTo", "Should have redirected");
    expect(arguments.result.redirectedTo).toContain(arguments.path, 
           "Should redirect to: #arguments.path#");
  }
  
  function assertRendersTemplate(required any result, required string template) {
    expect(arguments.result).toHaveKey("template", "Should have rendered template");
    expect(arguments.result.template).toBe(arguments.template, 
           "Should render template: #arguments.template#");
  }
}
```

## Test Database Standards

### Database Configuration
```cfml
// Application.cfc or test-specific config
if (application.wheels.environment == "testing") {
  this.datasource = "wheels_test";
  
  // Test-specific settings
  set(dataSourceName="wheels_test");
  set(cacheActions=false);
  set(cachePartials=false);
  set(cacheQueries=false);
  set(showDebugInformation=false);
  set(showErrorInformation=true);
}
```

### Transaction Management
```cfml
component extends="testbox.system.BaseSpec" {
  
  function setup() {
    super.setup();
    
    // Start transaction for test isolation
    queryExecute("BEGIN TRANSACTION");
    variables.testTransaction = true;
  }
  
  function teardown() {
    super.teardown();
    
    // Rollback transaction to clean up test data
    if (StructKeyExists(variables, "testTransaction")) {
      queryExecute("ROLLBACK");
      StructDelete(variables, "testTransaction");
    }
  }
}
```

### Test Data Cleanup
```cfml
beforeEach(() => {
  // Clean specific test data
  queryExecute("DELETE FROM users WHERE email LIKE 'test_%'");
  queryExecute("DELETE FROM posts WHERE title LIKE 'Test Post%'");
});
```

## Integration Test Standards

### API Testing
```cfml
describe("Users API", () => {
  
  beforeEach(() => {
    variables.apiHeaders = {
      "Authorization": "Bearer #getTestToken()#",
      "Content-Type": "application/json"
    };
  });
  
  it("should return users list", () => {
    var response = makeAPIRequest(
      endpoint="/api/users",
      method="GET",
      headers=variables.apiHeaders
    );
    
    expect(response.status).toBe(200);
    expect(response.data).toHaveKey("users");
    expect(response.data.users).toBeArray();
  });
  
});
```

### Workflow Testing
```cfml
describe("User Registration Workflow", () => {
  
  it("should complete registration end-to-end", () => {
    var userData = {
      name: "Integration User",
      email: "integration@example.com",
      password: "password123"
    };
    
    // Step 1: Submit registration
    var registrationResponse = processRequest(
      route="register",
      method="POST",
      params={user: userData}
    );
    
    // Step 2: Verify user created
    var user = model("User").findOne(where="email = ?", whereParams=[userData.email]);
    expect(IsObject(user)).toBeTrue();
    
    // Step 3: Verify email sent (mock)
    expect(getEmailQueue()).toHaveLength(1);
    
    // Step 4: Verify redirect
    expect(registrationResponse.redirectedTo).toContain("login");
  });
  
});
```

## Performance Test Standards

### Query Performance
```cfml
describe("Query Performance", () => {
  
  beforeEach(() => {
    // Create test data
    variables.users = createList("user", 100);
  });
  
  it("should load users efficiently", () => {
    var startTime = getTickCount();
    
    var users = model("User").findAll(include="posts");
    
    var duration = getTickCount() - startTime;
    
    expect(users.recordCount).toBe(100);
    expect(duration).toBeLT(1000, "Query should complete within 1 second");
  });
  
});
```

## Error Handling Standards

### Exception Testing
```cfml
describe("Error Handling", () => {
  
  it("should handle invalid data gracefully", () => {
    expect(function() {
      model("User").create({email: "invalid"});
    }).notToThrow();
    
    var user = model("User").create({email: "invalid"});
    expect(user.hasErrors()).toBeTrue();
  });
  
  it("should throw on critical errors", () => {
    expect(function() {
      model("NonExistentModel").create();
    }).toThrow("Wheels.ModelNotFound");
  });
  
});
```

## Test Documentation Standards

### Test Descriptions
- Use clear, descriptive names that explain the behavior being tested
- Start with "should" to describe expected behavior
- Group related tests in describe blocks
- Use nested describe blocks for different aspects (validations, associations, etc.)

### Comments in Tests
```cfml
describe("User Authentication", () => {
  
  it("should authenticate with valid credentials", () => {
    // Arrange: Set up test data
    var user = create("user", {password: "secret123"});
    
    // Act: Perform authentication
    var result = authenticateUser(user.email, "secret123");
    
    // Assert: Verify expected outcome
    expect(result.authenticated).toBeTrue();
    expect(result.user.id).toBe(user.id);
  });
  
});
```

## TDD Workflow Standards

### Red-Green-Refactor Cycle
1. **Red**: Write failing test first
   ```bash
   wheels generate test model User
   # Add test that fails
   box wheels test --bundles=tests.specs.unit.models.UserTest
   ```

2. **Green**: Make test pass with minimal code
   ```bash
   # Implement just enough to pass
   box wheels test --bundles=tests.specs.unit.models.UserTest
   ```

3. **Refactor**: Improve code while keeping tests green
   ```bash
   # Refactor and run tests frequently
   box wheels test --bundles=tests.specs.unit.models.UserTest
   ```

### Continuous Testing
```bash
# Use watch mode during development
box testbox watch

# Run specific tests frequently
box wheels test --directory=tests/specs/unit/models
```

</conditional-block>