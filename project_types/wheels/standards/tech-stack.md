# Tech Stack

## Context

Global tech stack defaults for Agent OS projects, overridable in project-specific `.agent-os/product/tech-stack.md`.

- App Framework: Wheels 3.0+ (CFML MVC Framework)
- Language: CFML (ColdFusion Markup Language)
- CFML Engine: Lucee 6.2+ (preferred)
- Primary Database: PostgreSQL 17+
- ORM: Wheels ORM
- Database Migration: Wheels Migrator
- CLI Tool: CommandBox Wheels CLI
- JavaScript Framework: HTMX 2.0 / Alpine.js 4.0
- Package Manager: Commandbox/Forgebox.io (CFML modules)
- CSS Framework: TailwindCSS 4.0+
- UI Components: TailwindUI
- Font Provider: Google Fonts
- Font Loading: Self-hosted for performance
- Icons: FontAwesome Pro
- Testing Framework: TestBox (CFML) integrated into Wheels
- Testing Strategy: box wheels test 
- Application Hosting: Self hosted in Datacenter
- CFML Server: CommandBox with Undertow/Tomcat
- Hosting Region: Primary data center
- Database Hosting: Self hosted in Datacenter
- Database Backups: Daily automated
- Asset Storage: S3 flavor bucket (Wasabi) / Local file system (dev)
- CDN: Fastly
- CI/CD Platform: GitHub Actions
- CI/CD Trigger: Push to main/staging branches
- Tests: TestBox suite before deployment
- Production Environment: main branch
- Development Environment: Local CommandBox server

## CommandBox Server Management

Development server commands for Wheels applications:
- **Start server**: `box server start` - Starts development server using server.json configuration
- **Stop server**: `box server stop` - Stops the running development server
- **Restart server**: `box server restart` - Stops and restarts the development server
- **Check status**: `box server status` - Shows if server is running and on which port
- **Server logs**: `box server log` - View server logs
- **Open browser**: `box server open` - Opens application in default browser

Note: Commands can be run without `box` prefix when inside CommandBox shell.
