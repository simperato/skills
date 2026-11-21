---
name: personal_memories
description: >-
  Manage long-term personal memories organized by subject (person) and topic
  (argomento) on Google Drive, with one private folder per subject and one
  Google Doc per topic. Use this skill whenever the user asks to remember,
  save, recall, list, update, or delete long-term information about a specific
  person/subject, or to manage their memory folders and documents on Google
  Drive.
---

# Personal Memories (Google Drive)

Store, retrieve, update, and delete long-term memories for individual **subjects** (`soggetto`, e.g., `Mario Rossi`), organized by **topic** (`argomento`, e.g., `Informazioni personali`, `Piano di allenamenti`) in **Google Drive**.

## Storage Model

```text
Personal Memories/                  # Dedicated root folder (private, restricted)
└── <Subject Name>/                 # One folder per subject (e.g., "Mario Rossi")
    ├── Informazioni personali      # Google Doc per topic (argomento)
    ├── Piano di allenamenti        # Google Doc per topic (argomento)
    └── ...
```

- **One folder per subject**: All memories about a subject live inside their dedicated Google Drive folder. Never mix data from different subjects in the same folder or document.
- **One Google Doc per topic**: Inside a subject's folder, create/use one native Google Doc (`application/vnd.google-apps.document`) per topic.
- **Deduplication**: Always search inside the parent folder before creating a new subject folder or topic document (match case-insensitively on trimmed names) so duplicate folders or docs are never created.

## Prerequisites & Tooling

1. Use the available Google Drive and Google Docs tools/CLIs/MCP servers in the environment (for example, the `gdrive` and `gdocs` plugins).
2. If no Google Drive / Google Docs tool is available or authenticated, **stop and ask the user** to connect/authenticate the integration. Never fall back to unapproved storage or public links.

## Security & Access Control (Mandatory)

Memories may contain sensitive personal, health, financial, or private data. Enforce strict least-privilege access on **every** folder and document:

1. **Restricted by default**:
   - The root folder, every subject folder, and every topic Google Doc MUST be **Restricted** (private).
   - **Never** grant `anyone` / `anyoneWithLink` or `domain`-wide permissions.
2. **Authorized principals only**:
   - Access is limited strictly to:
     - The **owner / authorized agent account** managing the Drive workspace.
     - Any explicitly **authorized agent accounts** configured by the user.
     - The **subject user** (via their verified Google account email), **only** if they are meant to have direct access (with the minimum required role, e.g., `reader` or `writer`).
3. **Permission verification & remediation**:
   - **On Create**: Immediately after creating a subject folder or topic Google Doc, inspect its permissions (`permissions.list`). Verify that no `anyone`, `domain`, or unauthorized user permissions were inherited or attached. Remove any unexpected permissions immediately.
   - **Before Read/Update on existing items**: Verify permissions on the subject folder/doc if there is any doubt about inheritance. If an unauthorized permission is found and cannot be removed, **stop and alert the user** before writing sensitive data.
   - **No cross-subject sharing**: Never grant Subject A access to Subject B's folder or documents.
4. **Minimal exposure in output**:
   - When replying to the user, quote or summarize only the relevant memory entries rather than dumping entire sensitive documents unnecessarily.

## Document Format

Use a consistent structure inside each topic Google Doc so entries remain easy to read and update:

```markdown
# <Topic Name> — <Subject Name>

## Memories
- [YYYY-MM-DD] <Concise, factual memory entry> (Context/Source: <optional>)
- [YYYY-MM-DD] <Concise, factual memory entry>
```

Keep entries atomic, dated, and factual.

## CRUD Operations

### 1. Create (`create` / `remember`)

1. Identify the **subject** (`soggetto`), **topic** (`argomento`), and **memory content**. If subject or topic is ambiguous, clarify with the user.
2. Locate (or create) the root `Personal Memories` folder; verify its permissions are Restricted.
3. Locate (or create) the `<Subject Name>` folder inside the root folder; verify/enforce Restricted permissions and authorized ACLs.
4. Locate (or create) the `<Topic Name>` Google Doc inside `<Subject Name>/`; verify/enforce Restricted permissions and authorized ACLs.
5. Append or insert the new dated memory entry into the Google Doc (avoiding exact duplicates).
6. Confirm to the user what was saved, under which subject and topic, and that permissions were verified.

### 2. Read (`read` / `recall` / `list`)

1. Identify the **subject** and optionally the **topic** (or search query).
2. Locate the `<Subject Name>` folder:
   - If listing topics: list all Google Docs inside `<Subject Name>/`.
   - If reading a specific topic: read that topic Google Doc and return the relevant memories.
   - If searching across a subject's memories without a specific topic: list the docs in `<Subject Name>/`, search/read the relevant ones, and synthesize the answer.
3. If the subject folder or topic doc does not exist, clearly report that no memories were found for that subject/topic.

### 3. Update (`update` / `edit`)

1. Identify the **subject**, **topic**, the existing memory to change, and the new information.
2. Locate the `<Subject Name>` folder and `<Topic Name>` Google Doc.
3. Read the current document content to locate the exact entry or section to update.
4. Update the entry in place (updating the date to `[YYYY-MM-DD]` or noting the revision) without accidentally deleting unrelated memories.
5. Confirm the updated content to the user.

### 4. Delete (`delete` / `forget`)

1. Identify whether the user wants to delete:
   - A **specific memory entry** inside a topic Google Doc,
   - An entire **topic Google Doc**, or
   - An entire **subject folder** (all memories for that subject).
2. **Confirmation gate**: Always ask for explicit user confirmation before deleting an entire topic Google Doc, deleting a subject folder, or removing significant memory content—stating the exact subject name, topic name, and Drive file/folder ID.
3. Execute the deletion (edit the doc to remove the entry, or trash/delete the doc/folder as requested).
4. Confirm completion to the user.

## Verification Checklist

Before finishing any operation, verify:
- [ ] Subject folder exists under `Personal Memories/` with no duplicate folders.
- [ ] Topic Google Doc exists inside the subject folder with no duplicate docs.
- [ ] Permissions on both the subject folder and topic doc are **Restricted** (no `anyone` or `domain` access; only the subject user if authorized and authorized agent/owner accounts).
- [ ] The requested CRUD change is reflected accurately in the Google Doc.
