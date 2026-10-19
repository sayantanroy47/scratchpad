# Sidebar and session management: implementation spec

Goal: make the harness's session sidebar and session management look and behave like the reference screenshot (Pinned section, project groups with headers, two-line rows, nested child rows, status dots, Show more).

This spec is for the harness built on the Copilot SDK. It is self-contained so an implementer can build it in a separate repository. It describes structure and behaviour, not code.

Source tags:
- **OBS**: seen in the reference screenshot.
- **MOCK**: taken from the harness mockup in this repo (`harness-ui-mockup.html`).

Visual polish (type scale, spacing rhythm, dark-mode tokens, final colours) is a separate pass. Use the **impeccable** skill for it on the other machine, after the structure below is built.

---

## 1. Shell layout (desktop)

Left to right:

```
+------+----------------+--------------------------------------+------+
| host |    sidebar     |  main column (title bar on top,      | right|
| rail |  (sessions)    |  chat or code workspace, status bar  | rail |
| 44px |   300-340px    |  at bottom)                          | 44px |
+------+----------------+--------------------------------------+------+
```

- **Host rail, 44px, far left [MOCK].** The host app's own navigation. The harness adds nothing to it. It is a placeholder column in the mockup.
- **Sidebar [OBS][MOCK].** This spec's subject. Default width 300–340px, resizable between about 220px and 420px. Background is the surface colour, with a 1px border on its right edge.
- **Title bar, 40px, above the main column [MOCK].** Holds the Chat / Code segmented control, the Details switch, the sidebar toggle and the theme toggle.
- **Main column.** Chat thread (Chat mode) or code workspace (Code mode). See section 9.
- **Right rail, 44px icon column [MOCK].** Opens harness views (Agents, Task board, Mailbox, Run, Plan, Background, Worktrees, Event log). Outside this spec, except that it sits to the right of the sidebar.
- **Status bar, bottom [MOCK].**

Sidebar visibility:
- Toggled with a sidebar icon button in the title bar [MOCK]. When hidden, the sidebar is removed from the grid (`display: none`). It is not collapsed to a strip.
- Under 768px wide, the sidebar becomes a fixed overlay drawer, about 88vw wide and at most 320px, that slides in from the left (0.2s transform) with a backdrop that closes it [MOCK].

---

## 2. Sidebar anatomy, top to bottom

### 2.1 Top of the sidebar

- **OBS:** the reference has no large heading and no visible search box. The first thing is the Pinned section header.
- **Rule:** a slim toolbar, about 32px tall, sits at the top with three controls:
  - **Filter** button: opens a popover to filter by status (Running, Needs input, Unread, Idle, Done), by project (multi-select), and by mode (Chat, Code).
  - **Group** toggle: group by project, or show one flat list.
  - **New session** button (pencil or plus icon). Tooltip: "New session, Ctrl+N".
- Search is inside the filter popover, not a permanent bar.

### 2.2 Section: Pinned

- Header reads "Pinned" in muted, small text (about 12px), with a **down chevron**. Clicking collapses or expands the section. **OBS**
- Pinned rows sit directly below the header, in the same row layout as the rest. **MOCK**
- A pinned run appears **under Pinned only**, with its project shown as its second line. It is not repeated in its project group.
- Pinned rows are sorted by most recent activity, newest first.
- Hide the whole section when nothing is pinned.

### 2.3 Section: Sessions (the main list)

- Sessions are grouped under **project headers**. The reference shows five headers in this order: "Adda", "AI guide", "archify", "khai-khai", "Ninicons". **OBS**
- In the harness, a project is a repository (for example `payments-api` in the mockup). **MOCK**

#### 2.3.1 Project header

- Height about 30px. Label is the same size as session titles (13px) and semibold, in the strong text colour, so it reads as a header. **MOCK**
- **Chevron:** right-pointing when collapsed, down when expanded. The reference mixes these and shows no children under collapsed headers. **OBS**
- **"+" button** at the right edge starts a new session in that project. It is always visible in the reference. **OBS**
- **Empty projects are listed.** "Adda", "AI guide" and "archify" have no sessions and still show as headers. The project list comes from known repositories, not only from sessions that exist. **OBS**
- **Order:** case-insensitive alphabetical by project display name. The reference is not sorted by recent activity: Ninicons has the newest session and sits last. **OBS**
- **Identity:** key groups by repository identity (machine + path or repo id), not by folder name alone. Two machines with the same path must not merge into one group.
- **Display name:** stored separately from the path. The reference shows "khai-khai" for a folder named "KhaiKhai". **OBS**

