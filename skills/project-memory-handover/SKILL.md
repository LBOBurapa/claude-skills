---
name: project-memory-handover
description: Save, load and hand over the working state of ANY project (web app, Apps Script, data/Power BI, documents, slides, games, anything) across chat sessions and Claude accounts. Use whenever the user names a project to load or continue, says "Save", "Update memory", "Change account", "Switch to [project]", or "Status", or asks to resume work from where it stopped. Applies to every project regardless of name or type. Keeps one short Markdown memory file per project, a project index, and a versioned backup, and never mixes projects.
---

# Project Memory & Handover

This skill lets work continue seamlessly across chat sessions and across different Claude accounts (for example when a usage limit is reached). Each project has its own memory file. Claude reads it at the start and writes it at every meaningful milestone, so the next account can resume with no copy-paste from the user.

Reply to the user in the language they write in. Keep the memory files themselves in the language the user uses for the project.

---

## 1. Principles

1. **One project, one memory.** Every project has its own memory file. Never read or write another project's memory during a task.
2. **Load before you act.** When a project is named, read its memory before doing any work on it.
3. **Save at milestones, not at the limit.** Usage limits cannot be seen in advance, so save automatically after each meaningful step.
4. **Verify every save.** Read the memory back after writing. Only then say it was saved.
5. **Short memory, detailed pointers.** The memory holds state, decisions and locations. It does not hold full code, full data or secrets.
6. **Never guess which project.** If the name is ambiguous, ask.
7. **Never store secrets.** API keys, passwords, tokens, OAuth or clasp credential files, and private keys never go into memory, files, repos or chat.

---

## 2. Storage layout

### 2.0 Where memory lives (overrides everything below)
- All project memory lives only in **Google Drive**, in the folder `Claude_Memory` (folder ID `1HcMcQCjxPrNccVrDI4KF-TQhRaum6-v4`). Never look for memory anywhere else, and never use the chat's built-in memory for projects.
- When the user types "โหลดความจำ", "โหลด", "บันทึกความจำ", or "บันทึก", go to that Drive folder right away. Do not ask where the memory is.
- Actual file layout in Drive: `Claude_Memory/_index.md` (registry) and `Claude_Memory/<slug>/mem_vN.md` (one file per version, never edited in place). Load the highest N. Save by creating `mem_v(N+1).md`. Requirement and design files sit in the same project folder.
- If nothing is found in Drive, say so plainly. Do not guess.
- For any **new** app, web, or Apps Script project, before writing code, walk the user through GitHub setup (section 3.8) so code can flow from GitHub into Apps Script.

### 2.1 Memory root
Use one memory root for all projects. The default name is `Claude_Memory/` (in Google Drive, a local folder, or whichever storage the environment provides). The user may choose another name once; remember it in the index.

```
Claude_Memory/
  _index.md              <- project registry (read at startup, see 2.2)
  <project-slug>/
    MEM.md               <- the current memory (the only file read at startup)
    MEM.bak.md           <- previous version (backup)
    notes/               <- optional longer notes, linked from MEM.md
```

`<project-slug>` is a short lowercase name with hyphens, for example `drink-shop-app`, `p4-english-slides`, `powerbi-sales`.

### 2.2 Project index: `_index.md`
The index maps the names the user may say to a project. It is the only place the project list lives.

```
---
name: _index
description: Registry of all projects, their names, memory paths and code locations.
---

| Slug | Names the user may use (aliases) | Memory path | Code / data location | Status |
|---|---|---|---|---|
| drink-shop-app | Web App, drink-shop-app, ร้านน้ำ | Claude_Memory/drink-shop-app/MEM.md | GitHub: owner/drink-shop-app, branch main | Active |
```

Rules for the index:
- Match the user's words against the Slug and the aliases, case-insensitive, including Thai names.
- When a project is created, add a row. Tell the user the row was added.
- Never delete a row unless the user asks.

### 2.3 Storage method (choose by what the environment allows)

The memory file is Markdown (`.md`). How it is written depends on the tools available. Check once at session start (see section 3.0).

| Option | When to use | How to save a new version |
|---|---|---|
| A. Local filesystem with Write/Edit | Claude can write files in the workspace or a mounted folder | Edit MEM.md in place; copy the old file to MEM.bak.md first |
| B. Google Drive with a text connector | Only Drive connectors exist and content can be created but not overwritten | Create a new immutable file `mem_v[N].md` each save (this is the Drive convention in section 2.0). Keep only the last few versions if asked |
| C. Inside the project's Git repo | The project already has a repo and Claude can commit | Save as `MEMORY.md` at the repo root (or `docs/MEMORY.md`), commit and push. Git history becomes the version history |
| D. Google Doc with a Docs connector | The user prefers a Doc; the Docs connector can update content | Same schema, stored as a Doc. Backup with copy before editing |

Never claim you updated the content of a file when the tool only changed metadata or created a new file. If the method cannot overwrite content, use the method that works and say so.

---

## 3. Commands and procedures

