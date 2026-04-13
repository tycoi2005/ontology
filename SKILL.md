---
name: ontology
description: Typed knowledge graph for structured agent memory. Use when creating/querying entities (Person, Project, Task, Event, Document), linking related objects, or planning multi-step actions. Trigger on "remember", "what do I know about", "link X to Y", "show dependencies", or entity CRUD.
metadata: {"nanobot":{"emoji":"🕸️","requires":{"bins":["python3"]}}}
---

# Ontology

Typed entity graph with constraint validation for structured knowledge storage.

## Quick Reference

| Trigger | Command |
|---------|---------|
| Create entity | `python3 scripts/ontology.py create --type Person --props '{"name":"Alice"}'` |
| Get by ID | `python3 scripts/ontology.py get --id p_001` |
| List by type | `python3 scripts/ontology.py list --type Task` |
| Query by props | `python3 scripts/ontology.py query --type Task --where '{"status":"open"}'` |
| Link entities | `python3 scripts/ontology.py relate --from proj_001 --rel has_task --to task_001` |
| Get related | `python3 scripts/ontology.py related --id proj_001 --rel has_task` |
| Validate | `python3 scripts/ontology.py validate` |

## Storage

- Graph: `memory/ontology/graph.jsonl`
- Schema: `memory/ontology/schema.yaml`

Initialize:
```bash
mkdir -p memory/ontology
touch memory/ontology/graph.jsonl
```

## Core Types

```yaml
Person: { name, email?, phone?, notes? }
Organization: { name, type?, members[] }

Project: { name, status, goals[], owner? }
Task: { title, status, due?, priority?, assignee?, blockers[] }
Goal: { description, target_date?, metrics[] }

Event: { title, start, end?, location?, attendees[] }
Location: { name, address?, coordinates? }

Document: { title, path?, url?, summary? }
Message: { content, sender, recipients[], thread? }
Note: { content, tags[], refs[] }

Account: { service, username, credential_ref? }
Credential: { service, secret_ref }  # Never store secrets directly
```

## Relations

```yaml
has_owner: [Project, Task] -> [Person]
has_task: [Project] -> [Task]
blocks: [Task] -> [Task]  # acyclic: true
assigned_to: [Task] -> [Person]
attendee_of: [Person] -> [Event]
part_of: [Task, Document] -> [Project]
```

## Constraints

Define in `memory/ontology/schema.yaml`:

```yaml
types:
  Task:
    required: [title, status]
    status_enum: [open, in_progress, blocked, done]
  Event:
    required: [title, start]
  Credential:
    required: [service, secret_ref]
    forbidden_properties: [password, secret, token]

relations:
  blocks:
    from_types: [Task]
    to_types: [Task]
    acyclic: true
```

## Workflows

### Create and Link

```bash
python3 scripts/ontology.py create --type Person --props '{"name":"Alice"}'
# Output: {"id":"pers_abc123",...}

python3 scripts/ontology.py create --type Project --props '{"name":"Redesign","status":"active"}'

python3 scripts/ontology.py relate --from proj_001 --rel has_owner --to pers_abc123
```

### Query Related

```bash
# Outgoing: what does this project have?
python3 scripts/ontology.py related --id proj_001 --rel has_task

# Incoming: what is this task part of?
python3 scripts/ontology.py related --id task_001 --rel part_of --dir incoming

# Both directions
python3 scripts/ontology.py related --id p_001 --dir both
```

### Validate

```bash
python3 scripts/ontology.py validate
# Checks: required props, enums, forbidden props, relation types, cardinality, acyclicity
```

## JSONL Format

Append-only log at `memory/ontology/graph.jsonl`:

```jsonl
{"op":"create","entity":{"id":"p_001","type":"Person","properties":{"name":"Alice"},"created":"2026-01-15T10:00:00Z","updated":"2026-01-15T10:00:00Z"}}
{"op":"relate","from":"proj_001","rel":"has_owner","to":"p_001","timestamp":"2026-01-15T10:01:00Z"}
{"op":"update","id":"p_001","properties":{"email":"alice@example.com"},"timestamp":"2026-01-15T11:00:00Z"}
{"op":"delete","id":"p_001","timestamp":"2026-01-15T12:00:00Z"}
```

## Planning with Ontology

Model multi-step plans as graph operations:

```
Plan: "Schedule meeting and create follow-up tasks"

1. CREATE Event {title: "Team Sync", start: "2026-01-20T10:00:00Z"}
2. RELATE Event -> attendee_of -> p_001, p_002
3. CREATE Task {title: "Prepare agenda", assignee: p_001}
4. RELATE Task -> part_of -> proj_001
5. CREATE Task {title: "Send notes", blockers: [task_above]}
```

Each step validates before execution. Rollback on constraint violation.

## References

- `references/schema.md` — Full type definitions and constraint patterns
- `references/queries.md` — Query patterns and traversal examples
