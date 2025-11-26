# AGENTS.md - Migration from CUBA to Jmix

This is a guide for an AI agent to iteratively migrate a project from CUBA Platform v.7 to Jmix v.2.

## Project Structure

```
projectWorkspace/
│
├── migration-rules/                                # Wave-specific transformation rules
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
│    └── 110 UI UX Rules.md
│
├── source-projects/                                 # Source CUBA project
├── target-projects/                                 # Target Jmix project
└── AGENTS.md                                        # This document
```

## Wave Strategy

Migration proceeds incrementally in the following waves:

1. **Entities** - domain model classes and annotations
2. **Fetch Plans** - data loading configurations
3. **Business Logic** - services, beans, entity and transaction listeners
4. **Fragments** - reusable UI elements
5. **Screens** - UI layer

Do one wave at a time. You will be explicitly asked to proceed with a particular wave.

Do not commit any changes automatically.

Read **010 Common.md** when starting each migration wave.
When migrating entities, read **020 Entities.md**.
When migrating fetch plans, read **030 Fetch Plans.md**.
When migrating business logic, read **040 Business Logic.md**.
When migrating fragments and screens, read all documents from **050 UI Fragments.md** to **110 UI UX Rules.md**.
