# Wheels Framework Patterns

## Context

Wheels CFML framework specific patterns and best practices for Agent OS projects.

<conditional-block context-check="wheels-patterns">
IF this Wheels Patterns section already read in current context:
  SKIP: Re-reading this section
  NOTE: "Using Wheels patterns already in context"
ELSE:
  READ: The following Wheels framework patterns

## Core Framework Principles

### The $ Prefix Convention
- All internal framework methods prefixed with `$` (e.g., `$callAction()`, `$performedRenderOrRedirect()`)
- **CRITICAL**: Never call $ methods directly from application code
- Clear boundary between framework and application code
- Framework uses these internally, application code uses public API

### Initialization Pattern
- **ALWAYS use `config()` method for initialization, NOT `init()`**
- The `config()` method is where associations, validations, filters are defined
- Framework uses `$init()` internally, application code uses `config()`

### Content Negotiation
- Use `provides()` or `onlyProvides()` in controller `config()`
- Automatic format detection from URL or Accept headers
- Format-specific views (e.g., `show.json.cfm`)
- View rendering automatic for HTML, skipped for JSON/XML when using `renderText()`/`renderWith()`

</conditional-block>

## Model Patterns

### Basic Model Structure
```cfml
component extends="Model" {
  
  function config() {
    // Table configuration
    table("users");
    
    // Associations - define relationships
    hasMany("posts");
    hasMany("comments", through="posts");
    belongsTo("role");
    hasOne("profile");
    
    // Validations - data integrity rules
    validatesPresenceOf("name,email");
    validatesUniquenessOf("email");
    validatesFormatOf(property="email", regex="^[^@]+@[^@]+\.[^@]+$");
    validatesLengthOf(property="name", minimum=2, maximum=100);
    
    // Callbacks - lifecycle hooks
    beforeValidation("sanitizeInput");
    beforeCreate("setDefaults");
    afterCreate("sendWelcomeEmail");
    
    // Calculated properties
    property(name="fullName", sql="CONCAT(firstName, ' ', lastName)");
  }
  
  // Private callback methods
  private function sanitizeInput() {
    if (StructKeyExists(this, "email")) {
      this.email = Trim(LCase(this.email));
    }
  }
  
  // Public instance methods
  public boolean function isActive() {
    return this.status == "active";
  }
  
  // Custom finders
  public query function findActive() {
    return this.findAll(where="status='active'", order="name ASC");
  }
}
```

### Advanced Model Patterns

#### Nested Properties
```cfml
component extends="Model" {
  
  function config() {
    hasMany("orderItems");
    
    // Allow nested attributes for complex forms
    nestedProperties(
      association="orderItems",
      allowDelete=true,
      rejectIf="isBlank"
    );
  }
  
  private boolean function isBlank(struct properties) {
    return !Len(Trim(arguments.properties.productId ?: ""));
  }
}
```

#### Scoped Queries
```cfml
component extends="Model" {
  
  function config() {
    // Define reusable query scopes
    scope(name="published", where="publishedAt IS NOT NULL");
    scope(name="recent", order="createdAt DESC", maxRows=10);
    scope(name="byAuthor", where="authorId = ?");
  }
}

// Usage: model("Post").published().recent().findAll()
```

## Controller Patterns

### Basic CRUD Controller
```cfml
component extends="Controller" {
  
  function config() {
    // Content negotiation - support both HTML and JSON
    provides("html,json");
    
    // Filters - authentication/authorization
    filters("authenticate", except="index,show");
    filters("authorize", only="edit,update,delete");
    
    // Set common variables
    filters("loadCurrentUser");
  }
  
  function index() {
    posts = model("Post").findAll(
      order="createdAt DESC",
      include="author"  // Avoid N+1 queries
    );
    renderWith(posts);
  }
  
  function show() {
    post = model("Post").findByKey(params.key);
    if (!IsObject(post)) {
      flashInsert(error="Post not found");
      redirectTo(route="posts");
    }
    renderWith(post);
  }
  
  function create() {
    post = model("Post").create(params.post);
    if (post.hasErrors()) {
      flashInsert(error="Please fix the errors below");
      renderView("new");
    } else {
      flashInsert(success="Post created successfully");
      redirectTo(route="post", key=post.key());
    }
  }
}
```

