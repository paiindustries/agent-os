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

## CLI Template Variable Patterns

### Template Variable System
Wheels CLI uses a sophisticated template variable system for code generation:

#### Variable Naming Conventions
```
|ObjectNameSingular|     - Lowercase singular (e.g., "user", "post")
|ObjectNameSingularC|    - Capitalized singular (e.g., "User", "Post")
|ObjectNamePlural|       - Lowercase plural (e.g., "users", "posts")
|ObjectNamePluralC|      - Capitalized plural (e.g., "Users", "Posts")
|tableName|              - Database table name
|appName|                - Application name
|DBMigrateExtends|       - Migration base class
|DBMigrateDescription|   - Migration description
|Action|                 - Action method name
|ActionHint|             - Action description
```

#### CLI Marker Comments
Templates use special comments for dynamic content insertion:
```cfm
<!-- CLI-Appends-Here -->        - General append marker
<!-- CLI-Appends-thead-Here -->  - Table header append point
<!-- CLI-Appends-tbody-Here -->  - Table body append point
|Actions|                       - Controller action methods
|FormFields|                    - Form field generation
|arguments|                     - Function arguments
```

#### Template Variable Usage Examples
```cfml
// Controller template with variables
component extends="Controller" {
  function index() {
    |ObjectNamePlural| = model("|ObjectNameSingular|").findAll();
  }
}

// View template with variables
<h1>|ObjectNamePluralC| index</h1>
#linkTo(route="new|ObjectNameSingularC|", text="Create New |ObjectNameSingularC|")#

// Migration template with variables
component extends="|DBMigrateExtends|" hint="|DBMigrateDescription|" {
  function up() {
    createTable(name='|tableName|' |arguments|);
  }
}
```

#### Dynamic Content Replacement
CLI generators dynamically replace these variables:
- Text transformation (singular/plural, case changes)
- Context-aware content injection at marker points
- Argument list generation based on column definitions
- Route name generation following REST conventions

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

### API Filter Patterns

#### Force JSON Response Filter  
Common pattern to ensure all API responses are JSON format:

```cfml
component extends="Controller" {
  
  function config() {
    // Only provide JSON responses
    onlyProvides("json");
    
    // Apply JSON filter to all actions
    filters(through="setJsonResponse");
  }
  
  /**
   * Private filter to force JSON format
   * Ensures params.format is always "json" for API endpoints
   */
  private function setJsonResponse() {
    params.format = "json";
  }
}
```

#### API Authentication Filter
Pattern for token-based API authentication:

```cfml
component extends="Controller" {
  
  function config() {
    onlyProvides("json");
    filters(through="setApiHeaders,authenticateToken");
  }
  
  private function setApiHeaders() {
    header name="X-API-Version" value="1.0";
    header name="Content-Type" value="application/json";
    params.format = "json";
  }
  
  private function authenticateToken() {
    local.authHeader = $getRequestHeader("Authorization");
    
    if (!Len(local.authHeader) || !local.authHeader.startsWith("Bearer ")) {
      renderWith(data={error="Authentication required"}, status=401);
      return;
    }
    
    local.token = local.authHeader.replace("Bearer ", "");
    
    // Validate token (implement your validation logic)
    if (!isValidToken(local.token)) {
      renderWith(data={error="Invalid token"}, status=401);
      return;
    }
    
    // Set current user or context based on token
    request.currentUserId = getUserIdFromToken(local.token);
  }
  
  private boolean function isValidToken(string token) {
    // Implement token validation logic
    return true;  // Placeholder
  }
  
  private numeric function getUserIdFromToken(string token) {
    // Implement user extraction from token
    return 1;  // Placeholder
  }
}
```

#### API Error Handling Filter
Standardized error responses for APIs:

