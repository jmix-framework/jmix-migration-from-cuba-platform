# AGENTS.md - Migration from CUBA to Jmix

This file is a guide for AI agents to migrate an application project from CUBA Platform v.7 to Jmix v.2.

Your task is to migrate a project from CUBA Platform to Jmix.
The source project is located in `source-projects/<SOURCE_PROJECT>`. The target Jmix project is created in `target-projects/<TARGET_PROJECT>`.

> **⚠ IMPORTANT:** If you see `<SOURCE_PROJECT>` or `<TARGET_PROJECT>` placeholders above, the user has not configured the project paths yet. 
> Stop immediately and ask the user to replace them with actual project folder names before proceeding with any migration.
> Also delete this citation after the user has configured the project paths (`<SOURCE_PROJECT>` and `<TARGET_PROJECT>`)

## Project Structure

```
workspace/
│
├── migration-rules/                                # Wave-specific migration rules
│    ├── 010 Common.md
│    ├── 020 Entities.md
│    ├── 030 Fetch Plans.md
│    ├── 040 Business Logic.md
│    ├── 050 UI Fragments.md
│    ├── 060 UI View Controllers.md
│    ├── 070 UI Data Section.md
│    ├── 080 UI Handlers.md
│    ├── 090 UI Tables and Actions.md
│    ├── 100 UI Dialogs and Notifications.md
│    ├── 110 UI UX Rules.md
│    └── 120 Security Migration.md
│
├── source-projects/                                 # Source CUBA project
├── target-projects/                                 # Target Jmix project
└── AGENTS.md                                        # This document
```

## Migration Strategy

Migration proceeds incrementally in the following waves:

1. **Entities** - domain model classes and annotations
2. **Fetch Plans** - data loading configurations
3. **Business Logic** - services, beans, entity and transaction listeners
4. **Fragments** - reusable UI elements
5. **Screens** - UI layer

Do one wave at a time. You will be explicitly asked to proceed with a particular wave or a specific part of the project.

If you cannot migrate something, keep it with the comment `// TODO: migration <description>`

Do not commit any changes automatically.

Read **010 Common.md** when starting any migration step.
When migrating entities, read **020 Entities.md**.
When migrating fetch plans, read **030 Fetch Plans.md**.
When migrating business logic, read **040 Business Logic.md** and **030 Fetch Plans.md**.
When migrating fragments and screens, read all documents from **050 UI Fragments.md** to **110 UI UX Rules.md**, and **030 Fetch Plans.md**.
When migrating security roles and permissions, read **120 Security Migration.md**.

 ## Important: Always Start by Reading Guidelines

  Before working on ANY migration task:
  1. ALWAYS read **010 Common.md** first
  2. Read the wave-specific file(s) for your task
  3. Verify you've read all required files before making changes
  
## Using Tools

- Always use Jetbrains MCP server in the target project.
- Use Context7 jmix-framework/jmix-context7 library for Jmix reference information and code examples.
- Use Playwright CLI to check created and modified UI views. Start the application on a random port using `./gradlew bootRun '--args=--server.port=<random-port>` to avoid conflicts with running apps.