### API Controller Pattern
```cfml
component extends="Controller" {
  
  function config() {
    // JSON-only API
    onlyProvides("json");
    
    // API-specific headers
    filters("setApiHeaders");
    filters("authenticateToken");
  }
  
  function setApiHeaders() {
    header name="X-API-Version" value="1.0";
    header name="Content-Type" value="application/json";
  }
  
  function index() {
    users = model("User").findAll(
      select="id,name,email,createdAt",
      order="name ASC"
    );
    
    renderWith(data={
      users: users,
      meta: {
        total: model("User").count(),
        page: params.page ?: 1
      }
    });
  }
  
  function create() {
    user = model("User").create(params.user);
    
    if (user.hasErrors()) {
      renderWith(
        data={errors: user.allErrors()}, 
        status=422
      );
    } else {
      renderWith(
        data={user: user}, 
        status=201
      );
    }
  }
}
```

## Routing Patterns

### RESTful Resources
```cfml
// config/routes.cfm
mapper()
  .resources("posts", only="index,show,create,update,delete")
  .resource("account")  // Singular resource
  .namespace("admin")
    .resources("users")
    .resources("products", except="destroy")
  .end()
  .scope(path="api/v1", name="apiV1")
    .resources("posts", controller="api.v1.posts")
  .end()
  .root(to="pages##home")
.end();
```

### Custom Routes with Constraints
```cfml
mapper()
  .get(
    name="userProfile",
    pattern="users/[username]",
    to="users##show",
    constraints={username="[a-zA-Z0-9_]+"}
  )
  .post(
    name="apiLogin", 
    pattern="api/auth/login",
    to="api.auth##create"
  )
.end();
```

## View Patterns

### Form Helpers with Error Handling
```cfm
<cfoutput>
#startFormTag(route="posts", method="post", class="post-form")#

  <!--- Display validation errors --->
  #errorMessagesFor("post")#
  
  <div class="form-group">
    #textField(
      objectName="post", 
      property="title", 
      label="Title",
      class="form-control",
      required=true
    )#
  </div>
  
  <div class="form-group">
    #textArea(
      objectName="post", 
      property="body", 
      label="Content",
      class="form-control",
      rows=10
    )#
  </div>
  
  <div class="form-group">
    #select(
      objectName="post",
      property="categoryId",
      options=categories,
      includeBlank="-- Select Category --",
      label="Category",
      class="form-select"
    )#
  </div>
  
  <div class="form-check">
    #checkBox(
      objectName="post", 
      property="published", 
      label="Publish immediately?",
      class="form-check-input"
    )#
  </div>
  
  <div class="form-actions">
    #submitTag(value="Save Post", class="btn btn-primary")#
    #linkTo(text="Cancel", route="posts", class="btn btn-secondary")#
  </div>

#endFormTag()#
</cfoutput>
```

### Partial Views
```cfm
<!--- views/posts/_post_card.cfm --->
<cfoutput>
<div class="post-card">
  <h3>#linkTo(text=post.title, route="post", key=post.key())#</h3>
  <p class="post-meta">
    By #post.author().name# on #dateFormat(post.createdAt, "mmm d, yyyy")#
    <cfif post.categoryId GT 0>
      in #post.category().name#
    </cfif>
  </p>
  <div class="post-excerpt">
    #post.excerpt#
  </div>
  <div class="post-actions">
    #linkTo(text="Read More", route="post", key=post.key(), class="btn btn-outline-primary")#
  </div>
</div>
</cfoutput>

<!--- Usage in main view --->
<cfoutput>
<div class="posts-grid">
  #includePartial(partial="post_card", query=posts)#
</div>
</cfoutput>
```

