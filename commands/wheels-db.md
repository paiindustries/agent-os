# Wheels Database Management

Manage database operations for Wheels CFML applications including migrations, setup, reset, and maintenance tasks.

## Usage

This command provides comprehensive database management for Wheels applications, handling migrations, data seeding, and database maintenance through Agent OS workflows.

### Basic Database Commands

```bash
# Run pending migrations
/wheels-db migrate

# Create database and run all migrations
/wheels-db setup

# Reset database (drop, create, migrate, seed)
/wheels-db reset

# Check migration status
/wheels-db status

# Rollback migrations
/wheels-db rollback --steps=3

# Database shell access
/wheels-db shell
```

## Implementation Workflow

When this command is executed, follow these steps:

### 1. Database Context Analysis
```
ANALYZE current database state:
- Check if database exists and is accessible
- Identify current schema version
- Detect pending migrations
- Verify database adapter compatibility
- Check for data safety requirements
```

### 2. Operation Execution

Based on the requested operation:

#### Migration Operations
```
FOR migration commands:
1. USE wheels-migration agent for validation
2. EXECUTE CommandBox/Wheels CLI commands
3. MONITOR for errors and rollback if needed
4. VERIFY migration success
5. UPDATE schema version tracking
```

#### Database Setup
```
FOR setup operations:
1. CREATE database if not exists
2. CONFIGURE datasource settings
3. RUN all pending migrations in order
4. SEED database with initial data
5. VERIFY complete setup
```

#### Database Reset
```
FOR reset operations (DANGEROUS):
1. CONFIRM reset operation with user
2. BACKUP existing data (optional)
3. DROP all tables and constraints
4. RECREATE database structure
5. RUN all migrations from scratch
6. SEED with fresh data
```

### 3. Multi-Engine Database Support

Handle different database adapters appropriately:

#### H2 Database (Development)
```bash
# H2 specific operations
wheels db create --adapter=H2
wheels db shell --web  # Opens H2 web console

# H2 characteristics:
- File-based or in-memory
- Great for development and testing
- Limited constraint support
- Case-insensitive by default
```

#### MySQL Operations
```bash
# MySQL specific operations  
wheels db create --adapter=MySQL --host=localhost --database=myapp_dev

# MySQL considerations:
- Case-sensitive table names on Linux
- Full-text search capabilities
- Specific charset/collation settings
- MyISAM vs InnoDB engine differences
```

#### PostgreSQL Operations
```bash
# PostgreSQL specific operations
wheels db create --adapter=PostgreSQL --host=localhost --database=myapp_dev

# PostgreSQL advantages:
- Advanced constraint support
- JSON and array column types
- Sophisticated indexing options
- Strong ACID compliance
```

#### SQL Server Operations
```bash
# SQL Server specific operations
wheels db create --adapter=SQLServer --server=localhost --database=myapp_dev

# SQL Server considerations:
- Windows authentication vs SQL auth
- T-SQL specific syntax
- Identity column handling
- Nonclustered index patterns
```

## Database Management Operations

### Migration Management
```
MIGRATION WORKFLOW:
1. Check migration status: wheels db status
2. Review pending migrations
3. Backup database before major changes
4. Run migrations: wheels dbmigrate latest
5. Verify schema changes
6. Update application if needed
7. Test application functionality
```

### Schema Verification
```
AFTER migrations, verify:
- All tables created successfully
- Indexes added as specified
- Foreign key constraints in place
- Data types match expectations
- No orphaned or missing columns
```

### Data Management
```
FOR data operations:
1. Seed initial data: wheels db seed
2. Import production data: wheels db import backup.sql
3. Export data: wheels db export --output=backup.sql
4. Anonymize data: wheels db anonymize --tables=users,orders
```

## Database Safety and Backup

### Pre-Operation Safety Checks
```
BEFORE destructive operations:
1. Confirm operation with explicit user consent
2. Check for dependent applications
3. Verify backup availability
4. Assess rollback feasibility
5. Document current state
```

### Backup Operations
```cfml
// Automated backup before major operations
function createBackup(string operation) {
  timestamp = dateFormat(now(), "yyyy-mm-dd") & "-" & timeFormat(now(), "HHmmss");
  backupFile = "backup-#operation#-#timestamp#.sql";
  
  // Execute backup command
  command = "wheels db dump --output=#backupFile#";
  result = runCommand(command);
  
  if (result.success) {
    writeLog("Backup created: #backupFile#", "application");
    return backupFile;
  } else {
    throw("Backup failed: #result.error#");
  }
}
```

### Rollback Planning
```
FOR each operation, plan rollback:
1. Document current state
2. Create recovery scripts
3. Test rollback procedures
4. Set recovery time objectives
5. Define rollback triggers
```

## Advanced Database Operations

### Performance Optimization
```bash
# Analyze database performance
/wheels-db analyze --tables=users,posts,orders

# Rebuild indexes
/wheels-db reindex --tables=all

# Update statistics
/wheels-db stats --update

# Check for missing indexes
/wheels-db missing-indexes
```

### Data Integrity Checks
```bash
# Verify data integrity
/wheels-db check-integrity

# Find orphaned records
/wheels-db find-orphans --tables=posts,comments

# Validate foreign key constraints
/wheels-db check-constraints

# Analyze data distribution
/wheels-db analyze-data --table=users
```

