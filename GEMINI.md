# PB-ASSIST — Project Instructions

## What This Is

PB-ASSIST is an AI-augmented playbook development system for building consistent, auditable, regulation-ready incident response playbooks. It is architected identically to IR-ASSIST (an incident response coordination tool) — append-only revision logs, derived state, generated outputs, and human-gated AI advisory.

**Read `pb-assist-architecture-diagram.html` first.** It is the authoritative design document. Everything in this file derives from that architecture.

## Core Philosophy

- **The revision log is the source of truth.** Everything else is derived.
- **Every action is a deliberate CLI command.** No hidden state, no GUI.
- **AI advises; humans decide.** All AI output passes through human review before entering the record.
- **Each playbook is a self-contained directory.** Portable, no database, no server.
- **The registry is derived.** Scan directories → regenerate. Never authoritative.
- **Templates govern structure.** Define once, every playbook inherits.

## Tech Stack

- **Language:** Python 3.11+
- **CLI Framework:** `click` (preferred) or `typer`
- **YAML:** `ruamel.yaml` (preserves comments and formatting)
- **JSONL:** Standard `json` module, one object per line, append-only writes
- **Word Output:** `python-docx`
- **Hashing:** `hashlib` (SHA-256 for integrity verification)
- **No database.** Files are the database.
- **No web server.** CLI only.
- **No external API calls in core.** AI integration is file-based (air gap).

## Directory Structure

```
pb-workspace/                          # Workspace root
├── GEMINI.md                          # This file
├── BUILD-SPEC.md                      # Detailed build specification
├── pb-assist-architecture-diagram.html # Architecture reference
├── registry.yaml                      # Derived index of all playbooks
│
├── reference/                         # Shared reference library
│   ├── nist-sp-800-61r3/
│   │   ├── metadata.yaml              # Title, version, URL, date retrieved
│   │   └── excerpts/                  # Key sections as markdown files
│   ├── mandiant-ir/
│   │   ├── metadata.yaml
│   │   └── excerpts/
│   ├── sans-ih/
│   │   ├── metadata.yaml
│   │   └── excerpts/
│   ├── ians/
│   │   ├── metadata.yaml
│   │   └── excerpts/
│   └── org-doctrine/                  # Internal policies
│       ├── metadata.yaml
│       └── excerpts/
│
├── templates/                         # Shared template definitions
│   ├── standard-ir.yaml               # Standard IR playbook template
│   ├── headings.yaml                  # Required heading structure
│   ├── docx-style.yaml                # Font, spacing, TOC, page layout
│   └── appendix-structure.yaml        # Appendix ordering & format
│
├── pb_assist/                         # Python package
│   ├── __init__.py
│   ├── cli.py                         # Click CLI entry points
│   ├── playbook.py                    # Playbook management (init, open, list)
│   ├── section.py                     # Section content management
│   ├── revision.py                    # Append-only revision log engine
│   ├── reference.py                   # Reference library management
│   ├── render.py                      # .docx rendering engine
│   ├── registry.py                    # Workspace registry (derived)
│   ├── integrity.py                   # SHA-256 hashing & verification
│   ├── schemas.py                     # Data models / schemas
│   └── ai_exchange.py                 # AI drafting request/response handler
│
├── tests/                             # Test suite
│   ├── test_revision.py
│   ├── test_playbook.py
│   ├── test_section.py
│   ├── test_reference.py
│   ├── test_render.py
│   └── test_registry.py
│
└── playbooks/                         # Where individual playbooks live
    └── ransomware-ir/                 # Example: one self-contained playbook
        ├── revisions/
        │   ├── revisions.jsonl
        │   ├── references.jsonl
        │   ├── approvals.jsonl
        │   ├── versions.jsonl
        │   └── ai-suggestions.jsonl
        ├── content/
        │   ├── playbook.yaml
        │   ├── sections/
        │   │   ├── 01-purpose.yaml
        │   │   ├── 02-scope.yaml
        │   │   ├── 03-roles.yaml
        │   │   └── ...
        │   └── ref-index.yaml
        ├── output/
        │   ├── drafts/
        │   └── diagrams/
        └── ai-exchange/
            ├── to-ai/
            └── from-ai/
```

## CLI Commands — Complete Reference

All commands use the `pb` entry point.

