```yaml
doc_id: "std-doc-guide-lw"
doc_type: "technical_standards"
title: "Documentation Authoring Guidelines - GameDev-Last-War (RAG-ready)"
owner: "team-unity"
tags:
  - "doc_authoring_principles"
  - "doc_mandatory_structure"
  - "doc_yaml_metadata"
  - "doc_tags_to_sections"
  - "doc_text_structure"
  - "doc_example"
  - "doc_precommit_checklist"
scope: "Rules for writing RAG-ready internal technical documents for GameDev-Last-War"
last_modified: "2026-01-22T09:34:05Z"
```

# Documentation Authoring Guidelines — GameDev-Last-War (RAG-ready)

## **1. General principles**

All internal technical documents in the **GameDev-Last-War** project must follow a unified format that ensures:
- **Machine readability** (for automated RAG processing, indexing, and search),
- **Human clarity** (for quick understanding),
- **Structural integrity** (so each section is self-contained and correctly mapped to metadata).

## **2. Mandatory document structure**

Each document **must start** with a **YAML** metadata block wrapped in triple backticks with the language specified:

> ⚠️ **Important**: this YAML block **must be the first content** in the document. Any text before it will be ignored during indexing.

After the YAML block, the document must include:
- Exactly one level-1 heading (`#`) — the document title.
- Main sections only as level-2 headings (`##`) in the format `## **N. ...**` (see section 5). These headings are used for automatic chunk splitting.

### **2.1. Copy-paste template (minimal template)**

Note: heading markers in the example below are prefixed with `> ` so this guide cannot accidentally break tag/section validation if a tool scans headings inside fenced code blocks. Remove the `> ` prefix on those lines when copy-pasting.

````markdown
```yaml
doc_id: "std-your-doc-id"
doc_type: "technical_standards"
title: "Human readable title"
owner: "team-or-user"
tags:
  - "tag_for_section_1"
  - "tag_for_section_2"
scope: "Short scope description"
last_modified: "YYYY-MM-DDTHH:MM:SSZ"
```

> # Document Title

> ## **1. Section title**
...

> ## **2. Section title**
...
````

## **3. YAML metadata authoring rules**

### **3.1. Required fields**

| Field | Type | Description |
|------|-----|----------|
| `doc_id` | string | Unique, type-prefixed ID (kebab-case; see validation rules in 3.3) |
| `doc_type` | string | Document type (see allowed values in 3.3) |
| `title` | string | Human-readable name |
| `owner` | string | Document owner (see format in 3.3) |
| `tags` | array of strings | **Key field!** List of tags per section (see section 4) |
| `scope` | string | Short description of the document applicability scope |

### **3.2. Value format**

- All string values (e.g. `doc_id`, `doc_type`, `title`, `owner`, `scope`, `last_modified`) must be in **double quotes**.
- Arrays (`tags`) must use only `-` (YAML list).
- `doc_id` must:
  - Use **only**: latin letters `a-z`, digits `0-9`, plus `-` (kebab-case only; **no `_`**)
  - Be **type-prefixed**: `std-...`, `howto-...`, or `gdd-...` (see 3.3)
- Each element of `tags` must use **only**: latin letters `a-z`, digits `0-9`, plus `-` and `_`.
  - The `/` character is **not allowed** in `doc_id` and `tags` to avoid confusing identifiers with paths.
- `title` and `scope` may be “free text” (spaces and standard punctuation are allowed).

✅ Good:

```yaml
doc_id: "std-addressables"
tags:
  - "groups"
  - "tagging"
  - "naming"
```

❌ Bad:

```yaml
tags: ["addressable", "asset_management"]  # Invalid array syntax
```

### **3.3. Recommended “schema” (validation rules)**

- `doc_id`:
  - Regex: `^(std|howto|gdd)-[a-z0-9]+(?:-[a-z0-9]+)*$`
  - Recommendation: `doc_id` matches the filename without extension (especially in `Assets/Docs/RAG/`).
  - Prefix mapping:
    - `std-...` → `doc_type: "technical_standards"`
    - `howto-...` → `doc_type: "how_to"`
    - `gdd-...` → `doc_type: "gdd"`
- `doc_type`:
  - Allowed values (enum): `"technical_standards"`, `"how_to"`, `"gdd"`, `"architecture"`, `"runbook"`
  - If you only use standards, keep `"technical_standards"` and remove the other values from this list.
- `owner`:
  - Format: `"team-<name>"` or `"user-<name>"` (example: `"team-ui"`, `"user-ivanov"`).
  - One owner per document (if you need a list, add a separate `owners` field, but only if ingestion supports it).
- `tags`:
  - Regex for each tag: `^[a-z0-9]+(?:[-_][a-z0-9]+)*$`
  - Recommendation: 3–24 characters per tag.

## **4. Requirements for tags (`tags`) and their relationship to content**

> 🔗 **Tags directly participate in automatic indexing of document sections.**

### **4.1. One-to-one rule (tags to sections)**