```cfml
component extends="Controller" {
  
  function config() {
    onlyProvides("json");
    filters(through="setApiHeaders,handleApiErrors", type="after");
  }
  
  private function handleApiErrors() {
    // Check if an error occurred during processing
    if ($hasErrors()) {
      local.errors = $allErrors();
      
      renderWith(
        data={
          error="Validation failed",
          errors=local.errors,
          timestamp=now()
        },
        status=422
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

## CRUD View Conventions

### Standard CRUD View File Structure
Wheels follows consistent naming conventions for CRUD views:

```
views/
  controllerName/
    index.cfm         - List all records
    show.cfm          - Display single record
    new.cfm           - Create new record form
    edit.cfm          - Edit existing record form
    _form.cfm         - Partial for form fields (shared by new/edit)
```

### Index View Pattern
Standard listing view with table layout and action links:

```cfm
<!--- posts/index.cfm --->
<cfparam name="posts">
<cfoutput>
  <h1>Posts</h1>
  
  <p>#linkTo(route="newPost", text="Create New Post", class="btn btn-primary")#</p>
  
  <cfif posts.recordcount>
    <table class="table">
      <thead>
        <tr>
          <th>Title</th>
          <th>Author</th>
          <th>Created</th>
          <!--- CLI-Appends-thead-Here --->
          <th>Actions</th>
        </tr>
      </thead>
      <tbody>
        <cfloop query="posts">
        <tr>
          <td>#posts.title#</td>
          <td>#posts.authorName#</td>
          <td>#dateFormat(posts.createdAt, "mmm d, yyyy")#</td>
          <!--- CLI-Appends-tbody-Here --->
          <td>
            <div class="btn-group">
              #linkTo(route="post", key=posts.id, text="View", class="btn btn-xs btn-info")#
              #linkTo(route="editPost", key=posts.id, text="Edit", class="btn btn-xs btn-primary")#
              #buttonTo(route="post", method="delete", key=posts.id, text="Delete", 
                       class="pull-right", inputClass="btn btn-danger btn-xs")#
            </div>
          </td>
        </tr>
        </cfloop>
      </tbody>
    </table>
  <cfelse>
    <p>Sorry, there are no posts yet</p>
    <p>#linkTo(route="newPost", text="Create the first post", class="btn btn-success")#</p>
  </cfif>
</cfoutput>
```

### Show View Pattern
Single record display with navigation links:

```cfm
<!--- posts/show.cfm --->
<cfparam name="post">
<cfoutput>
  <article class="post">
    <h1>#post.title#</h1>
    
    <div class="post-meta">
      <p>By #post.author().name# on #dateFormat(post.createdAt, "mmmm d, yyyy")#</p>
      <cfif post.updatedAt NEQ post.createdAt>
        <p><small>Updated #dateFormat(post.updatedAt, "mmmm d, yyyy")#</small></p>
      </cfif>
    </div>
    
    <div class="post-content">
      #post.body#
    </div>
    
    <div class="post-actions">
      #linkTo(route="posts", text="← Back to Posts", class="btn btn-default")#
      #linkTo(route="editPost", key=post.key(), text="Edit", class="btn btn-primary")#
      #buttonTo(route="post", method="delete", key=post.key(), text="Delete", 
                class="btn btn-danger", confirm="Are you sure?")#
    </div>
  </article>
</cfoutput>
```

### New/Edit Form Pattern
Forms with shared partial for DRY principles:

```cfm
<!--- posts/new.cfm --->
<cfparam name="post">
<cfoutput>
  <h1>Create New Post</h1>
  
  #startFormTag(route="posts", method="post")#
    #includePartial("form")#
    
    <div class="form-actions">
      #submitTag(value="Create Post", class="btn btn-primary")#
      #linkTo(route="posts", text="Cancel", class="btn btn-default")#
    </div>
  #endFormTag()#
</cfoutput>

<!--- posts/edit.cfm --->
<cfparam name="post">
<cfoutput>
  <h1>Edit Post</h1>
  
  #startFormTag(route="post", method="put", key=post.key())#
    #includePartial("form")#
    
    <div class="form-actions">
      #submitTag(value="Update Post", class="btn btn-primary")#
      #linkTo(route="post", key=post.key(), text="Cancel", class="btn btn-default")#
    </div>
  #endFormTag()#
