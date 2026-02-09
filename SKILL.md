---
name: mac-notes-agent
description: |
  Integrate with the macOS Notes app (Apple Notes).
  Supports creating, listing, reading, updating, deleting, and searching notes
  via a simple Node.js CLI that bridges to AppleScript.
version: 0.1.0
author: swancho
license: CC-BY-NC-4.0
metadata:
  openclaw:
    emoji: "📝"
---

# Mac Notes Agent

## Overview

This skill lets the agent talk to **Apple Notes** on macOS using AppleScript
(via `osascript`). It is implemented as a small Node.js CLI:

```bash
node skills/mac-notes-agent/cli.js <command> [options]
```

> ⚠️ Requires macOS with the built-in **Notes** app and `osascript` available.
> The CLI must be run on the Mac host (not a sandboxed container without GUI).

All operations target the **default Notes account**. Optionally you can specify
which folder to use.

---

## Commands

### 1) Add a new note

```bash
# 직접 본문 문자열로 전달
node skills/mac-notes-agent/cli.js add \
  --title "오늘 회의 메모" \
  --body "첫 줄\n둘째 줄\n셋째 줄" \
  [--folder "Jarvis"]

# 혹은 파일에서 본문 읽어오기 (줄바꿈 많은 긴 텍스트에 추천)
node skills/mac-notes-agent/cli.js add \
  --title "MAM RAG 리뷰 (정리본)" \
  --bodyFile "./notes/mamrag_review.txt" \
  [--folder "Jarvis"]
```

- `--title` (required): Note title (name field in Notes)
- Exactly one of the following is required:
  - `--body`     : Inline body text. Literal `\n` 시퀀스는 실제 줄바꿈으로 해석됨.
  - `--bodyFile` : UTF-8 텍스트 파일 경로. 파일 내용을 그대로 본문으로 사용.
- `--folder` (optional): Folder name under the default Notes account.
  - If omitted, the note is created in the system default folder.
  - If the folder does not exist, it is created.

> 내부적으로는 줄바꿈을 `<div>...<br>...<br>...</div>` 형식의 간단한 HTML로
> 변환해서 Notes에 전달하므로, Notes 앱에서 실제 줄바꿈이 잘 보인다.

**Result (JSON on stdout):**

```json
{
  "status": "ok",
  "id": "UUID-or-internal-id-if-available",
  "title": "오늘 회의 메모",
  "folder": "Jarvis"
}
```

> Note: Apple Notes does not expose stable IDs via basic AppleScript.
> This CLI therefore uses a synthetic `id` of the form `folder::title::timestamp`
> where needed. See *Identification model* below.

---

### 2) List notes

```bash
node skills/mac-notes-agent/cli.js list [--folder "Jarvis"] [--limit 50]
```

- Lists notes in the given folder (or all folders if omitted).
- Output is JSON array of notes with `title`, `folder`, `creationDate`,
  and a synthetic `id`.

Example output:

```json
[
  {
    "id": "Jarvis::2026-02-09T08:40:00::오늘 회의 메모",
    "title": "오늘 회의 메모",
    "folder": "Jarvis",
    "creationDate": "2026-02-09T08:40:00+09:00"
  },
  {
    "id": "Jarvis::2026-02-09T08:10:00::MAMRAG 리뷰",
    "title": "MAMRAG 리뷰",
    "folder": "Jarvis",
    "creationDate": "2026-02-09T08:10:00+09:00"
  }
]
```

---

### 3) Read a note (get)

You can locate a note by `id` (if you stored it) or by `(folder, title)`.

```bash
# By folder + title
node skills/mac-notes-agent/cli.js get \
  --folder "Jarvis" \
  --title "MAMRAG 리뷰"

# (Optional) by synthetic id if you saved it earlier
node skills/mac-notes-agent/cli.js get --id "Jarvis::2026-02-09T08:10:00::MAMRAG 리뷰"
```

**Output (JSON):**

```json
{
  "title": "MAMRAG 리뷰",
  "folder": "Jarvis",
  "body": "<div>... HTML-ish body ...</div>",
  "plainText": "... plain text version ..."
}
```

The CLI returns both the raw rich-text body and a best-effort plain text
extraction.

---

### 4) Update a note (replace body)