- The number of elements in `tags` **must equal the number of main level-2 sections** (headings of the form `## **N. ...**`).
- The order of tags **must match the order of sections**.
- The document **must not** contain any other level-2 headings (`##`) besides `## **N. ...**`.
  - If you need appendices/links, format them as `###` inside the last section.

Example:

```yaml
tags:
  - "groups"
  - "tagging"
  - "naming"
  - "performance"
  - "checklist"
```

corresponds to:

```markdown
> ## **1. Addressable Group Creation...**       → tag: "groups"
> ## **2. Asset Tagging Rules**                → tag: "tagging"
> ## **3. Addressable Name Naming...**         → tag: "naming"
> ## **4. Performance-Optimized...**           → tag: "performance"
> ## **5. Addressable Management Checklist**   → tag: "checklist"
```

### **4.2. Semantic relationship**

Each tag **must reflect the essence of its section**:
- Not generic words (`rules`, `guide`, `info`).
- But **specific concepts**: `naming`, `memory`, `bundle_size`, `lifecycle`, `validation`.

Goal: searching for the `performance` tag should return one relevant section, not the entire document.

### **4.4. Sufficient tag specificity (recommendation)**

If tags are used globally (search/filter by tag across many documents), avoid overly “broad” tags like `groups`, `labels`, `naming` — they quickly become noisy.

Recommendation: **prefix ambiguous tags** with the document topic or subsystem while keeping the `^[a-z0-9]+(?:[-_][a-z0-9]+)*$` format:
- ✅ Good: `addressable_groups`, `ui_naming`, `network_errors`
- ⚠️ Acceptable (if tags are used only within one doc_id): `groups`, `labels`, `naming`

### **4.3. Consequences of mismatch**

If the number of tags ≠ the number of `## **N. ...**` headings, then:
- Chunk splitting will still happen, but **the “tag → chunk” mapping will be incorrect**.
- RAG search will start returning irrelevant fragments for the “correct” tags.

Recommendation: validate documents automatically (pre-commit/CI) using the simple rule “tag count == number of `## **N. ...**` headings”.

## **5. Requirements for the main text**

### **5.1. Headings**

- Use **exactly one** level-1 heading (`#`) after YAML — it is the document title.
- Use **only** level-2 headings (`##`) for main sections, and they must be formatted as:
  - `## **1. ...**`
  - `## **2. ...**`
  - …
- Numbering must be:
  - Sequential (no gaps),
  - Unique,
  - Increasing.

### **5.2. Configuration for automatic chunk splitting**

The document will be automatically split by headings of the form:

```markdown
> ## **1. ...
> ## **2. ...
...
> ## **N. ...
```

→ Each such heading becomes a **separate chunk** in the vector database.

### **5.3. Recommended fields for maintainability (optional)**

If your ingestion pipeline supports additional fields, it is recommended to add:
- `last_modified`: timestamp in `"YYYY-MM-DDTHH:MM:SSZ"` format (UTC),
- `version`: version string (e.g. `"1.0"`),
- `related_docs`: a list of related document `doc_id`s.

Recommendation: update `last_modified` automatically using script.

For local usage in unity project repo it is `python rag_update_last_modified.py`.
Bootstrap existing docs once with `python rag_update_last_modified.py --bootstrap`.

### **5.4. Links to other documents (doc_id links)**

To make links to other internal documents **machine-detectable** and independent of file paths/names:

- In the document text, use an **explicit marker** in the format `doc_id:<id>`, where `<id>` is the `doc_id` value of the target document.
- Do not use links to documents by filename (`*.md`) or by path (`Assets/...`) as the primary navigation method between documents.
- If you add a “Related documents” table or lines like “Detailed rules: …”, list documents via `doc_id:<id>` for stable indexing.

## **6. Example of a valid document (fragment)**

````markdown
```yaml
doc_id: "std-addressables"
doc_type: "technical_standards"
title: "Addressable Assets Management Rules - GameDev-Last-War"
owner: "team-unity"
tags:
  - "groups"
  - "labels"
  - "naming"
  - "performance"
  - "validation"
  - "blockers"
  - "monitoring"
scope: "Unity Addressables workflow, asset organization, memory safety"
last_modified: "YYYY-MM-DDTHH:MM:SSZ"
```

> # Addressable Assets Management Rules

> ## **1. Addressable Group Creation and Organization**
... text ...

> ## **2. Asset Tagging Rules**
... text ...
````

## **7. Pre-commit checklist**

- [ ] The document starts with a ` ```yaml ` block and it is the **first** content in the file
- [ ] All required metadata fields are filled (`doc_id`, `doc_type`, `title`, `owner`, `tags`, `scope`)
- [ ] `doc_id` matches the required format (type-prefixed + kebab-case; see 3.3)
- [ ] `tags` match the allowed format (latin letters/digits/`-`/`_`)
- [ ] The number of `tags` elements = the number of `## **N. ...**` headings
- [ ] The document contains no other level-2 headings (`##`) besides `## **N. ...**`
- [ ] `N` numbering is sequential, unique, and increasing
- [ ] Tags semantically match the content of the sections
- [ ] There is one `#` heading after YAML (the document title)
- [ ] Internal links to other documents are written as `doc_id:<id>` (no `*.md` and no paths)