</cfoutput>
```

### Form Partial Pattern
Reusable form fields shared between new and edit views:

```cfm
<!--- posts/_form.cfm --->
<cfoutput>
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
    #select(
      objectName="post",
      property="categoryId",
      options=categories,
      includeBlank="-- Select Category --",
      label="Category",
      class="form-control"
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
  
  <div class="form-check">
    #checkBox(
      objectName="post", 
      property="published", 
      label="Publish immediately"
    )#
  </div>
  
  <!--- CLI-Appends-Here --->
</cfoutput>
```

### CLI Append Markers
Views use special markers for CLI code generation:

- `<!--- CLI-Appends-Here --->` - General append point for additional fields
- `<!--- CLI-Appends-thead-Here --->` - Table header column insertion  
- `<!--- CLI-Appends-tbody-Here --->` - Table body column insertion

### View Helper Patterns

#### Conditional Content Display
```cfm
<cfoutput>
  <!--- Show edit link only if user has permission --->
  <cfif hasPermission("edit", post)>
    #linkTo(route="editPost", key=post.key(), text="Edit")#
  </cfif>
  
  <!--- Conditional CSS classes --->
  <div class="post #post.published ? 'published' : 'draft'#">
    #post.title#
  </div>
  
  <!--- Safe display of optional fields --->
  <cfif Len(post.excerpt)>
    <p class="excerpt">#post.excerpt#</p>
  </cfif>
</cfoutput>
```

#### Partial Rendering with Query Data
```cfm
<cfoutput>
  <!--- Render partial for each record in query --->
  <div class="post-list">
    #includePartial(partial="post_card", query=posts)#
  </div>
  
  <!--- Render partial with individual object --->
  #includePartial(partial="comment", comment=post.latestComment())#
</cfoutput>
```

## Bootstrap Configuration Patterns

### Global Form Helper Settings
Configure Bootstrap 3.x form styling using the `set()` function in your environment configuration:

```cfml
// Bootstrap form settings (typically in config/settings.cfm or environment files)

// Submit button defaults
set(functionName="submitTag", class="btn btn-primary", value="Save Changes");

// Checkbox styling with Bootstrap wrapper
set(
  functionName="checkBox,checkBoxTag", 
  labelPlacement="aroundRight",
  prependToLabel='<div class="checkbox">',
  appendToLabel="</div>",
  uncheckedValue="0"
);

// Radio button styling with Bootstrap wrapper  
set(
  functionName="radioButton,radioButtonTag",
  labelPlacement="aroundRight", 
  prependToLabel='<div class="radio">',
  appendToLabel="</div>"
);

// Text inputs with form-control class and form-group wrapper
set(
  functionName="textField,textFieldTag,select,selectTag,passwordField,passwordFieldTag,textArea,textAreaTag,fileFieldTag,fileField",
  class="form-control",
  labelClass="control-label",
  labelPlacement="before",
  prependToLabel='<div class="form-group">',
  prepend='<div class="">',
  append="</div></div>",
  encode="attributes"
);

// Date/time selects with Bootstrap styling
set(
  functionName="dateTimeSelect,dateSelect",
  prepend='<div class="form-group">',
  append="</div>",
  timeSeparator="",
  minuteStep="5",
  secondStep="10",
  dateOrder="day,month,year",
  dateSeparator="",
  separator=""
);

// Error message styling  
set(
  functionName="errorMessagesFor",
  class='alert alert-dismissable alert-danger" style="margin-left:0;padding-left:26px;"'
);

set(
  functionName="errorMessageOn",
  wrapperElement="div",
  class="alert alert-danger"
);

