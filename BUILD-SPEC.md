# PB-ASSIST — Build Specification

This document provides the complete data schemas, template definitions, and implementation details needed to build PB-ASSIST. Read `GEMINI.md` first for the high-level architecture and CLI reference.

---

## 1. JSONL Schemas (Revision Log — Layer 1)

All JSONL files live in `<playbook>/revisions/`. Every line is one JSON object. Files are append-only.

### revisions.jsonl

Records every content change to any section.

```json
{
  "id": "REV-001",
  "timestamp": "2025-02-09T14:30:00Z",
  "actor": "C. Kraft",
  "section": "containment",
  "action": "create",
  "content_hash": "sha256:abc123...",
  "message": "Initial containment procedures based on NIST 800-61r3 §3.3",
  "source": "manual"
}
```

Fields:
- `id`: Sequential revision ID (REV-001, REV-002, ...)
- `action`: One of `create`, `update`, `delete`
- `content_hash`: SHA-256 of the section content at this revision
- `source`: One of `manual`, `ai-accepted`, `template-init`

### references.jsonl

Records every reference citation linked to a section.

```json
{
  "id": "REF-001",
  "timestamp": "2025-02-09T14:32:00Z",
  "actor": "C. Kraft",
  "source_id": "nist-sp-800-61r3",
  "source_section": "3.3",
  "source_title": "NIST SP 800-61r3 — Containment Strategy",
  "target_section": "containment",
  "action": "add",
  "note": "Primary framework for containment procedures in financial services"
}
```

Fields:
- `action`: One of `add`, `remove`
- `source_id`: Matches directory name in `reference/`
- `source_section`: Specific section/paragraph in the source document

### approvals.jsonl

Records section-level sign-offs.

```json
{
  "id": "APR-001",
  "timestamp": "2025-02-09T16:00:00Z",
  "actor": "C. Kraft",
  "section": "containment",
  "action": "approve",
  "revision_id": "REV-012",
  "note": "Approved after legal review"
}
```

Fields:
- `action`: One of `approve`, `revoke`
- `revision_id`: The specific revision being approved

### versions.jsonl

Records version milestones for the playbook as a whole.

```json
{
  "id": "VER-001",
  "timestamp": "2025-02-09T17:00:00Z",
  "actor": "C. Kraft",
  "version": "1.0",
  "version_type": "major",
  "message": "Initial release — all sections approved",
  "revision_count": 47,
  "sections_approved": 14,
  "sections_total": 14
}
```

### ai-suggestions.jsonl

Records AI drafts submitted for review (accepted or rejected).

```json
{
  "id": "AIS-001",
  "timestamp": "2025-02-09T15:00:00Z",
  "actor": "C. Kraft",
  "section": "containment",
  "draft_file": "from-ai/draft-containment-20250209.md",
  "action": "accept",
  "references_cited": ["nist-sp-800-61r3:3.3", "mandiant-ir:containment-steps"],
  "note": "Accepted with minor edits to step 3"
}
```

Fields:
- `action`: One of `accept`, `reject`
- If `reject`, include `"reason": "..."`

---

## 2. YAML Schemas (Content — Layer 2)

All YAML files live in `<playbook>/content/`. These are derived by replaying the revision log.

### playbook.yaml

```yaml
name: ransomware-ir
title: "Ransomware Incident Response Playbook"
template: standard-ir
version: "1.2"
status: in-progress          # draft | in-progress | review | approved
created: "2025-01-02T10:00:00Z"
last_updated: "2025-02-09T14:30:00Z"
author: "C. Kraft"
revision_count: 47
sections_total: 14
sections_approved: 6
sections_draft: 6
sections_empty: 2
```

### sections/<nn>-<name>.yaml

```yaml
section_id: containment
title: "Containment"
order: 6
status: draft                # empty | draft | review | approved
version: "0.3"
author: "C. Kraft"
last_updated: "2025-02-09T14:30:00Z"
revision_id: "REV-045"      # Last revision that touched this section
references:
  - source_id: nist-sp-800-61r3
    source_section: "3.3"
    note: "Primary containment framework"
  - source_id: mandiant-ir
    source_section: "containment-steps"
    note: "Tactical containment procedures"
content: |
  ## Containment

  ### Immediate Containment Actions

  Upon confirmation of ransomware execution, the following containment
  actions should be initiated within the first 30 minutes...

  [full section content in markdown]
```