## Database Migration Patterns

### Create Table Migration
```cfml
component extends="Migration" {
  
  function up() {
    transaction {
      createTable(name="posts", id=true, force=true) {
        t.string(columnNames="title", limit=255, null=false);
        t.text(columnNames="body");
        t.string(columnNames="excerpt", limit=500);
        t.integer(columnNames="userId", null=false);
        t.integer(columnNames="categoryId");
        t.string(columnNames="status", limit=20, default="draft");
        t.boolean(columnNames="published", default=false);
        t.datetime(columnNames="publishedAt");
        t.timestamps();
      };
      
      // Add indexes for performance
      addIndex(table="posts", columnNames="userId");
      addIndex(table="posts", columnNames="status,published");
      addIndex(table="posts", columnNames="publishedAt");
      
      // Add foreign key constraints
      addForeignKey(
        table="posts",
        column="userId",
        referencedTable="users",
        referencedColumn="id"
      );
    }
  }
  
  function down() {
    transaction {
      dropTable("posts");
    }
  }
}
```

### Data Migration
```cfml
component extends="Migration" {
  
  function up() {
    transaction {
      // Add new column
      addColumn(
        table="users",
        columnName="slug",
        columnType="string",
        limit=100
      );
      
      // Populate existing records
      users = queryExecute("SELECT id, name FROM users");
      
      for (user in users) {
        slug = generateSlug(user.name);
        queryExecute(
          "UPDATE users SET slug = ? WHERE id = ?",
          [slug, user.id]
        );
      }
      
      // Add unique constraint after populating
      addIndex(table="users", columnNames="slug", unique=true);
    }
  }
  
  private string function generateSlug(string text) {
    return lcase(reReplace(arguments.text, "[^a-zA-Z0-9\s]", "", "all"))
           .replace(" ", "-");
  }
}
```

## Testing Patterns

### Model Tests
```cfml
component extends="wheels.Test" {
  
  function setup() {
    super.setup();
    
    // Create test data using factories
    testUser = model("User").create(
      name="Test User",
      email="test@example.com"
    );
  }
  
  function test_user_validation() {
    user = model("User").new();
    
    // Test required validations
    assert("!user.valid()");
    assert("ArrayLen(user.errorsOn('name')) > 0");
    
    // Test email format validation  
    user.name = "John Doe";
    user.email = "invalid-email";
    user.valid();
    assert("ArrayLen(user.errorsOn('email')) > 0");
    
    // Test valid data
    user.email = "john@example.com";
    assert("user.valid()");
  }
  
  function test_user_associations() {
    // Test association loading
    user = model("User").findOne(include="posts");
    assert("IsObject(user)");
    assert("IsArray(user.posts)");
    
    // Test association creation
    post = user.posts().create(title="Test Post", body="Content");
    assert("IsObject(post)");
    assert("post.userId == user.id");
  }
}
```

### Controller Tests
```cfml
component extends="wheels.Test" {
  
  function test_posts_index() {
    // Create test data
    model("Post").create(title="Test Post", body="Content");
    
    // Make request
    result = processRequest(route="posts", method="GET");
    
    // Assert response
    assert("result.status == 200");
    assert("IsArray(result.posts)");
    assert("ArrayLen(result.posts) > 0");
  }
  
  function test_create_post_with_validation_errors() {
    params = {post: {title: ""}};  // Invalid data
    
    result = processRequest(
      route="posts", 
      method="POST", 
      params=params
    );
    
    // Should render new form with errors
    assert("result.template == 'new'");
    assert("IsObject(result.post)");
    assert("result.post.hasErrors()");
  }
  
  function test_api_authentication() {
    // Test without token
    result = processRequest(
      route="apiPosts",
      method="GET",
      headers={}
    );
    assert("result.status == 401");
    
    // Test with valid token
    result = processRequest(
      route="apiPosts",
      method="GET", 
      headers={"Authorization": "Bearer valid-token"}
    );
    assert("result.status == 200");
  }
}
```

