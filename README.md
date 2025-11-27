# Jmix AI-Assisted Migration

This repository is designed to facilitate migration of application projects from CUBA Platform to Jmix with the help of AI agents.

Below we assume you are using Claude Code, but other agents should work as well.

## Migration process

### Setup

- Create `workspace/source-projects` folder and copy or clone the source CUBA project into it, for example `source-projects/myapp`.
- Create project `workspace/target-projects/myapp-jmix` using Jmix Studio, with the same base package as in source project.
- Init git repo and commit all.
- Setup IntelliJ IDEA MCP Server
	- Settings -> Tools -> MCP Server
		- Enable
		- Claude Code -> Auto-Configure
- Open terminal in `workspace` and run Claude Code
- `/mcp`, check that `jetbrains` is connected
- Open the single `workspace/target-projects/myapp-jmix` project in IntelliJ.

### Sequence

We recommend to proceed with migration in the following sequence: entities, fetch plans, business logic, fragments, screens. If the project is large, split each phase to the smaller steps, e.g. by packages.

### Example prompts

**Init agent:**

```
Your task is to migrate a project from CUBA Platform to Jmix.
The source project is located in the `source-projects/myapp`. The empty target Jmix project is created in `target-projects/myapp-jmix`.
Read `AGENTS.md` to understand the project structure, migration approach and rules.
```

**Migrate entities:**

```
Read AGENTS.md and migrate all entities. If you cannot migrate something, keep it with the comment `// TODO: migration <description>`
```

**Migrate fetch plans:**

```
Read AGENTS.md and migrate all shared fetch plans.
```

**Migrate business logic:**

```
Read AGENTS.md and migrate all business logic.
```

**Migrate UI:**

```
Read AGENTS.md and migrate fragments if any.
```

```
Read AGENTS.md and Migrate all screens in `com.company.myapp.gui.sample` package.
```

```
Read AGENTS.md and migrate all screens.
```