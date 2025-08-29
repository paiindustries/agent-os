# Generate Wheels Components

Generate Wheels CFML framework components including models, controllers, migrations, and scaffolds using Agent OS workflows.

## Usage

This command provides intelligent code generation for the Wheels framework, leveraging specialized agents and following established patterns.

### Basic Generation Commands

```bash
# Generate a model
/wheels-generate model User name:string email:string active:boolean

# Generate a controller
/wheels-generate controller Users index,show,new,create,edit,update,delete

# Generate a migration
/wheels-generate migration CreateUsersTable

# Generate a full scaffold
/wheels-generate scaffold Product name:string price:decimal description:text
```

## Implementation Workflow

When this command is executed, follow these steps:

### 1. Parse Generation Request
```
ANALYZE the generation request:
- Component type (model, controller, migration, scaffold)
- Component name and parameters
- Associated attributes and relationships
- Target location in project structure
```

### 2. Context Analysis
```
USE context-fetcher agent to:
- Load existing project structure
- Identify related components
- Check for naming conflicts
- Verify database schema alignment
```

### 3. Specialized Agent Delegation

Based on the component type, delegate to the appropriate specialized agent:

#### Model Generation
```
USE wheels-model agent to:
- Generate model component with proper associations
- Add validations based on attribute types
- Create relationships to existing models
- Follow Wheels ActiveRecord patterns
```

#### Controller Generation  
```
USE general-purpose agent with Wheels context to:
- Generate controller with CRUD actions
- Implement proper content negotiation
- Add authentication/authorization filters
- Follow Wheels controller patterns
```

#### Migration Generation
```
USE wheels-migration agent to:
- Create database migration with proper structure
- Handle multi-database compatibility
- Add appropriate indexes and constraints
- Implement safe rollback strategy
```

#### Scaffold Generation
```
COORDINATE multiple agents:
1. USE wheels-model agent for model creation
2. USE wheels-migration agent for database structure
3. USE general-purpose agent for controller creation
4. USE general-purpose agent for view generation
5. USE git-workflow agent for organized commits
```

### 4. Component Generation

For each component type:

#### Model Components
```cfml
// Generated model structure
component extends="Model" {
  
  function config() {
    // Table mapping
    table("tableName");
    
    // Generated associations based on naming conventions
    hasMany("relatedModels");
    belongsTo("parentModel");
    
    // Generated validations based on attribute types
    validatesPresenceOf("requiredField1,requiredField2");
    validatesUniquenessOf("uniqueField");
    validatesFormatOf(property="email", regex="^[^@]+@[^@]+\.[^@]+$");
    
    // Generated callbacks when appropriate
    beforeCreate("setDefaults");
  }
  
  // Generated helper methods based on context
  public boolean function isActive() {
    return this.status == "active";
  }
  
  // Generated custom finders
  public query function findActive() {
    return this.findAll(where="status='active'", order="name ASC");
  }
}
```

#### Controller Components
```cfml
component extends="Controller" {
  
  function config() {
    // Generated content negotiation
    provides("html,json");
    
    // Generated filters based on patterns
    filters("authenticate", except="index,show");
    
    // Generated common setup
    filters("loadResource", only="show,edit,update,delete");
  }
  
  // Generated CRUD actions following Wheels patterns
  function index() {
    resources = model("Resource").findAll(order="createdAt DESC");
    renderWith(resources);
  }
  
  function show() {
    renderWith(resource);
  }
  
  function new() {
    resource = model("Resource").new();
  }
  
  function create() {
    resource = model("Resource").create(params.resource);
    if (resource.hasErrors()) {
      renderView("new");
    } else {
      redirectTo(route="resource", key=resource.key());
    }
  }
  
  // Additional CRUD actions...
  
  private function loadResource() {
    resource = model("Resource").findByKey(params.key);
    if (!IsObject(resource)) {
      redirectTo(route="resources");
    }
  }
}
```

#### Migration Components
```cfml
component extends="Migration" {
  
  function up() {
    transaction {
      createTable(name="tableName", id=true) {
        // Generated columns based on attributes
        t.string(columnNames="name", limit=255, null=false);
        t.text(columnNames="description");
        t.boolean(columnNames="active", default=true);
        t.decimal(columnNames="price", precision=10, scale=2);
        t.timestamps();
      };
      
      // Generated indexes for performance
      addIndex(table="tableName", columnNames="name");
      addIndex(table="tableName", columnNames="active");
      
      // Generated foreign key constraints
      addForeignKey(
        table="tableName",
        column="userId",
        referencedTable="users",
        referencedColumn="id"
      );
    }
  }
  
  function down() {
    transaction {
      dropTable("tableName");
    }
  }
}
```

### 5. Post-Generation Tasks

After generating components:

#### Route Integration
```
IF controller generated:
  UPDATE config/routes.cfm to add resource routes:
  
  mapper()
    .resources("componentName")
  .end();
```