### ref-index.yaml

```yaml
references:
  - source_id: nist-sp-800-61r3
    sections_linked:
      - target: containment
        source_section: "3.3"
      - target: eradication
        source_section: "3.4"
    times_cited: 4
  - source_id: mandiant-ir
    sections_linked:
      - target: containment
        source_section: "containment-steps"
    times_cited: 2
```

### integrity.yaml

```yaml
generated: "2025-02-09T17:00:00Z"
hashes:
  revisions.jsonl: "sha256:a1b2c3..."
  references.jsonl: "sha256:d4e5f6..."
  approvals.jsonl: "sha256:789abc..."
  versions.jsonl: "sha256:def012..."
  ai-suggestions.jsonl: "sha256:345678..."
```

---

## 3. Reference Library Schema

Each reference source is a directory in `reference/`.

### metadata.yaml

```yaml
source_id: nist-sp-800-61r3
title: "Computer Security Incident Handling Guide"
author: "NIST"
version: "Revision 3"
publication_date: "2024-04-01"
url: "https://csrc.nist.gov/publications/detail/sp/800-61/rev-3/final"
date_retrieved: "2025-01-15"
description: "Primary NIST framework for incident response procedures"
excerpts:
  - filename: "section-3.3-containment.md"
    section: "3.3"
    title: "Containment Strategy"
  - filename: "section-3.4-eradication.md"
    section: "3.4"
    title: "Evidence Gathering and Eradication"
```

### excerpts/<filename>.md

```markdown
# NIST SP 800-61r3 — §3.3 Containment Strategy

> Source: NIST SP 800-61 Revision 3, Section 3.3
> Retrieved: 2025-01-15

Containment is the process of limiting the scope and magnitude of an
incident. The organization should define acceptable risks in
determining appropriate containment strategies...

[key excerpts, not full reproduction — respect copyright]
```

---

## 4. Template Schema

### templates/standard-ir.yaml

The standard IR playbook template. When `pb init` runs with `--template standard-ir`, it creates section stubs for each entry.

```yaml
template_id: standard-ir
title: "Standard Incident Response Playbook"
description: "14-section IR playbook following NIST 800-61r3 structure"
version: "1.0"

sections:
  - id: purpose
    title: "Purpose"
    order: 1
    required: true
    description: "Why this playbook exists and what it covers"

  - id: scope
    title: "Scope"
    order: 2
    required: true
    description: "What threats, systems, and environments are in scope"

  - id: roles
    title: "Roles and Responsibilities"
    order: 3
    required: true
    description: "Who does what during an incident"

  - id: definitions
    title: "Definitions and Severity Levels"
    order: 4
    required: true
    description: "Key terms, incident severity classification"

  - id: preparation
    title: "Preparation"
    order: 5
    required: true
    description: "Pre-incident readiness activities"

  - id: detection
    title: "Detection and Analysis"
    order: 6
    required: true
    description: "How the threat is identified and confirmed"

  - id: containment
    title: "Containment"
    order: 7
    required: true
    description: "Immediate and long-term containment actions"

  - id: eradication
    title: "Eradication"
    order: 8
    required: true
    description: "Removing the threat from the environment"

  - id: recovery
    title: "Recovery"
    order: 9
    required: true
    description: "Restoring systems and validating normal operations"

  - id: post-incident
    title: "Post-Incident Activity"
    order: 10
    required: true
    description: "Lessons learned, report writing, process improvement"

  - id: communication
    title: "Communication Plan"
    order: 11
    required: true
    description: "Internal and external communication procedures and templates"

  - id: escalation
    title: "Escalation Procedures"
    order: 12
    required: true
    description: "When and how to escalate, including regulatory notification"

  - id: references
    title: "References"
    order: 13
    required: true
    description: "All cited industry standards and sources (auto-generated from ref-index)"

  - id: appendices
    title: "Appendices"
    order: 14
    required: false
    description: "Checklists, contact lists, network diagrams, etc."

appendices_structure:
  - id: checklist
    title: "Appendix A: Incident Response Checklist"
  - id: contacts
    title: "Appendix B: Key Contacts and Escalation Matrix"
  - id: network
    title: "Appendix C: Network Architecture Reference"
  - id: evidence
    title: "Appendix D: Evidence Collection Procedures"
  - id: regulatory
    title: "Appendix E: Regulatory Notification Requirements"
```