## Security Patterns

### Input Sanitization
```cfml
component extends="Controller" {
  
  function config() {
    filters("sanitizeInput");
  }
  
  private function sanitizeInput() {
    // Sanitize text inputs
    for (key in params) {
      if (IsSimpleValue(params[key])) {
        params[key] = htmlEditFormat(params[key]);
      }
    }
    
    // Additional cleaning for specific fields
    if (StructKeyExists(params, "email")) {
      params.email = lcase(trim(params.email));
    }
  }
}
```

### SQL Injection Prevention
```cfml
// ALWAYS use parameterized queries
result = queryExecute(
  sql="SELECT * FROM users WHERE email = ? AND active = ?",
  params=[
    {value: userEmail, cfsqltype: "cf_sql_varchar"},
    {value: true, cfsqltype: "cf_sql_boolean"}
  ]
);

// Or use Wheels ORM (automatically parameterized)
user = model("User").findOne(where="email = ? AND active = ?", whereParams=[userEmail, true]);
```

## Performance Patterns

### Query Optimization
```cfml
// Use select to limit columns
posts = model("Post").findAll(
  select="id,title,excerpt,createdAt,userId",
  order="createdAt DESC"
);

// Use include to avoid N+1 queries  
posts = model("Post").findAll(
  include="author,category",
  order="createdAt DESC"
);

// Use pagination for large datasets
posts = model("Post").findAll(
  order="createdAt DESC",
  page=params.page ?: 1,
  perPage=20
);
```

### Caching Strategies
```cfml
component extends="Controller" {
  
  function index() {
    // Cache expensive queries
    posts = $cacheRead(key="recent_posts");
    
    if (!IsArray(posts)) {
      posts = model("Post").findAll(
        where="published = ?",
        whereParams=[true],
        order="createdAt DESC",
        maxRows=10,
        include="author,category"
      );
      
      $cacheWrite(key="recent_posts", value=posts, timeSpan=createTimeSpan(0,1,0,0));
    }
    
    renderWith(posts);
  }
}
```

## Common Anti-Patterns to Avoid

### Framework Violations
```cfml
// WRONG - calling framework internals
user.$save();
user.$init();

// CORRECT - using public API
user.save();
// (init is handled by framework)

// WRONG - using init() for configuration
function init() {
  hasMany("posts");  // Won't work
}

// CORRECT - using config() for setup
function config() {
  hasMany("posts");  // Proper way
}
```

### Performance Issues
```cfml
// WRONG - N+1 query problem
posts = model("Post").findAll();
for (post in posts) {
  author = post.author();  // Separate query for each post
}

// CORRECT - eager loading
posts = model("Post").findAll(include="author");
for (post in posts) {
  author = post.author();  // Already loaded
}
```

### Security Vulnerabilities
```cfml
// WRONG - SQL injection risk
queryExecute("SELECT * FROM users WHERE name = '#params.name#'");

// CORRECT - parameterized query
queryExecute(
  "SELECT * FROM users WHERE name = ?", 
  [{value: params.name, cfsqltype: "cf_sql_varchar"}]
);
```

## Agent OS Integration Patterns

### Task-Based Development
When implementing Wheels features in Agent OS workflows:

1. **Start with Model**: Define data structure and relationships
2. **Add Migrations**: Create database schema changes
3. **Build Controller**: Implement business logic and request handling
4. **Create Views**: Build user interface
5. **Write Tests**: Ensure functionality works correctly

### Context-Aware Code Generation
Agent OS should automatically detect:
- Existing model associations when generating controllers
- Database schema when creating models  
- Route patterns when generating views
- Testing patterns when creating test files

### Multi-Engine Validation
For Agent OS test runners working with Wheels:
- Test against multiple CFML engines (Lucee, Adobe CF, BoxLang)
- Validate database adapter compatibility
- Check for engine-specific syntax issues
- Ensure consistent behavior across platforms

</conditional-block>