#### Test Generation
```
USE wheels-test agent to:
- Generate corresponding test files
- Create test data factories
- Add basic test coverage for generated methods
```

#### Documentation Updates
```
UPDATE project documentation:
- Add component to API documentation
- Update database schema documentation
- Add usage examples
```

## Advanced Generation Patterns

### Model Relationships
```bash
# Generate model with associations
/wheels-generate model Post title:string body:text userId:integer categoryId:integer

# This generates:
# - belongsTo("user") based on userId
# - belongsTo("category") based on categoryId  
# - Appropriate foreign key constraints in migration
```

### API Controllers
```bash
# Generate API-only controller
/wheels-generate controller api/v1/Users index,show,create,update,delete --api-only

# This generates:
# - onlyProvides("json") content negotiation
# - API-specific error handling
# - JSON response patterns
```

### Complex Scaffolds
```bash
# Generate full application scaffold
/wheels-generate scaffold Product name:string:required price:decimal:required description:text category:belongs_to user:belongs_to

# This generates:
# - Product model with validations and associations
# - Migration with proper indexes and foreign keys
# - Products controller with full CRUD
# - Views with form helpers and associations
# - Routes configuration
# - Test files with coverage
```

## Integration with Agent OS Workflows

### Spec-Driven Generation
```
IF current task is part of Agent OS spec:
  1. READ spec requirements for component details
  2. GENERATE components that fulfill spec needs
  3. UPDATE tasks.md to mark generation complete
  4. COMMIT changes with descriptive messages
```

### Pattern-Aware Generation
```
ALWAYS apply Agent OS patterns:
- Load @.agent-os/standards/wheels-patterns.md
- Follow @.agent-os/standards/code-style/cfml-style.md
- Use established naming conventions
- Implement proper error handling
- Add appropriate documentation
```

## Quality Assurance

### Generated Code Validation
```
AFTER generation, verify:
1. Component syntax is valid CFML
2. Follows Wheels framework patterns
3. Implements proper associations
4. Includes appropriate validations
5. Has corresponding test files
6. Integrates with existing codebase
```

### Multi-Engine Compatibility
```
ENSURE generated code:
- Works on Lucee 6+, Adobe CF 2021+, BoxLang
- Uses cross-engine compatible syntax
- Avoids engine-specific features
- Includes proper function parentheses for Adobe CF
- Uses invoke() for dynamic method calls when needed
```

## Error Handling and Validation

### Pre-Generation Checks
```
BEFORE generating components:
1. Verify project is Wheels-based (check box.json)
2. Confirm database migrations are up to date
3. Check for naming conflicts with existing components
4. Validate attribute types and relationships
5. Ensure required dependencies are available
```

### Generation Failure Recovery
```
IF generation fails:
1. Report specific error and location
2. Suggest corrective actions
3. Offer to retry with different parameters
4. Clean up any partially created files
5. Restore project to clean state
```

## Usage Examples

### Basic Model Generation
```
User Request: "Generate a User model with name, email, and phone"

Agent Response:
1. Parse: model=User, attributes=name:string,email:string,phone:string
2. Delegate to wheels-model agent
3. Generate User.cfc with validations
4. Generate corresponding migration
5. Generate UserTest.cfc
6. Report completion with file paths
```

### Complex Scaffold Generation
```
User Request: "Create a complete blog system with posts and categories"

Agent Response:
1. Generate Category model and migration
2. Generate Post model with belongsTo("category")
3. Generate Categories controller and views
4. Generate Posts controller with category association
5. Update routes.cfm with resources
6. Generate comprehensive test suite
7. Commit changes with git-workflow agent
8. Report complete blog system ready for use
```

## Output Format

After successful generation, provide:

```
🎯 Wheels Component Generation Complete

Generated Components:
✅ app/models/User.cfc - User model with validations and associations
✅ db/migrate/2024-01-15-120000-CreateUsersTable.cfc - Database migration  
✅ app/controllers/Users.cfc - CRUD controller with content negotiation
✅ app/views/users/ - Complete view templates with form helpers
✅ tests/specs/models/UserTest.cfc - Model test coverage
✅ tests/specs/controllers/UsersTest.cfc - Controller test coverage

Updated Configuration:
📝 config/routes.cfm - Added users resource routes

Next Steps:
1. Run migration: wheels dbmigrate latest
2. Run tests: wheels test run
3. Start server: server start
4. Visit: http://localhost:8080/users

Generated code follows:
- Wheels framework best practices
- Agent OS code standards
- Multi-engine compatibility patterns
- Test-driven development approach

Ready for development!
```

## Command Integration

This command integrates with other Agent OS commands:

- **After generation**: Automatically run `/wheels-db migrate` if migrations created
- **Code quality**: Use `/format` to ensure consistent code style  
- **Testing**: Automatically run `/wheels-test` for generated components
- **Git workflow**: Use git-workflow agent to create meaningful commits
- **Deployment**: Update deployment scripts if needed

The command returns control to the main agent with a summary of generated components and recommended next steps.