### Multi-Environment Management
```bash
# Environment-specific operations
/wheels-db setup --env=development
/wheels-db migrate --env=testing  
/wheels-db reset --env=staging --confirm

# Cross-environment synchronization
/wheels-db sync-schema --from=production --to=development
/wheels-db copy-data --from=production --to=staging --anonymize
```

## Integration with Agent OS

### Spec-Driven Database Operations
```
IF part of Agent OS specification:
1. READ database requirements from spec
2. GENERATE necessary migrations
3. EXECUTE database operations
4. VERIFY specification compliance
5. UPDATE tasks.md progress
6. COMMIT database changes
```

### Error Handling and Recovery
```
FOR database operation failures:
1. CAPTURE detailed error information
2. ATTEMPT automatic recovery
3. RESTORE from backup if needed
4. REPORT failure with recovery options
5. SUGGEST corrective actions
6. ESCALATE to manual intervention if required
```

## Command Examples

### Development Workflow
```bash
# Set up new development environment
/wheels-db setup
✅ Database 'myapp_dev' created
✅ Ran 15 migrations successfully  
✅ Seeded initial data
✅ Database ready for development

# Add new migration and apply
/wheels-generate migration AddUserPreferences
/wheels-db migrate
✅ Applied 1 new migration
✅ Added user_preferences table
✅ Schema version: 16
```

### Production Deployment
```bash
# Safe production migration
/wheels-db backup --output=pre-deploy-backup.sql
✅ Backup created: pre-deploy-backup.sql (45.2MB)

/wheels-db migrate --dry-run
📋 Would apply 3 migrations:
   - AddIndexToUsers (safe)
   - UpdateProductPricing (data modification)
   - RemoveObsoleteColumns (destructive)

/wheels-db migrate --confirm
✅ Applied 3 migrations successfully
✅ Migration time: 2.3 minutes
✅ No data loss detected
```

### Testing Database Management
```bash
# Reset test database
/wheels-db reset --env=testing --confirm
✅ Dropped all tables
✅ Recreated database structure  
✅ Applied all migrations
✅ Seeded test data
✅ Test database ready

# Parallel test databases
/wheels-db create --env=testing --suffix=_thread_1
/wheels-db create --env=testing --suffix=_thread_2
```

## Error Scenarios and Recovery

### Migration Failures
```
WHEN migration fails:
1. STOP migration process immediately
2. LOG detailed error information
3. ATTEMPT rollback to previous state
4. VERIFY database consistency
5. REPORT failure with specific error
6. SUGGEST manual intervention steps

Example output:
❌ Migration failed: 20240115120000-AddUserIndex.cfc
   Error: Duplicate key name 'idx_users_email'
   Location: Line 15 - addIndex(table="users", columnNames="email")
   
   Recovery options:
   1. Check if index already exists
   2. Remove duplicate index definition
   3. Modify migration to handle existing indexes
   
   Database status: Rolled back to previous state
   Action required: Fix migration and retry
```

### Database Connection Issues
```
WHEN connection fails:
1. VERIFY database server status
2. CHECK connection parameters
3. TEST network connectivity  
4. VALIDATE credentials
5. SUGGEST connection troubleshooting

Example output:
❌ Database connection failed
   Error: Connection refused to localhost:5432
   Database: myapp_dev (PostgreSQL)
   
   Troubleshooting steps:
   1. Check if PostgreSQL server is running
   2. Verify port 5432 is accessible
   3. Check firewall settings
   4. Validate database credentials
   5. Review datasource configuration
```

### Data Corruption Detection
```
WHEN data inconsistency detected:
1. HALT destructive operations
2. REPORT corruption details
3. SUGGEST data recovery options
4. RECOMMEND integrity checks
5. ESCALATE to manual review

Example output:
⚠️  Data inconsistency detected
   Issue: Found 15 orphaned order items
   Impact: References missing products
   
   Recommended actions:
   1. Run data integrity analysis
   2. Identify root cause of orphans
   3. Clean up orphaned records
   4. Add foreign key constraints
   5. Review application logic
```

## Output Format

### Successful Operations
```
🗄️  Wheels Database Operation Complete

Operation: Migration
Environment: development
Database: myapp_dev (H2)
Adapter: H2

📊 Migration Results:
✅ Applied: 3 new migrations
✅ Schema version: 18 → 21
✅ Execution time: 1.2 seconds
✅ Tables added: user_preferences, audit_logs
✅ Indexes created: 5
✅ Constraints added: 2

📋 Schema Summary:
- Total tables: 12
- Total indexes: 28
- Foreign keys: 15
- Database size: 2.1MB

🎯 Next Steps:
- Restart application to recognize schema changes
- Run tests to verify application compatibility
- Update model associations if needed
- Deploy to staging environment

Database ready for continued development.
```

### Operation Warnings
```
⚠️  Database Operation Completed with Warnings

Operation: Reset
Environment: development  
Warnings: 2

⚠️  Warning 1: Large dataset detected
   - Found 50,000+ records in users table
   - Reset operation took 45 seconds
   - Consider using development data subset

⚠️  Warning 2: Missing indexes detected
   - Table 'orders' missing index on 'customer_id'
   - Table 'posts' missing index on 'published_at'
   - Performance may be impacted

Recommendations:
- Generate migrations to add missing indexes
- Implement data archiving strategy
- Consider using smaller development dataset

Operation successful despite warnings.
```

The command returns control to the main agent with database operation results and any recommended follow-up actions.