```bash
# 문자열로 바로 교체
node skills/mac-notes-agent/cli.js update \
  --folder "Jarvis" \
  --title "MAM RAG 리뷰 (정리본)" \
  --body "첫 줄\n둘째 줄\n셋째 줄"

# 파일에서 읽어서 교체
node skills/mac-notes-agent/cli.js update \
  --folder "Jarvis" \
  --title "MAM RAG 리뷰 (정리본)" \
  --bodyFile "./notes/mamrag_review.txt"
```

- Replaces the entire body of the matching note.
- Identification can also be done by `--id` if you stored it.
- Exactly one of `--body` or `--bodyFile` is required.

> ⚠️ No partial edit: this is **replace-all** semantics. If you need
> append/prepend behavior, use `append` below.

---

### 5) Append to a note

```bash
# 문자열로 덧붙이기
node skills/mac-notes-agent/cli.js append \
  --folder "Jarvis" \
  --title "MAM RAG 리뷰 (정리본)" \
  --body "\n---\n2026-02-09 추가 메모: ..."

# 파일에서 읽어서 덧붙이기
node skills/mac-notes-agent/cli.js append \
  --folder "Jarvis" \
  --title "MAM RAG 리뷰 (정리본)" \
  --bodyFile "./notes/mamrag_addendum.txt"
```

- Reads the existing note, concatenates the new body (after HTML 변환) to the end,
  and writes back.
- 줄바꿈은 `<br>`로 렌더링되어 Notes에서 자연스럽게 보인다.

---

### 6) Delete a note

```bash
node skills/mac-notes-agent/cli.js delete \
  --folder "Jarvis" \
  --title "MAMRAG 리뷰"
```

- Deletes the first note that matches folder + title.
- If multiple notes share the same title in a folder, the CLI deletes the
  most recently created one (based on Notes ordering) and reports which one
  was deleted.

**Output:**

```json
{ "status": "ok", "deleted": { "title": "MAMRAG 리뷰", "folder": "Jarvis" } }
```

---

### 7) Search notes

```bash
node skills/mac-notes-agent/cli.js search \
  --query "MAMRAG" \
  [--folder "Jarvis"] \
  [--limit 20]
```

- Performs a basic search over note titles and bodies (AppleScript-side filter).
- Returns a JSON array of notes with `title`, `folder`, and a short snippet.

> Note: Apple Notes AppleScript API does not provide full-text indexing.
> The CLI will fetch candidate notes and filter them in Node.js. This is
> fine for moderate numbers of notes but not meant for tens of thousands.

---

## Identification Model & Limitations

Apple Notes' AppleScript interface does not expose a stable, portable unique
ID for each note in a friendly way. This CLI therefore uses a **best-effort
identification model**:

- Primary key for most operations: `(folderName, title)`
- Synthetic `id` string for convenience: `folderName::ISO-creationDate::title`

When multiple notes share the same title in the same folder, the CLI defaults
to operating on the **most recently created** one. For critical workflows,
prefer using the synthetic `id` returned from `add`/`list`.

---

## Implementation Notes

- Language: Node.js (no external npm dependencies; uses `child_process` only).
- Bridge: `osascript` to run small AppleScript snippets.
- Encoding: Assumes UTF-8.
- Safety: Does not touch iCloud/other accounts explicitly; always uses
  `default account` from the Notes app.

---

## Typical Usage Patterns

### Add quick scratch notes

```bash
node skills/mac-notes-agent/cli.js add \
  --folder "Jarvis" \
  --title "2026-02-09 일정 체크" \
  --body "- jptaku 중간점검 15:00\n- 강남 모두연 19:00"
```

### Maintain a running log

```bash
node skills/mac-notes-agent/cli.js append \
  --folder "Jarvis" \
  --title "MAMRAG 리뷰" \
  --body "\n[2026-02-09 08:30] 논문 공격 포인트 정리 완료"
```

### Search notes for a keyword

```bash
node skills/mac-notes-agent/cli.js search --query "MAMRAG" --folder "Jarvis"
```

This skill is intended to mirror the ergonomics of `mac-reminders-agent`, but
for Apple Notes. When in doubt, prefer simple folder+title-based operations
and keep note titles unique within a folder.