// Pagination with Bootstrap styling
set(
  functionName="paginationLinks",
  prepend='<ul class="pagination">',
  append="</ul>",
  prependToPage="<li>",
  appendToPage="</li>",
  linkToCurrentPage=true,
  classForCurrent="active",
  anchorDivider='<li class="disabled"><a href="##">...</a></li>'
);
```

### Bootstrap Layout Template
Standard Bootstrap layout with CDN assets:

```cfm
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  
  <cfoutput>#csrfMetaTags()#</cfoutput>
  <title>|appName|</title>
  
  <!-- Bootstrap CSS -->
  <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css" 
        integrity="sha384-BVYiiSIFeK1dGmJRAkycuHAHRg32OmUcww7on3RYdg4Va+PmSTsz/K68vbdEjh4u" 
        crossorigin="anonymous">
  
  <!-- Bootstrap Theme -->
  <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap-theme.min.css" 
        integrity="sha384-rHyoN1iRsVXV4nD0JutlnGaslCJuC7uwjduW9SVrLvRYooPp2bWYgmgJQIXwl/Sp" 
        crossorigin="anonymous">
  
  <!-- HTML5 shim for IE8 support -->
  <!--[if lt IE 9]>
    <script src="https://oss.maxcdn.com/libs/html5shiv/3.7.0/html5shiv.js"></script>
    <script src="https://oss.maxcdn.com/libs/respond.js/1.4.2/respond.min.js"></script>
  <![endif]-->
  
  <link rel="shortcut icon" href="/favicon.ico">
</head>
<body>

<header>
  <div class="container">
    <h1 class="site-title">
      <cfoutput>#linkTo(route="root", text="|appName|")#</cfoutput>
    </h1>
  </div>
</header>

<div id="content" class="container">
  <cfoutput>#flashMessages()#</cfoutput>
  <section>
    <cfoutput>#includeContent()#</cfoutput>
  </section>
</div>

<footer>
  <!-- Footer content -->
</footer>

<!-- Bootstrap JavaScript -->
<cfoutput>
#javascriptIncludeTag("https://ajax.googleapis.com/ajax/libs/jquery/1.12.4/jquery.min.js,https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/js/bootstrap.min.js")#
</cfoutput>

</body>
</html>
```

### Form Styling with Bootstrap Classes
When using Wheels form helpers with Bootstrap configuration:

```cfm
<cfoutput>
<!-- Form will automatically use Bootstrap classes from global settings -->
#startFormTag(route="users", method="post", class="user-form")#

  <!-- Text field with automatic form-group wrapper and form-control class -->
  #textField(objectName="user", property="name", label="Full Name", required=true)#
  
  <!-- Email field with validation styling -->
  #textField(objectName="user", property="email", label="Email Address", type="email")#
  
  <!-- Select with form-control class -->
  #select(objectName="user", property="roleId", options=roles, includeBlank="-- Select Role --", label="Role")#
  
  <!-- Checkbox with Bootstrap checkbox wrapper -->
  #checkBox(objectName="user", property="active", label="Active User")#
  
  <!-- Submit with btn btn-primary class -->
  #submitTag(value="Create User")#
  
#endFormTag()#
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

### Transaction Pattern with Error Handling
All migrations should use this standard transaction pattern:
```cfml
component extends="Migration" {
  
  function up() {
    transaction {
      try {
        // Migration operations here
        addColumn(
          table="users",
          columnType="string", 
          columnName="status",
          limit=50,
          default="active"
        );
        
      } catch (any e) {
        local.exception = e;
      }
      
      if (StructKeyExists(local, "exception")) {
        transaction action="rollback";
        throw(
          errorCode="1",
          detail=local.exception.detail,
          message=local.exception.message,
          type="any"
        );
      } else {
        transaction action="commit";
      }
    }
  }
  
  function down() {
    transaction {
      try {
        // Reverse operations here
        removeColumn(table="users", columnName="status");
        
      } catch (any e) {
        local.exception = e;
      }
      
      if (StructKeyExists(local, "exception")) {
        transaction action="rollback";
        throw(
          errorCode="1",
          detail=local.exception.detail,
          message=local.exception.message,
          type="any"
        );
      } else {
        transaction action="commit";
      }
    }
  }
}
```

### Additional Migration Types

#### Add Column Migration
```cfml
// Parameters: table, columnType, columnName, default, null, limit, precision, scale
component extends="Migration" {
  function up() {
    transaction {
      try {
        addColumn(
          table="products",
          columnType="decimal",
          columnName="price",
          precision=10,
          scale=2,
          null=false,
          default=0.00
        );
      } catch (any e) {
        local.exception = e;
      }
      // ... standard error handling
    }
  }
  
  function down() {
    transaction {
      try {
        removeColumn(table="products", columnName="price");
      } catch (any e) {
        local.exception = e;
      }
      // ... standard error handling
    }
  }
}
```