### Workspace Commands
```bash
pb list                                    # List all playbooks with status, version, last updated
pb scan                                    # Rebuild registry.yaml by scanning playbooks/ directory
pb status                                  # Show section-level status of active playbook
```

### Playbook Lifecycle
```bash
pb init <name> --template <template>       # Create new playbook from template
pb open <name>                             # Open existing playbook, rebuild state from revision log
pb info                                    # Show metadata for active playbook
pb version <major|minor|patch> --msg "..."  # Create version milestone
pb approve <section> --approver "Name"     # Sign off on a section
```

### Content Authoring
```bash
pb draft <section> [--use-refs] [--context "..."]  # Generate AI drafting request (writes to ai-exchange/to-ai/)
pb commit <section> --msg "..."                     # Commit content to revision log
pb edit <section>                                    # Open section in $EDITOR for manual editing
pb diff <section>                                    # Show changes since last commit
```

### Reference Management
```bash
pb ref list                                         # List all references in the shared library
pb ref add <source-id> --section <num> --target <playbook-section>  # Link reference to section
pb ref search <query>                               # Search reference excerpts
pb ref show <source-id>                             # Show metadata and available excerpts
```

### Output
```bash
pb render --format docx                    # Render current playbook to Word document
pb render --format md                      # Render to markdown (for review)
pb integrity                               # Verify SHA-256 hashes of all revision log files
```

### AI Exchange
```bash
pb ai submit                               # Submit pending draft request to ai-exchange/to-ai/
pb ai review                               # List AI drafts waiting for review in from-ai/
pb ai accept <draft-file> --section <name> # Accept AI draft into section (still requires pb commit)
pb ai reject <draft-file> --reason "..."   # Reject AI draft with reason logged
```

## Critical Implementation Rules

1. **Append-only writes for revision logs.** Always `open(path, 'a')`. Never overwrite, truncate, or delete JSONL files. Status changes produce new entries.

2. **State is always derived.** `content/` files are built by `rebuild_content()` which replays `revisions/` from scratch. Delete content/, rebuild — identical result.

3. **Registry is always derived.** `registry.yaml` is built by scanning `playbooks/` directories. Delete it, run `pb scan` — identical result.

4. **No hidden state.** The CLI holds nothing important in memory. Everything lives in files. Restart Python, re-run any command — same result.

5. **Human-gated AI.** The `ai-exchange/` directory is the only interface between AI and the system. AI output in `from-ai/` must be explicitly accepted via `pb ai accept` and then committed via `pb commit` before it enters the revision log.

6. **SHA-256 integrity.** Every JSONL file gets a hash computed on `pb render` and `pb integrity`. Store hashes in `content/integrity.yaml`. If a hash changes between runs, flag it.

7. **Template enforcement.** When `pb init` creates a playbook, it generates section stubs from the template. The render engine validates that all required sections exist and are in the correct order.

8. **Timestamps are ISO 8601 UTC.** Every JSONL entry includes `"timestamp": "2025-02-09T14:30:00Z"`.

9. **Every JSONL entry includes an actor field.** `"actor": "C. Kraft"` or `"actor": "ai-draft"` — always attributable.

## File Schemas

See `BUILD-SPEC.md` for complete JSONL and YAML schemas.

## Build Order

Build in this order. Each phase should be testable before moving to the next.

1. **Revision engine** (`revision.py`) — append-only JSONL writes, replay, integrity hashing
2. **Playbook management** (`playbook.py`) — init from template, open/rebuild, metadata
3. **Section management** (`section.py`) — content CRUD via revision log, status tracking
4. **Reference management** (`reference.py`) — library CRUD, linking refs to sections, search
5. **Registry** (`registry.py`) — scan workspace, build/rebuild registry.yaml
6. **CLI layer** (`cli.py`) — Click commands wrapping all the above
7. **Render engine** (`render.py`) — .docx generation from content + template using python-docx
8. **AI exchange** (`ai_exchange.py`) — draft request generation, acceptance/rejection workflow
9. **Tests** — unit tests for each module

## Testing

- Use `pytest`
- Each test creates a temporary playbook directory, runs operations, verifies file contents
- Key test: create playbook → make changes → delete content/ → rebuild → verify identical output
- Key test: verify all JSONL files are append-only (line count never decreases)
- Key test: integrity hashes match after rebuild
