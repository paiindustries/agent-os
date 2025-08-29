# CFML Code Style Guide

## Context

CFML-specific code style rules for Agent OS projects working with ColdFusion Markup Language, including both script-based and tag-based syntax.

<conditional-block context-check="cfml-formatting">
IF this CFML Formatting section already read in current context:
  SKIP: Re-reading this section
  NOTE: "Using CFML Formatting rules already in context"
ELSE:
  READ: The following CFML formatting rules

## CFML Formatting Rules

### Language Preference
- **Prefer script-based syntax** over tag-based for new code
- Use tag-based syntax only when required (views, specific functionality)
- Convert tag-based to script when refactoring existing code
- Maintain consistency within files (don't mix script and tags unnecessarily)

### Component Structure (.cfc files)
```cfml
component extends="BaseComponent" displayname="User" hint="User model" {
  
  // Use config() for framework initialization, NOT init()
  function config() {
    // Wheels-specific configuration
    table("users");
    hasMany("posts");
    validatesPresenceOf("name,email");
  }
  
  // Public methods first
  public string function getFullName() {
    return this.firstName & " " & this.lastName;
  }
  
  // Private methods last, prefixed with private access modifier
  private void function sanitizeData() {
    // Implementation
  }
}
```

### Indentation and Spacing
- Use **2 spaces** for indentation (never tabs)
- Add blank lines between functions
- Add space around operators: `x = y + z` not `x=y+z`
- No trailing whitespace
- End files with single newline

### Naming Conventions
- **Components**: PascalCase (e.g., `UserController.cfc`, `PaymentService.cfc`)
- **Functions**: camelCase (e.g., `getUserById()`, `calculateTotal()`)
- **Variables**: camelCase (e.g., `userList`, `currentDate`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `MAX_RETRY_COUNT`)
- **Database tables**: snake_case (e.g., `user_profiles`, `order_items`)
- **Framework internals**: $ prefix (e.g., `$init()`, `$save()`) - Never call directly

### String Formatting
- Use **double quotes** for strings: `"Hello World"`
- Use single quotes only to avoid escaping: `'He said "Hello"'`
- Use `##` for variable output in strings: `"User ID: #userId#"`
- Use pound signs sparingly in script: `result = "Total: " & total`

### Function Definitions
```cfml
// Script syntax (preferred)
public User function findActiveUsers(numeric limit = 10, string orderBy = "name") {
  return this.findAll(
    where = "active = ?",
    whereParams = [true],
    order = arguments.orderBy,
    maxRows = arguments.limit
  );
}

// Tag syntax (when necessary)
<cffunction name="findActiveUsers" access="public" returntype="User">
  <cfargument name="limit" type="numeric" default="10">
  <cfargument name="orderBy" type="string" default="name">
  
  <cfreturn this.findAll(
    where = "active = ?",
    whereParams = [true],
    order = arguments.orderBy,
    maxRows = arguments.limit
  )>
</cffunction>
```

### Query Formatting
```cfml
// Script syntax (preferred)
result = queryExecute(
  sql = "
    SELECT u.id, u.name, p.title
    FROM users u
    INNER JOIN posts p ON u.id = p.user_id
    WHERE u.active = ?
      AND p.published = ?
    ORDER BY u.name, p.created_at DESC
  ",
  params = [
    {value: true, cfsqltype: "cf_sql_boolean"},
    {value: true, cfsqltype: "cf_sql_boolean"}
  ],
  options = {datasource: "myapp"}
);

// Tag syntax (legacy)
<cfquery name="result" datasource="myapp">
  SELECT u.id, u.name, p.title
  FROM users u
  INNER JOIN posts p ON u.id = p.user_id
  WHERE u.active = <cfqueryparam value="#true#" cfsqltype="cf_sql_boolean">
    AND p.published = <cfqueryparam value="#true#" cfsqltype="cf_sql_boolean">
  ORDER BY u.name, p.created_at DESC
</cfquery>
```

### Array and Struct Formatting
```cfml
// Arrays
users = [
  {id: 1, name: "John Doe", email: "john@example.com"},
  {id: 2, name: "Jane Smith", email: "jane@example.com"}
];

// Structs
config = {
  database: {
    host: "localhost",
    port: 5432,
    name: "myapp_dev"
  },
  cache: {
    enabled: true,
    timeout: 3600
  }
};
```

### Comments and Documentation
```cfml
/**
 * Calculates the total price including tax and discounts
 * 
 * @param basePrice The original price before calculations
 * @param taxRate The tax rate as a decimal (e.g., 0.08 for 8%)
 * @param discount Optional discount amount to subtract
 * @returns The final calculated price
 */
public numeric function calculateTotalPrice(
  required numeric basePrice,
  required numeric taxRate,
  numeric discount = 0
) {
  // Calculate tax amount
  taxAmount = arguments.basePrice * arguments.taxRate;
  
  // Apply discount and return total
  return arguments.basePrice + taxAmount - arguments.discount;
}
```

### Error Handling
```cfml
// Script syntax (preferred)
try {
  user = model("User").findByKey(params.id);
  user.update(params.user);
  
  return {success: true, user: user};
} catch (any e) {
  logError("User update failed", e);
  return {success: false, error: e.message};
}

// Tag syntax
<cftry>
  <cfset user = model("User").findByKey(params.id)>
  <cfset user.update(params.user)>
  
  <cfreturn {success: true, user: user}>
  
  <cfcatch type="any">
    <cfset logError("User update failed", cfcatch)>
    <cfreturn {success: false, error: cfcatch.message}>
  </cfcatch>
</cftry>
```

</conditional-block>

## View File Formatting (.cfm files)

### HTML Structure
```cfm
<!--- views/users/index.cfm --->
<cfoutput>
<div class="users-index">
  <h1>Users</h1>
  
  <div class="actions">
    #linkTo(text="New User", route="newUser", class="btn btn-primary")#
  </div>
  
  <div class="users-grid">
    <cfloop array="#users#" item="user">
      <div class="user-card">
        <h3>#linkTo(text=user.name, route="user", key=user.key())#</h3>
        <p>#user.email#</p>
        <p class="meta">Created: #dateFormat(user.createdAt, "mmm d, yyyy")#</p>
      </div>
    </cfloop>
  </div>
</div>
</cfoutput>
```

### Form Helpers
```cfm
<cfoutput>
#startFormTag(route="users", method="post", class="user-form")#
  
  #errorMessagesFor("user")#
  
  <div class="form-group">
    #textField(
      objectName = "user",
      property = "name",
      label = "Full Name",
      class = "form-control",
      required = true
    )#
  </div>
  
  <div class="form-group">
    #textField(
      objectName = "user",
      property = "email",
      label = "Email Address",
      class = "form-control",
      required = true
    )#
  </div>
  
  <div class="form-actions">
    #submitTag(value="Save User", class="btn btn-primary")#
    #linkTo(text="Cancel", route="users", class="btn btn-secondary")#
  </div>
  
#endFormTag()#
</cfoutput>
```

## Engine Compatibility

### Adobe ColdFusion Specific
- Use `invoke()` for dynamic method calls: `invoke(component, methodName)`
- Avoid abbreviated function calls: use `abort()` not `abort`
- Be explicit with variable scoping in includes
- Use `<cfheader>` tag instead of `header()` function when needed

### Lucee Specific  
- Take advantage of modern syntax features
- Use `param` statement for default values
- Leverage enhanced closure support
- Utilize built-in performance optimizations

### Cross-Engine Patterns
```cfml
// Safe dynamic method invocation
if (isCallable(component, methodName)) {
  result = invoke(component, methodName, argumentStruct);
} else {
  throw(message="Method not found: #methodName#");
}

// Safe function existence checks
if (isCallable("functionName")) {
  result = functionName();
} else {
  // Fallback implementation
}
```

## Wheels Framework Specific

### Controller Patterns
- Always use `config()` method for initialization
- Use `provides()` for content negotiation
- Check `$performedRenderOrRedirect()` before rendering
- Use `renderWith()` for automatic format detection

### Model Patterns
- Define associations, validations, and callbacks in `config()`
- Never call `$` prefixed methods directly
- Always check `IsObject()` before using model instances
- Use transactions in complex operations

### Common Anti-Patterns to Avoid
```cfml
// WRONG - calling framework internal methods
user.$save();

// CORRECT - using public API
user.save();

// WRONG - using init() instead of config()
function init() {
  // This won't work in Wheels
}

// CORRECT - using config() for framework setup
function config() {
  // Wheels configuration goes here
}
```

## Code Quality Standards

### Validation Rules
- All components must have `config()` method when extending Wheels classes
- No direct calls to `$` prefixed methods
- All database queries must use parameterized queries
- Error handling required for all external operations
- Type hints required on function parameters and return values

### Performance Considerations
- Use appropriate `select` clauses to limit data retrieval
- Implement query caching for expensive operations
- Use `include` parameter judiciously to avoid N+1 queries
- Cache calculated properties when appropriate
- Use database indexes for frequent query conditions