#### Change Column Migration
```cfml
component extends="Migration" {
  function up() {
    transaction {
      try {
        changeColumn(
          table="users",
          columnName="email",
          columnType="string",
          limit=320,  // Increase limit
          null=false
        );
      } catch (any e) {
        local.exception = e;
      }
      // ... standard error handling
    }
  }
  
  function down() {
    transaction {
      try {
        changeColumn(
          table="users",
          columnName="email",
          columnType="string",
          limit=255,  // Original limit
          null=false
        );
      } catch (any e) {
        local.exception = e;
      }
      // ... standard error handling
    }
  }
}
```

#### Create Index Migration
```cfml
component extends="Migration" {
  function up() {
    transaction {
      try {
        addIndex(
          table="posts",
          columnNames="userId,createdAt",
          indexName="idx_posts_user_created"
        );
      } catch (any e) {
        local.exception = e;
      }
      // ... standard error handling
    }
  }
  
  function down() {
    transaction {
      try {
        removeIndex(
          table="posts",
          indexName="idx_posts_user_created"
        );
      } catch (any e) {
        local.exception = e;
      }
      // ... standard error handling
    }
  }
}
```

#### Remove Table Migration
```cfml
component extends="Migration" {
  function up() {
    transaction {
      try {
        dropTable("legacy_data");
      } catch (any e) {
        local.exception = e;
      }
      // ... standard error handling
    }
  }
  
  function down() {
    transaction {
      try {
        // Recreate table structure if needed
        createTable(name="legacy_data") {
          t.string(columnNames="name", limit=100);
          t.text(columnNames="data");
          t.timestamps();
        };
      } catch (any e) {
        local.exception = e;
      }
      // ... standard error handling
    }
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

### TestBox-Specific Patterns

#### Basic Test Suite Structure
Standard TestBox BDD structure using Wheels BaseSpec:

```cfml
component extends="testbox.system.BaseSpec" {
  
  function run() {
    
    describe("User Model Tests", () => {
      
      beforeEach(() => {
        // Setup data for each test
        variables.testUser = model("User").new({
          name: "Test User",
          email: "test@example.com",
          password: "password123"
        });
      });
      
      afterEach(() => {
        // Cleanup after each test
        if (StructKeyExists(variables, "testUser") && IsObject(variables.testUser)) {
          variables.testUser.delete();
        }
      });
      
      it("should validate required fields", () => {
        var user = model("User").new();
        
        expect(user.valid()).toBeFalse("User should not be valid without required fields");
        expect(user.errorsOn("name")).toHaveLength(1, "Should have name validation error");
        expect(user.errorsOn("email")).toHaveLength(1, "Should have email validation error");
      });
      
      it("should validate email format", () => {
        variables.testUser.email = "invalid-email";
        
        expect(variables.testUser.valid()).toBeFalse("User should not be valid with invalid email");
        expect(variables.testUser.errorsOn("email")).toHaveLength(1, "Should have email format error");
      });
      
      it("should save valid user", () => {
        expect(variables.testUser.save()).toBeTrue("Valid user should save successfully");
        expect(variables.testUser.hasErrors()).toBeFalse("Saved user should not have errors");
        expect(variables.testUser.key()).toBeNumeric("Saved user should have numeric key");
      });
      
    });
  }
}
```

#### Wheels-Aware Test Base Class
Custom base class extending TestBox with Wheels-specific helpers:

```cfml
component extends="testbox.system.BaseSpec" {
  
  function beforeAll() {
    // Setup test database connection
    application.wheels.dataSourceName = "wheels_test";
    
    // Disable caching for tests
    application.wheels.cacheActions = false;
    application.wheels.cachePartials = false;
    application.wheels.cacheQueries = false;
  }
  
  function setup() {
    // Called before each test method
    super.setup();
    
    // Start transaction for test isolation
    this.$wheels_test_transaction = true;
    queryExecute("BEGIN TRANSACTION");
  }
  
  function teardown() {
    // Called after each test method
    super.teardown();
    
    // Rollback transaction to clean up test data
    if (StructKeyExists(this, "$wheels_test_transaction")) {
      queryExecute("ROLLBACK");
      StructDelete(this, "$wheels_test_transaction");
    }
  }
  
  // Helper method to create test models
  function createTestModel(string modelName, struct properties = {}) {
    return model(arguments.modelName).create(arguments.properties);
  }
  
  // Helper method for controller testing
  function processTestRequest(required string route, string method = "GET", struct params = {}) {
    local.originalParams = Duplicate(params);
    local.originalCGI = Duplicate(CGI);
    
    try {
      // Mock request parameters
      params.clear();
      params.putAll(arguments.params);
      
      // Set request method
      CGI.REQUEST_METHOD = arguments.method;
      
      // Process the request (simplified)
      local.result = $request(argumentCollection=arguments);
      
      return local.result;
      
    } finally {
      // Restore original state
      params.clear();
      params.putAll(local.originalParams);
      CGI = local.originalCGI;
    }
  }
}
```

#### Model Test with Factory Pattern
Using factories for consistent test data:

```cfml
component extends="WheelsTestBase" {
  
  function run() {
    
    describe("Post Model with Associations", () => {
      
      beforeEach(() => {
        // Create test data using factories
        variables.testAuthor = createTestModel("User", {
          name: "John Doe",
          email: "john@example.com"
        });
        
        variables.testCategory = createTestModel("Category", {
          name: "Technology"
        });
      });
      
      it("should create post with associations", () => {
        var post = model("Post").create({
          title: "Test Post",
          body: "This is a test post",
          userId: variables.testAuthor.id,
          categoryId: variables.testCategory.id
        });
        
        expect(post.hasErrors()).toBeFalse("Post should save without errors");
        expect(post.author().name).toBe("John Doe", "Should load associated author");
        expect(post.category().name).toBe("Technology", "Should load associated category");
      });
      
      it("should validate associations exist", () => {
        var post = model("Post").create({
          title: "Test Post",
          body: "This is a test post",
          userId: 99999,  // Non-existent user
          categoryId: variables.testCategory.id
        });
        
        expect(post.hasErrors()).toBeTrue("Post should have validation errors");
        expect(post.errorsOn("userId")).toHaveLength(1, "Should have user association error");
      });
      
    });
  }
}
```

#### Controller Test with HTTP Mocking
Testing controller actions with mocked requests:

```cfml
component extends="WheelsTestBase" {
  
  function run() {
    
    describe("Posts Controller", () => {
      
      beforeEach(() => {
        // Create test data
        variables.testUser = createTestModel("User", {
          name: "Test User",
          email: "test@example.com"
        });
        
        variables.testPost = createTestModel("Post", {
          title: "Sample Post",
          body: "Sample content",
          userId: variables.testUser.id
        });
      });
      
      describe("GET /posts", () => {
        
        it("should display posts index", () => {
          var result = processTestRequest(route="posts", method="GET");
          
          expect(result).toHaveKey("posts", "Should include posts in response");
          expect(result.posts.recordCount).toBeGTE(1, "Should have at least one post");
        });
        
      });
      
      describe("POST /posts", () => {
        
        it("should create new post with valid data", () => {
          var params = {
            post: {
              title: "New Test Post",
              body: "New test content",
              userId: variables.testUser.id
            }
          };
          
          var result = processTestRequest(route="posts", method="POST", params=params);
          
          expect(result.post.hasErrors()).toBeFalse("Post should be created without errors");
          expect(result.post.title).toBe("New Test Post", "Should save correct title");
        });
        
        it("should show errors with invalid data", () => {
          var params = {
            post: {
              title: "",  // Required field empty
              body: "Some content"
            }
          };
          
          var result = processTestRequest(route="posts", method="POST", params=params);
          
          expect(result.post.hasErrors()).toBeTrue("Post should have validation errors");
          expect(result.template).toBe("new", "Should render new template with errors");
        });
        
      });
      
    });
  }
}
```

#### Integration Test Pattern
Full stack testing with database and HTTP:

```cfml
component extends="testbox.system.BaseSpec" {
  
  function run() {
    
    describe("User Registration Integration", () => {
      
      beforeEach(() => {
        // Clean slate for integration tests
        queryExecute("DELETE FROM users WHERE email LIKE 'integration_test_%'");
      });
      
      it("should register user end-to-end", () => {
        var registrationData = {
          user: {
            name: "Integration Test User",
            email: "integration_test_#createUUID()#@example.com",
            password: "securePassword123",
            passwordConfirmation: "securePassword123"
          }
        };
        
        // Submit registration form
        var response = processRequest(
          route="users",
          method="POST",
          params=registrationData
        );
        
        // Verify user was created in database
        var createdUser = model("User").findOne(where="email = ?", 
                                               whereParams=[registrationData.user.email]);
        
        expect(IsObject(createdUser)).toBeTrue("User should be created in database");
        expect(createdUser.name).toBe(registrationData.user.name, "Should save correct name");
        
        // Verify redirect to success page
        expect(response.redirectedTo).toContain("login", "Should redirect to login page");
      });
      
    });
  }
}
```

#### Test Factory Helpers
Reusable factory methods for test data:

```cfml
component {
  
  function userFactory(struct overrides = {}) {
    var defaults = {
      name: "Test User #randRange(1, 9999)#",
      email: "test#randRange(1, 9999)#@example.com",
      password: "password123",
      status: "active"
    };
    
    defaults.putAll(overrides);
    
    return model("User").create(defaults);
  }
  
  function postFactory(struct overrides = {}) {
    // Ensure we have a user for the post
    if (!StructKeyExists(overrides, "userId")) {
      var user = userFactory();
      overrides.userId = user.id;
    }
    
    var defaults = {
      title: "Test Post #randRange(1, 9999)#",
      body: "This is test content for the post.",
      published: true,
      publishedAt: now()
    };
    
    defaults.putAll(overrides);
    
    return model("Post").create(defaults);
  }
  
  function categoryFactory(struct overrides = {}) {
    var defaults = {
      name: "Test Category #randRange(1, 9999)#",
      description: "Test category description"
    };
    
    defaults.putAll(overrides);
    
    return model("Category").create(defaults);
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

## Configuration File Templates

### Routes Configuration Template
Standard routes.cfm with CLI integration markers:

```cfml
<cfscript>
  // Use this file to add routes to your application and point the root route to a controller action.
  // Don't forget to issue a reload request (e.g. reload=true) after making changes.
  // See https://guides.wheels.dev/docs/routing for more info.

  mapper()
    // CLI-Appends-Here
    
    // RESTful resource examples:
    // .resources("posts")
    // .resources("users", except="delete")
    // .namespace("admin")
    //   .resources("products")
    // .end()
    
    // The "wildcard" call enables automatic mapping of "controller/action" type routes.
    // This way you don't need to explicitly add a route every time you create a new action.
    .wildcard()

    // The root route is called on your application's home page (e.g. http://127.0.0.1/).
    // Change "wheels##wheels" to "home##index" to call the "index" action on the "home" controller.
    .root(to="wheels##wheels", method="get")
  .end();
</cfscript>
```

### Application Configuration Template
Basic Application.cfc structure for Wheels:

```cfml
component extends="wheels.Application" output="false" {
  
  this.name = "|appName|";
  this.applicationTimeout = createTimeSpan(0,0,30,0);
  this.sessionManagement = true;
  this.sessionTimeout = createTimeSpan(0,2,0,0);
  this.setClientCookies = true;
  this.setDomainCookies = false;
  
  // Wheels-specific settings
  this.wheelsReloadPassword = "|reloadPassword|";
  
  // Mappings
  this.mappings["/wheels"] = expandPath("vendor/wheels");
  this.mappings["/config"] = expandPath("config");
  
  // Data source configuration
  this.datasource = "|datasourceName|";
  
  function onApplicationStart() {
    super.onApplicationStart();
    // Application startup logic here
    return true;
  }
  
  function onRequestStart(required string targetPage) {
    super.onRequestStart(argumentCollection=arguments);
    // Request initialization logic here
    return true;
  }
  
  function onError(required any exception, required string eventName) {
    super.onError(argumentCollection=arguments);
    // Error handling logic here
  }
}
```

### DataSource Configuration (H2)
H2 database configuration snippet:

```cfml
// H2 embedded database configuration
this.datasources["qrplatform"] = {
  type: "h2",
  database: "db/h2/qrplatform.h2.db",
  dbcreate: "update"
};

// Alternative file-based H2 configuration
this.datasources["|datasourceName|"] = {
  class: "org.h2.Driver",
  connectionString: "jdbc:h2:file:#expandPath('db/h2/|datasourceName|.h2.db')#;MODE=MySQL",
  username: "",
  password: ""
};
```

### Server Configuration (server.json)
CommandBox server configuration template:

```json
{
  "name": "|appName|",
  "openBrowser": false,
  "web": {
    "http": {
      "port": 8080
    },
    "rewrites": {
      "enable": true
    },
    "directoryBrowsing": false
  },
  "app": {
    "cfengine": "lucee@5",
    "serverHomeDirectory": ".engine"
  },
  "runwar": {
    "args": "--enable-ssl-client-auth"
  }
}
```

### Box.json Package Configuration
CommandBox package descriptor for Wheels projects:

```json
{
  "name": "|appName|",
  "version": "1.0.0",
  "author": "",
  "homepage": "",
  "documentation": "",
  "repository": {
    "type": "git",
    "URL": ""
  },
  "bugs": "",
  "shortDescription": "",
  "description": "",
  "instructions": "",
  "changelog": "",
  "type": "wheels",
  "keywords": [
    "wheels",
    "cfml",
    "framework"
  ],
  "private": false,
  "engines": [
    {
      "type": "lucee",
      "version": ">=5.3"
    },
    {
      "type": "adobe",
      "version": ">=2018"
    }
  ],
  "defaultEngine": "lucee@5",
  "license": [
    {
      "type": "MIT",
      "URL": "https://opensource.org/licenses/MIT"
    }
  ],
  "dependencies": {},
  "devDependencies": {
    "testbox": "^4.3.0",
    "wheels-cli": "^3.0.0"
  },
  "installPaths": {
    "testbox": "testbox/",
    "wheels": "vendor/wheels/"
  },
  "testbox": {
    "runner": [
      "http://localhost:8080/tests/runner.cfm"
    ],
    "directory": {
      "mapping": "tests",
      "recurse": true
    }
  },
  "scripts": {
    "test": "testbox run",
    "test:watch": "testbox run --watch",
    "server:start": "server start",
    "server:stop": "server stop",
    "format": "cfformat run --overwrite",
    "format:check": "cfformat check"
  }
}
```

### Environment-Specific Settings
Pattern for environment configuration files:

```cfml
<cfscript>
  // config/environments/development.cfm
  set(dataSourceName="wheels_dev");
  set(errorEmailAddress="developer@example.com");
  set(showErrorInformation=true);
  set(showDebugInformation=true);
  set(enableFlashMessages=true);
  set(cacheActions=false);
  set(cacheImages=false);
  set(cachePartials=false);
  set(cacheQueries=false);
  
  // Development-specific plugins or settings
  set(enableMigratorComponent=true);
  set(allowRebuildRoutes=true);
</cfscript>

<cfscript>
  // config/environments/production.cfm
  set(dataSourceName="wheels_prod");
  set(errorEmailAddress="admin@example.com");
  set(showErrorInformation=false);
  set(showDebugInformation=false);
  set(enableFlashMessages=true);
  set(cacheActions=true);
  set(cacheImages=true);
  set(cachePartials=true);
  set(cacheQueries=true);
  
  // Production optimizations
  set(enableMigratorComponent=false);
  set(allowRebuildRoutes=false);
</cfscript>
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