### templates/headings.yaml

```yaml
heading_hierarchy:
  h1: "Section Title"           # e.g., "Containment"
  h2: "Subsection"              # e.g., "Immediate Containment Actions"
  h3: "Procedure Step"          # e.g., "Network Isolation"
  h4: "Detail"                  # Rarely used

numbering: true                 # Auto-number sections (1, 1.1, 1.1.1)
toc_depth: 3                    # TOC includes h1, h2, h3
```

### templates/docx-style.yaml

```yaml
page:
  size: letter
  margins:
    top: 1.0
    bottom: 1.0
    left: 1.25
    right: 1.25

fonts:
  heading: "Calibri"
  body: "Calibri"
  code: "Consolas"

sizes:
  h1: 18
  h2: 14
  h3: 12
  body: 11
  footer: 9

spacing:
  after_heading: 6
  after_paragraph: 6
  line_spacing: 1.15

header:
  enabled: true
  text: "{playbook_title}"
  alignment: right

footer:
  enabled: true
  text: "{playbook_title} — v{version} — {classification}"
  page_numbers: true
  alignment: center

front_matter:
  - title_page: true
  - version_table: true          # Auto-populated from versions.jsonl
  - table_of_contents: true
  - revision_history: true       # Full revision history table

classification: "INTERNAL USE ONLY"
```

### templates/appendix-structure.yaml

```yaml
appendix_format:
  numbering: letter              # Appendix A, B, C...
  page_break_before: true
  include_in_toc: true

default_appendices:
  - id: checklist
    title: "Incident Response Checklist"
    format: checklist             # Renders as checkbox list
  - id: contacts
    title: "Key Contacts and Escalation Matrix"
    format: table
  - id: network
    title: "Network Architecture Reference"
    format: freeform
  - id: evidence
    title: "Evidence Collection Procedures"
    format: numbered-steps
  - id: regulatory
    title: "Regulatory Notification Requirements"
    format: table
```

---

## 5. Registry Schema

### registry.yaml (workspace root)

```yaml
generated: "2025-02-09T17:00:00Z"
workspace_path: "/home/user/pb-workspace"
playbook_count: 3

playbooks:
  - name: ransomware-ir
    path: "playbooks/ransomware-ir"
    title: "Ransomware Incident Response Playbook"
    template: standard-ir
    version: "1.2"
    status: in-progress
    last_updated: "2025-02-09T14:30:00Z"
    revision_count: 47
    sections_approved: 6
    sections_total: 14

  - name: cloud-ir
    path: "playbooks/cloud-ir"
    title: "Cloud Incident Response Playbook"
    template: standard-ir
    version: "0.4"
    status: in-progress
    last_updated: "2025-02-01T10:00:00Z"
    revision_count: 23
    sections_approved: 3
    sections_total: 14

  - name: it-infiltrator
    path: "playbooks/it-infiltrator"
    title: "IT Infiltrator / Insider Threat Playbook"
    template: standard-ir
    version: "1.0"
    status: approved
    last_updated: "2024-12-20T16:00:00Z"
    revision_count: 61
    sections_approved: 14
    sections_total: 14
```

---

## 6. Word Document Output Structure

The rendered .docx should follow this structure exactly:

```
1.  Title Page
    - Playbook title
    - Version number
    - Classification
    - Date
    - Author(s)

2.  Version History Table
    | Version | Date       | Author    | Description              |
    |---------|------------|-----------|--------------------------|
    | 1.0     | 2025-01-15 | C. Kraft  | Initial release          |
    | 1.1     | 2025-02-01 | C. Kraft  | Updated containment      |
    (auto-populated from versions.jsonl)

3.  Table of Contents
    (auto-generated, depth 3)

4.  Sections 1-12
    (from template, each with consistent heading hierarchy)

5.  Section 13: References
    (auto-generated from ref-index.yaml)
    Format: "NIST SP 800-61r3, §3.3 — Containment Strategy. Retrieved 2025-01-15."

6.  Section 14: Appendices
    (from appendix-structure.yaml)
```

### python-docx Implementation Notes

- Use `Document()` from `python-docx`
- Define custom styles matching `docx-style.yaml`
- Use `add_heading()` with levels 1-4
- Use `add_paragraph()` for body text
- Use `add_table()` for version history and contact tables
- Use `add_page_break()` before each major section and appendix
- Parse section content (markdown) and convert to docx elements
- For TOC: insert a TOC field code (`\o "1-3" \h \z \u`) — this creates a placeholder that Word will populate when the document is opened and the user presses "Update Fields"