### 3.0 Session start check (once per session)
Confirm which of these you can do: search the memory root, read a file, create or edit a file, and copy a file. If a needed capability is missing, tell the user at once and say what is not possible. Do not pretend to save. Load deferred tool schemas first if the environment requires it.

### 3.1 Load: `Load [project]` or any mention of resuming a project
1. Match the name against `_index.md` (Slug or aliases). If the index is missing, search the memory root for `MEM.md` or `mem_v*.md` files and match by their title. If nothing matches, or several are similar, ask the user which project they mean, or whether to create a new one. Never guess.
2. Read the project's latest memory file in full (highest version number). Note `Version` and `Last saved at`.
3. Reply briefly (in the user's language) with:
   - the project goal (one line)
   - the last completed task
   - the immediate next step
   - the version loaded
4. Start the immediate next step unless the user says otherwise. Load code, files or repos only when that step needs them.

### 3.2 Create a new project
Triggered when the user starts a project that is not in the index, or says "new project".
1. Create the folder `<memory-root>/<project-slug>/`.
2. Create the first memory file (`MEM.md` or `mem_v1.md`) with the schema in section 4. Fill what is known. Leave unknown fields as "TBD" and ask only for what blocks the next step.
3. Add a row to `_index.md`. Tell the user the row was added.
4. Run the GitHub setup in section 3.8 before writing any code.
5. Continue with the work.

### 3.3 Save: `Save`, `Update memory`, `Change account`, or an automatic milestone save
Save automatically after:
- a subtask is finished,
- an important decision is made,
- a significant change is made to code, data or document,
- a user-run step is confirmed (for example the user tested something),
- before any long or risky operation (large refactor, data import, deletion, deployment).

Save procedure, in order:
1. **Conflict check** (section 3.5).
2. **Write** the new memory: in Drive, create `mem_v(N+1).md`; with a local file, back up the old one first.
3. Update the sections that changed. Increase `Version` by 1, set `Last saved at` in the user's timezone, and add one line to the Change Log.
4. **Code or data changed?** It must already be committed, pushed, or saved in a file the memory points to. Record the commit hash, file names, sheet IDs or document IDs in section 8 of the memory. Never write "pushed" or "saved" unless it was.
5. **Verify**: read the new file back and confirm it matches what you intended.
6. **Confirm** only after all steps succeed, for example: "State saved to `<project>` (v[N]). You may switch accounts now."

If any step fails, say so plainly with the reason, and do not claim the save succeeded. If the version is not yet saved, tell the user not to switch accounts.

### 3.4 Switch project: `Switch to [project]`
1. Save the current project (3.3) if it has unsaved changes.
2. Load the new project (3.1).
Never mix content from the two projects.

### 3.5 Conflict check (before every save)
Read the latest version number again. If it is higher than the version you loaded, another account or session has saved in the meantime.
- Do not overwrite.
- Show the user the differences, section by section.
- Ask how to merge (keep theirs, keep yours, or combine).
- After the user decides, save with the new version number.

### 3.6 Status: `Status`
Show the current version, last save time, status and immediate next step. Change nothing.

### 3.7 Honesty and limits
- If the conversation is getting long, or the work has been running for a while, save proactively and tell the user.
- Never say something was saved, tested, deployed, pushed or confirmed unless it was.
- If a tool result is unclear, verify by reading back before reporting.

### 3.8 GitHub to Apps Script setup (for every new app, web or Apps Script project)
Go through these steps with the user one at a time, waiting for confirmation at each step. Use simple steps for phone or computer. Never ask the user to paste secrets into the chat.
1. Ask whether a GitHub repo already exists. If not, have the user create a **private** repo named after the slug at github.com/new.
2. Create the Apps Script project at script.google.com (New project) and bind it to the spreadsheet if one is used.
3. Turn on the Apps Script API at script.google.com/home/usersettings.
4. Install clasp on a computer with Node.js: `npm install -g @google/clasp`, then run `clasp login` with the owner's Google account. The credential file is `.clasprc.json` in the user's home folder (Windows: `C:\Users\<name>\.clasprc.json`).
5. In the repo, go to Settings, then Secrets and variables, then Actions, and add two repository secrets:
   - `CLASPRC_JSON`: the full contents of `.clasprc.json`, pasted in GitHub only.
   - `APPS_SCRIPT_ID`: the Script ID from Project Settings in Apps Script.
6. Add `.github/workflows/deploy-apps-script.yml`, which runs the tests and then `clasp push` when a commit lands on main.
7. Confirm that the workflow passes on the first push before calling the setup done.
8. Record in memory which secrets exist (names only), the repo, the branch and the Script ID location, but never the values.

If the project is not Apps Script, first ask which host it will run on, and still set up GitHub first.

---

## 4. Memory schema

Use this structure exactly so every account reads it the same way. Keep the whole file under about 150 lines. Write the content in the language used on the project.

```
---
name: <project-slug>
description: <one line: what the project is, and what is in this file>
---

# <Project name>

## Meta
- Version: <integer, +1 per save>
- Last saved by: Account #<1-8 or name>
- Last saved at: <YYYY-MM-DD HH:MM, user's timezone>
- Status: <Active | Paused | Done>

## 1. Goal
<What the project is for, 2-4 lines.>

## 2. Current State
<What exists and works right now. Be specific.>

## 3. Last Completed Task
<What was just finished, with the date.>

## 4. Immediate Next Step
<The exact first action for the next account, specific enough to start without asking.>

## 5. Task Backlog
- [ ] <task>
- [x] <done task>

## 6. Key Decisions and Constraints
- <decision / requirement / preference, with the reason if useful>

## 7. Open Issues and Bugs
- <known problem, or something not yet verified>

## 8. Files and Locations
- Code: <repo, branch, last commit hash, or file names with version>
- Data: <sheet IDs, document IDs, folder paths, database names>
- Deployment: <where it runs, deployment ID or URL, if known>
- Notes: <links to notes/ files>

## 9. User Preferences for this Project
- Language: <...>
- Tone and format: <...>
- Style rules: <...>

## 10. Change Log (keep the latest 10)
- v[N] (Account #X, date): <what changed>
```

Section rules:
- Section 4 must be concrete enough to start without questions.
- Section 6 holds decisions only when the user made them or accepted them. Do not record your own suggestions as decisions.
- Section 8 holds identifiers and locations, never secrets.
- Keep the Change Log short; move older entries out of the file if needed.

---

## 5. Code, data and repositories (general)

Store the code or data outside memory and point to it.

- **Git repo (preferred when it exists).** Name the repo, the branch and the last commit in section 8. Attach the repo only when the next step needs it. If a repo connection is missing in the current account, tell the user how to connect it, and grant access only to the repo this project needs. Never make a private repo public.
- **Without a repo.** Store code as versioned files, one new file per version, named `<project>_<file>_v<N>.<ext>`. Record the names in section 8.
- **Data files (sheets, datasets, documents).** Record the ID or path and the tab or section names. Never copy large data into memory.
- **Apps Script projects (Google Sheets and similar).**
  - Keep the Script ID, the bound spreadsheet ID, the deployment ID and the web app URL in section 8.
  - Code lives in the repo; `clasp push` (run by the user or by Claude where possible) sends it to the script. Record which commit was pushed.
  - Record the sheet tab names and the column headers that the code depends on.
  - Never store `.clasprc.json`, OAuth tokens or any credential in memory, files or the repo.
- **Power BI and other models.** Record the file name, its location, the model version and the group or workspace it belongs to, plus the naming and modeling decisions made so far.
- **Slides, documents and designs.** Record the file or artifact ID, the version, the theme and the open edits.

---

## 6. Secrets (hard rule)
Never put these in memory, files, repos, commit messages or chat: API keys, passwords, access tokens, refresh tokens, OAuth client secrets, `.clasprc.json`, `.env` contents, private keys, or personal account credentials. If the user pastes one, tell them not to share it and that it will not be stored. Refer to where a secret lives (for example "stored in the user's password manager" or "stored in GitHub Secrets") instead.

---

## 7. Account notes (multi-account use)
- Each Claude account has its own connections (repo, Drive, Docs, Sheets). When a connection is missing in a new account, tell the user exactly what to connect in settings. Do not claim it is connected until a tool call succeeds.
- Connect Google services with the same Google account that owns the files.
- Ask the user once which account number they are on, and record it in `Last saved by`.

---

## 8. Quick commands
| The user says | Do |
|---|---|
| `Load <project>` or names a project, or "โหลดความจำ" | 3.1 |
| `Save`, `Update memory`, "บันทึกความจำ" | 3.3 |
| `Change account` | 3.3, then confirm it is safe to switch |
| `Switch to <project>` | 3.4 |
| `Status` | 3.6 |
| `New project <name>` | 3.2 (includes 3.8) |

When the user names a project in any other way (for example "continue the sales dashboard" or "back to the slides"), treat it as `Load` and match it against the index. If the match is unclear, ask.

---

## 9. Reply examples

Loading (in the user's language):
> Loaded `drink-shop-app` (v12, saved by Account #3 on 2026-11-20 14:10).
> Goal: a web app for drink-shop ingredients and sales.
> Last completed: sales report page now reads from the Sheet.
> Next step: fix the date filter on the report page. Starting now.

Saved:
> State saved to `drink-shop-app` (v13). Verified by reading back. You may switch accounts now.

Conflict:
> The memory is at v14, but this session loaded v12. Another account saved in between. Differences:
> - Section 4: the next step changed from ... to ...
> - Change Log: a new entry v13 (Account #5)
> Keep theirs, keep mine, or combine?

Missing capability:
> This account cannot write to the memory folder. I can continue the work, but the state will not be saved here. Please connect Google Drive in settings, or tell me another place to save.

---

## 10. Creating this skill for a new environment
If a Claude environment has no memory tools at all, apply the same procedure with a single `MEM.md` file in the workspace, and tell the user where it is. The rules above still apply: one project per file, verify each save, never store secrets.