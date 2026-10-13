# MRADiK - Multi Repo Agentic Development (intelligent) Kit

A Claude Code plugin that lets one repository ask another a question. It runs an MCP server; when Claude Code in project A calls `ask_project`, MRADiK starts a read-only Claude agent inside project B's folder, and that agent answers from B's files and cites them.

Use it when work in one repository depends on another: types and schemas, events, endpoints, conventions, how something is implemented.

## Install

Requirements: Claude Code, and Node.js 20 or newer on `PATH`.

```bash
claude plugin marketplace add madefast-ai/claude-plugins
claude plugin install mradik@madefast
```

In Claude Desktop: **Customize → Plugins → Add marketplace → Add from a repository**, enter `madefast-ai/claude-plugins`, then install **mradik**.

Then restart Claude Code. The server runs from the bundled `dist/mradik.js`; nothing to build or install.

Updating: `claude plugin marketplace update madefast`, then `claude plugin update mradik@madefast`, then restart.

### Finding the claude CLI

Each query spawns `claude`. MRADiK uses `claude` from `PATH`; if Claude Code was not started from a shell that has it (for example with nvm), set `CLAUDE_PATH` to its full path (`which claude`).

### Where state lives

`~/.mradik/` (or `$MRADIK_HOME`), so it survives plugin updates:

```
~/.mradik/
├── projects.json            namespaces and registered projects
└── cache/
    ├── queries.json         answer cache (10-minute TTL)
    └── conversations.json   conversation history (10-minute sliding window)
```

`projects.json` looks like this, and can be edited by hand:

```json
{
  "version": "1.1.0",
  "namespaces": {
    "default": {
      "name": "default",
      "description": "Projects not in any other namespace",
      "createdAt": "2026-10-12T14:00:00.000Z"
    },
    "shop": {
      "name": "shop",
      "description": "Storefront and its services",
      "createdAt": "2026-10-12T14:04:00.000Z"
    }
  },
  "projects": {
    "billing-service": {
      "name": "billing-service",
      "path": "/Users/me/work/billing-service",
      "description": "Invoicing, payments and subscription billing",
      "domains": ["billing", "invoices", "payments"],
      "aliases": ["billing"],
      "namespace": "shop",
      "status": "ready",
      "registeredAt": "2026-10-12T14:05:12.481Z"
    }
  }
}
```

Set `status` to `"disabled"` to stop routing questions to a project without removing it.

## Namespaces

A namespace groups projects, for example the repositories of one product. Every project is in exactly one namespace. A project added without one goes to `default`, which always exists and can't be renamed or deleted. Deleting a namespace moves its projects to `default`; nothing is unregistered. Namespaces only group and filter: `ask_project` can reach any registered project.

## Tools

### Project registry

| Tool | Description |
|------|-------------|
| `list_projects` | List registered projects, optionally filtered by `status` and `namespace` |
| `add_project` | Register a local repository: `name`, absolute `path`, `description`, optional `domains`, `aliases` and `namespace` (default `default`; it must exist) |
| `remove_project` | Unregister a project (no files are touched) |
| `move_project` | Move a project to another namespace |
| `get_project_info` | Details of one project, by `name` (or alias) or by `path`. Pass your working directory as `path` to check whether the current project is registered, and under which name |

### Namespaces

| Tool | Description |
|------|-------------|
| `list_namespaces` | List namespaces and the projects in each |
| `create_namespace` | Create a namespace: `name`, optional `description` |
| `update_namespace` | Rename a namespace (`new_name`) and/or change its `description` |
| `delete_namespace` | Delete a namespace; its projects move to `default` |

### Cross-project queries

| Tool | Description |
|------|-------------|
| `ask_project` | Ask a project a question. Params: `project` (name or alias), `query`, optional `context`, and `caller_path` (your working directory) or `from_project` (your project name) to keep your conversation separate from other callers |

### Conversations

| Tool | Description |
|------|-------------|
| `get_conversation_stats` | Active conversations and their history size |
| `clear_conversation` | Forget the history of one conversation |

## How a query works

```
┌─────────────────┐                              ┌─────────────────┐
│   Project A     │                              │   Project B     │
│  Claude Code    │                              │ answerer agent  │
└────────┬────────┘                              └────────▲────────┘
         │                                                │
         │ ask_project("project-b",                       │
         │   query: "Which events do you publish?",       │
         │   caller_path: "/path/to/project-a")           │
         ▼                                                │
┌─────────────────────────────────────┐                   │
│           MRADiK MCP Server         │                   │
│                                     │                   │
│  1. Look up project-b in registry   │                   │
│  2. Check cache (10-min TTL)        │                   │
│  3. Get conversation history        │                   │
│  4. Build prompt with history       │                   │
│  5. Spawn claude -p in project-b ──────────────────────►│
│  6. Cache response                  │                   │
│  7. Store in conversation history   │                   │
└─────────────────────────────────────┘
```

- The child `claude -p` gets only `Read`, `Grep` and `Glob`, and loads no MCP servers.
- Conversation channel key: `"project-a->project-b"` (or `"project-b"` when the caller is unknown). Earlier questions and answers are prepended to the prompt, so follow-ups work. The window is 10 minutes and resets on each query.

## The answerer agent

A registered project needs no setup. The answerer sees what the project already gives Claude (its `CLAUDE.md` and `.claude/rules`) and searches the code for the rest. A project with a good `CLAUDE.md` gets faster, better answers. MRADiK stores nothing about the projects themselves.

By default a built-in read-only answerer is used: it searches before answering, quotes exact definitions and cites `path:line`. To change how a project answers, give it its own agent: any `.md` file in `.claude/agents/` with `name: answerer`. MRADiK uses it instead of the built-in one. Keep its tools read-only:

```markdown
---
name: answerer
description: Answers questions from other projects about this codebase
tools: Read, Grep, Glob
---

You answer questions that other projects ask about this repository.
Start with docs/architecture.md; event schemas live in src/events/.
Cite files as path:line. If something is not in the code, say so.
```

## License

MIT