---

## 7. AI Exchange File Formats

### to-ai/draft-request-<section>-<timestamp>.md

```markdown
# Draft Request: Containment

## Playbook Context
- **Playbook:** Ransomware Incident Response Playbook
- **Template:** standard-ir
- **Section:** containment (Section 7 of 14)
- **Status:** empty (first draft requested)

## Section Description
Immediate and long-term containment actions for ransomware incidents.

## Additional Context
Financial services environment, PCI DSS scope, 24/7 SOC coverage.

## Linked References
The following reference excerpts are included for your use:

### NIST SP 800-61r3, §3.3 — Containment Strategy
[excerpt content from reference/nist-sp-800-61r3/excerpts/section-3.3-containment.md]

### Mandiant IR Guide — Containment Steps
[excerpt content from reference/mandiant-ir/excerpts/containment-steps.md]

## Instructions
Draft the containment section following the heading hierarchy defined in the template.
Cite all references used. Use markdown format. Do not include content from sections
other than containment. The author will review and edit your draft before committing.
```

### from-ai/draft-<section>-<timestamp>.md

The AI's response — a markdown draft of the section with inline citations. The author reviews this file, edits it, and then uses `pb ai accept` + `pb commit` to incorporate it.

---

## 8. Key Functions to Implement

### revision.py
- `append_revision(playbook_path, section, action, content, actor, message, source)` — append to revisions.jsonl
- `append_reference(playbook_path, source_id, source_section, target_section, actor, action, note)` — append to references.jsonl
- `append_approval(playbook_path, section, actor, revision_id, action, note)` — append to approvals.jsonl
- `append_version(playbook_path, version, version_type, actor, message)` — append to versions.jsonl
- `append_ai_suggestion(playbook_path, section, draft_file, actor, action, references_cited, note)` — append to ai-suggestions.jsonl
- `replay_revisions(playbook_path)` → returns current state dict by replaying all JSONL files

### playbook.py
- `init_playbook(workspace_path, name, template)` — create directory structure, section stubs from template
- `open_playbook(playbook_path)` — replay revision log, rebuild content/ files, return playbook state
- `get_playbook_info(playbook_path)` → playbook.yaml data
- `rebuild_content(playbook_path)` — delete content/, regenerate from revisions/

### section.py
- `get_section(playbook_path, section_id)` → section content and metadata
- `update_section(playbook_path, section_id, content, actor, message)` — writes content file + appends revision
- `get_section_status(playbook_path, section_id)` → status, version, ref count
- `list_sections(playbook_path)` → all sections with status

### reference.py
- `list_references(workspace_path)` → all available reference sources
- `add_reference_link(playbook_path, source_id, source_section, target_section, actor, note)` — link ref to section
- `search_excerpts(workspace_path, query)` → matching excerpts with source info
- `get_reference_detail(workspace_path, source_id)` → metadata + excerpt list

### render.py
- `render_docx(playbook_path, template_path, output_path)` — generate complete .docx
- `render_markdown(playbook_path, output_path)` — generate markdown version for review
- `build_version_table(playbook_path)` → version history data for the front matter
- `build_reference_list(playbook_path)` → formatted reference list for the references section

### registry.py
- `scan_workspace(workspace_path)` → rebuild registry.yaml by scanning playbooks/
- `list_playbooks(workspace_path)` → registry data
- `get_active_playbook(workspace_path)` → currently opened playbook (stored in .pb-active file)
- `set_active_playbook(workspace_path, playbook_name)` — write .pb-active

### integrity.py
- `compute_hashes(playbook_path)` → dict of filename → SHA-256 hash
- `verify_integrity(playbook_path)` → compare current hashes against stored hashes, flag changes
- `store_hashes(playbook_path)` — write integrity.yaml

### ai_exchange.py
- `generate_draft_request(playbook_path, section_id, workspace_path, context)` — build the to-ai/ markdown file
- `list_pending_drafts(playbook_path)` → drafts in from-ai/ not yet accepted/rejected
- `accept_draft(playbook_path, draft_file, section_id, actor, note)` — copy draft content to section, log acceptance
- `reject_draft(playbook_path, draft_file, actor, reason)` — log rejection