#### 2.3.2 Run row (two-line row)

Two-line rows are about **50px pitch**. One-line rows are about **30px pitch**. **OBS**

- **Line 1, title:** the run's title, regular weight, about 13.5–14px, one line, truncated with an ellipsis. The full title shows as a tooltip on hover.
- **Line 2, meta:** a branch glyph, then the branch name, muted, about 12px. Always shown. Runs on the main checkout show `main`. **MOCK**
- **Right edge:** one status indicator (section 3).
- **Child rows** (section 2.3.3) use the same layout, indented.

#### 2.3.3 Child rows (agents under a run)

- A **run** is the orchestrator's session. Its **children** are the subagents and teammates it started. **MOCK**
- Children are nested under their parent with an **elbow connector** (an L-shaped line from the parent's left edge down to each child). They are indented about 20px from the parent's title. **OBS**
- Each child stores a `parentId` that points to its run or to its orchestrator. Nesting follows that field. **MOCK** (`sb-kids`)
- Children are sorted by most recent activity, newest first. A parent keeps its position in the list regardless of child activity.
- A run with no children has no connector. **OBS**

#### 2.3.4 Show more

- A "Show more" text link sits at the bottom of the list, muted, aligned with the title indent. **OBS**, **MOCK** ("Show more sessions")
- Each click adds six more runs. The first page holds the most recent runs. **MOCK**

---

## 3. Status indicators on the right edge

One indicator per row. Show the highest-priority one that applies.

| Priority | Indicator | Looks like | Harness status it maps to |
|---|---|---|---|
| 1 | Warning triangle | Outline triangle with "!", about 14px, muted amber | Run or child is `cancelled`, or has an error. |
| 2 | Needs input | Yellow dot with a soft halo | Child is in `permission` (waiting for a tool approval). Also a run that has a held question. **MOCK** (`sdot need`) |
| 3 | Running | Spinning ring (the mock's running glyph). A child that is `waiting` gets the dotted spinner. A run shows the running glyph while any child is active. **MOCK** |
| 4 | Unread | Blue dot, about 7px | The run finished (`done`) or changed since the user last opened it. **OBS** |
| — | None | Nothing | `idle`, or `done` and already read. |

Rules:
- Clicking a row marks it read and clears its unread dot. A running row keeps its running dot.
- A child's status shows on the child row. The parent run shows the highest-priority status among its children, so a run with one child in `permission` shows the yellow dot. **MOCK**
- OS notifications are sent for permission requests and for runs that finish while the user is not viewing them. The sidebar dot is separate from the notification.

---

## 4. Data model

Use the harness's own names where they exist. The mockup's agent record has most of these fields.

| Field | Type | Notes |
|---|---|---|
| `id` | string | Stable id. The mockup uses short ids such as `7f3c09a2`. **MOCK** |
| `title` | string | The run's title. Renamable. |
| `kind` | `orchestrator` / `subagent` / `teammate` / `quick` | **MOCK** (`kind`, `role`) |
| `role` | `lead` / `subagent` / `teammate` | **MOCK** |
| `parentId` | id or null | The run or orchestrator that started this agent. Drives nesting. **MOCK** |
| `project` | ref | The repository this run belongs to. Drives grouping. |
| `where` | string | `main (read-only)` or `wt/<agent>`. **MOCK** |
| `branch` | string or null | Set for worktree agents, for example `harness/r12/coder-1-T2`. **MOCK** |
| `status` | `running` / `waiting` / `permission` / `idle` / `done` / `cancelled` | **MOCK** (`AG[*].status`) |
| `task` | string or null | Task id, for example `T2`. **MOCK** |
| `unread` | bool | Drives the unread dot. |
| `pinned` | bool | Shows under Pinned. |
| `archived` | bool | Hidden from the list. |
| `lastActivityAt` | timestamp | Drives sorting. |
| `ctx` | number (0–100) | Context use. Shown in the agent view, not on the row. **MOCK** |

---

## 5. Sorting, filtering and grouping rules

1. **Sections:** Pinned (recency, newest first), then Sessions (grouped by project, recency, newest first).
2. **Projects:** case-insensitive alphabetical by display name. Empty projects still show.
3. **Runs inside a project:** most recent `lastActivityAt` first.
4. **Children:** nested under `parentId`, most recent first. The parent keeps its position.
5. **Pinned:** most recent first. Each pinned run appears only here.
6. **Archived:** hidden from the list by default. Archiving is done from the row's hover action (section 6.2).
7. **Filter:** status (Running, Needs input, Unread, Idle, Done), project (multi-select), mode (Chat, Code).
8. **Group toggle:** when off, one flat list, most recent first.
9. **No manual reorder and no drag and drop.** Order is automatic.

---

## 6. Interactions

### 6.1 Click and modifiers

- **Click a run row:** opens it in the main column and marks it read. **MOCK** (`data-act="open"`)
- **Click a child row:** opens that agent's view and marks it read.
- **Ctrl+click:** opens the row in a second pane beside the focused one. A normal click then replaces the focused pane. **Ctrl+\\** closes the focused pane.
- **Click a project header:** collapses or expands its rows.
- **Click a section header (Pinned, Sessions):** collapses or expands the section.
- **Click the "+" on a project header:** starts a new run in that project.

### 6.2 Hover

- **Run row hover:** shows an **archive icon** at the right edge, before the status dot. Clicking it archives the run and removes it from the list.
- **Hover on any row:** the title shows its full text as a tooltip.

### 6.3 Right-click context menu

- Rename
- Pin / Unpin
- Mark as unread
- Archive
- Copy branch name (worktree rows only)

### 6.4 Keyboard

| Shortcut (Windows) | Action |
|---|---|
| Ctrl+N | New run |
| Ctrl+W | Close the current run view |
| Ctrl+Tab / Ctrl+Shift+Tab | Next / previous run |
| Ctrl+\\ | Close the focused pane |
| Ctrl+/ | Show all shortcuts |

Keyboard focus: rows are reachable by Tab. Up and Down move between rows, and Enter opens the focused row. The nested structure uses `role="tree"` and `role="treeitem"`, with `aria-level` on child rows. The focus ring is visible, not only a background change.

### 6.5 Drag and drop

- Not supported. Do not build it.

---

## 7. States

| State | What the user sees |
|---|---|
| **No runs at all** | Project headers for known repositories still show. With no repositories either, one muted line: "No runs yet. Start one with the + button or Ctrl+N." |
| **Project with no runs** | Header with a collapsed chevron and no rows. **OBS** |
| **Loading** | Three grey skeleton rows at row pitch under the toolbar. The toolbar stays visible. |
| **Load error** | One muted line with a retry link inside the Sessions section. |
| **Long title or branch name** | One line, ellipsis, full text in a tooltip. **OBS** |
| **Running** | Spinning ring glyph. **MOCK** |
| **Needs input (permission)** | Yellow dot with halo. **MOCK** |
| **Cancelled or error** | Warning triangle. **OBS** |
| **Unread** | Blue dot. Clears on click. **OBS** |
| **Active (selected) row** | Background one step above the sidebar surface, strong title text, meta text in the accent colour. **MOCK** |
| **Hover row** | Background tint, archive icon on the right. **MOCK** |
| **Focused row (keyboard)** | Visible focus ring. |
| **Collapsed section or project** | Chevron changes direction; rows hidden; header stays. **OBS** |
| **Sidebar hidden** | Removed from the grid; main column takes the space. **MOCK** |
| **Narrow screen (under 768px)** | Overlay drawer with backdrop. **MOCK** |
| **Archived run** | Not in the list. |

---

## 8. Visual specification (from the reference screenshot)

Approximate values. Sample exact colours from the screenshot before fixing tokens. Keep all colours as CSS variables with a dark set and a light set. **OBS**

- Sidebar background: near-black, warm grey (roughly `#1c1b1a`).
- Row hover and active background: one step lighter than the sidebar background.
- Primary text (titles): off-white (roughly `#e8e6e1`).
- Muted text (headers, meta): mid grey (roughly `#8f8c86`).
- Blue unread dot: roughly `#3b82f6`.
- Warning triangle: muted amber or orange.

Type and spacing:
- Section header: about 12px, muted, medium weight, about 30px tall.
- Project header: 13px, semibold, strong text colour, about 30px tall. Same size as session titles.
- Run title: about 13.5–14px, regular weight, primary colour, about 20px line height.
- Meta line: about 12px, muted, about 16px line height.
- Row pitch: about 50px for two-line rows, about 30px for one-line rows. Gap between rows about 6px.
- Child indent about 20px. Elbow connector about 6px from the parent's left edge.
- Sidebar horizontal padding about 8px.
- Icons about 14–16px, stroke-based, muted colour.

---

## 9. Chat mode vs Code mode

The harness has two modes, set by the segmented control in the title bar. **MOCK**

| Aspect | Chat mode | Code mode |
|---|---|---|
| Sidebar content | The run list (section 2) with child agents nested under their runs. **MOCK** (`sessionsHtml`) | Three tabs at the top: **Explorer** (file tree of the selected worktree), **Changes** (changed files, with a count), **Sessions** (the run list, same component as Chat mode). **MOCK** (`codeSideHtml`) |
| Main column | The orchestrator's conversation. | The file editor for the selected worktree (read-only). **MOCK** |
| Worktree picker | Not shown. | A row of buttons: `coder-1`, `coder-2`, `tester`, `main`. Selecting one loads that worktree's tree and changes. **MOCK** (`wtpick`) |
| Right rail | Same in both modes. | Same in both modes. |

---

## 10. Gap list: mockup to target

Work through these in order.

1. Remove the big "Sessions" heading. Use the slim toolbar in section 2.1.
2. Add the Pinned section at the top, with a collapse chevron.
3. Group runs by project, with project headers that have a "+" button and a collapse chevron. Sort projects case-insensitively. Show empty projects.
4. Change the run row to two lines: title on line 1, branch or project on line 2, with the correct glyph.
5. Indent child rows with the elbow connector, driven by `parentId`. Requires step 3.
6. Replace the status dots with the priority system in section 3, including the unread dot and the warning triangle.
7. Add hover actions: archive icon on the row.
8. Add the Show more link at the bottom of the list.
9. Add the right-click context menu.
10. Add Ctrl+click to open a second pane.
11. Wire the keyboard shortcuts in section 6.4.
12. Add the filter popover and the group toggle.
13. Add the states in section 7 that are missing: empty, loading, error, warning, unread, collapsed project.
14. Keep the Code mode tabs (Explorer, Changes, Sessions) and have the Sessions tab reuse the run list component.

Step 6 can run alongside step 4.

---

## 11. Acceptance checklist

Check each item in the built app, not only in code.

- [ ] Sidebar default width is 300–340px and resizes within bounds.
- [ ] The title bar toggle hides the sidebar completely; the main column takes the space.
- [ ] Under 768px the sidebar is an overlay drawer with a backdrop.
- [ ] Pinned shows pinned runs only, most recent first, and hides when empty.
- [ ] Project headers are in case-insensitive alphabetical order, including projects with no runs.
- [ ] A project header's "+" starts a new run in that project.
- [ ] A project header collapses and expands its rows, and its chevron changes direction.
- [ ] Runs inside a project are sorted by most recent activity.
- [ ] Every run and agent row shows its branch on the second line (`main` for main-checkout runs).
- [ ] Project headers and session titles are both 13px; headers are semibold.
- [ ] Running runs and agents show the spinning glyph; waiting agents show the dotted spinner.
- [ ] Child agents are indented under their `parentId` with the elbow connector. The parent stays in place.
- [ ] Status dots follow the priority order: warning, needs input, running, unread. Clicking a row clears its unread dot.
- [ ] A run whose child is in `permission` shows the yellow dot on the run row.
- [ ] Hovering a row shows the archive icon; clicking it archives the run and removes it from the list.
- [ ] Show more loads the next page.
- [ ] Ctrl+N starts a new run; Ctrl+Tab and Ctrl+Shift+Tab move between runs; Ctrl+W closes the run view; Ctrl+\\ closes the focused pane.
- [ ] Ctrl+click opens a second pane; a normal click replaces the focused pane.
- [ ] Filter by status, project and mode works, and the group toggle works.
- [ ] Empty, loading, error, warning, unread and collapsed states match section 7.
- [ ] Light and dark themes both work, with colours from CSS variables.
- [ ] Rows are keyboard-reachable with a visible focus ring, and the ARIA tree roles match the nesting.
