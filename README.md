# mac-notes-agent

macOS **Apple 메모(Notes)** 앱과 연동하는 OpenClaw 스킬입니다.

- Apple Notes에 **노트를 생성/조회/수정/삭제/검색**할 수 있습니다.
- 내부 구현은 `osascript`(AppleScript)를 통해 Notes 앱을 제어하는 **Node.js CLI**입니다.

## 파일 구조

```text
mac-notes-agent/
├─ README.md       # 이 파일
├─ SKILL.md        # OpenClaw 스킬 메타데이터 + 사용법
└─ cli.js          # Node.js CLI, AppleScript 브리지
```

## 설치 / 전제 조건

- macOS (Notes 앱 설치되어 있어야 함)
- Node.js (OpenClaw와 동일 환경)
- `osascript` 명령이 사용 가능해야 함 (기본 macOS 포함)

별도 npm 의존성은 없고, Node.js 내장 `child_process`, `fs`만 사용합니다.

## 사용법

모든 명령은 이 디렉토리 기준에서 다음과 같이 실행할 수 있습니다.

```bash
node cli.js <command> [options]
```

OpenClaw에서 사용할 때는 보통 전체 경로로:

```bash
node skills/mac-notes-agent/cli.js add --title "제목" --body "내용" --folder "Jarvis"
```

### 주요 명령

#### 1) 노트 추가 (add)

```bash
# 인라인 본문으로 추가
node cli.js add \
  --title "오늘 회의 메모" \
  --body "첫 줄\n둘째 줄\n셋째 줄" \
  --folder "Jarvis"

# 파일에서 본문을 읽어서 추가 (긴 텍스트에 권장)
node cli.js add \
  --title "MAM RAG 리뷰 (정리본)" \
  --bodyFile "./mamrag_review.txt" \
  --folder "Jarvis"
```

- `--title` (필수): 노트 제목
- `--folder` (선택): Notes의 폴더 이름 (예: "Jarvis"). 없으면 기본 폴더 사용.
- `--body` 또는 `--bodyFile` 중 하나는 필수
  - `--body`에 들어온 문자열의 `\n`은 실제 줄바꿈으로 처리하고,
  - 내부적으로 `<div>...<br>...<br>...</div>` HTML로 변환 후 Notes에 저장합니다.

#### 2) 노트 목록 (list)

```bash
node cli.js list --folder "Jarvis"
```

- 지정 폴더 안의 노트 목록을 JSON으로 출력합니다.

#### 3) 노트 읽기 (get)

```bash
node cli.js get --folder "Jarvis" --title "MAM RAG 리뷰 (정리본)"
```

- 제목/폴더 기준으로 노트를 찾아 본문(HTML 텍스트)을 JSON으로 반환합니다.

#### 4) 노트 전체 수정 (update)

```bash
# 인라인 본문으로 교체
node cli.js update \
  --folder "Jarvis" \
  --title "MAM RAG 리뷰 (정리본)" \
  --body "새로운 내용 전체로 교체"

# 파일에서 읽어서 교체
node cli.js update \
  --folder "Jarvis" \
  --title "MAM RAG 리뷰 (정리본)" \
  --bodyFile "./mamrag_review.txt"
```

#### 5) 노트에 내용 덧붙이기 (append)

```bash
node cli.js append \
  --folder "Jarvis" \
  --title "MAM RAG 리뷰 (정리본)" \
  --body "\n---\n2026-02-09 추가 메모: ..."
```

#### 6) 노트 삭제 (delete)

```bash
node cli.js delete --folder "Jarvis" --title "MAM RAG 리뷰 (정리본)"
```

#### 7) 검색 (search)

```bash
node cli.js search --query "MAM RAG" --folder "Jarvis"
```

- 제목/본문에 키워드를 포함하는 노트를 간단히 검색합니다.

## Notes-specific behavior

- Notes는 리치 텍스트/HTML 기반이라, 줄바꿈 처리를 위해 내부적으로 `<div><br>` HTML을 생성합니다.
- `--bodyFile` 지원으로, 긴 리뷰/문서를 복붙 없이 한 번에 메모로 저장할 수 있도록 설계했습니다.

---

# mac-notes-agent (English)

This is an OpenClaw skill that integrates with the macOS **Apple Notes** app.

- Create / list / read / update / append / delete / search notes in Apple Notes.
- Implemented as a small **Node.js CLI** that talks to Notes via `osascript` (AppleScript).

## Files

```text
mac-notes-agent/
├─ README.md   # this file (Korean + English)
├─ SKILL.md    # OpenClaw skill metadata + usage
└─ cli.js      # Node.js CLI, AppleScript bridge
```

## Requirements

- macOS with the built-in Notes app
- Node.js (same environment as OpenClaw)
- `osascript` available on PATH (default on macOS)

No external npm dependencies are required. The CLI only uses Node.js built-ins
(`child_process`, `fs`).

## Usage

Run commands from this directory:

```bash
node cli.js <command> [options]
```

Typical usage from the OpenClaw repo:

```bash
node skills/mac-notes-agent/cli.js add \
  --title "Title" \
  --body "First line\nSecond line" \
  --folder "Jarvis"
```

### Core commands

#### 1) Add note (add)

```bash
# Inline body
node cli.js add \
  --title "Meeting notes" \
  --body "First line\nSecond line\nThird line" \
  --folder "Jarvis"

# From file (recommended for long text)
node cli.js add \
  --title "MAM RAG review" \
  --bodyFile "./mamrag_review.txt" \
  --folder "Jarvis"
```

- `--title` (required): Note title
- `--folder` (optional): Notes folder name (e.g. "Jarvis"). If omitted, uses the
  default folder.
- Exactly one of the following is required:
  - `--body`     : Inline text; literal `\n` sequences are converted to real
    line breaks.
  - `--bodyFile` : UTF-8 text file path; contents become the note body.

The CLI converts line breaks into a minimal HTML representation
`<div>...<br>...<br>...</div>` so that Apple Notes renders line breaks correctly.

#### 2) List notes (list)

```bash
node cli.js list --folder "Jarvis"
```

#### 3) Get note (get)

```bash
node cli.js get --folder "Jarvis" --title "MAM RAG review"
```

#### 4) Update note (update)

```bash
# Replace body from inline text
node cli.js update \
  --folder "Jarvis" \
  --title "MAM RAG review" \
  --body "New full body"

# Replace body from file
node cli.js update \
  --folder "Jarvis" \
  --title "MAM RAG review" \
  --bodyFile "./mamrag_review.txt"
```

#### 5) Append to note (append)

```bash
node cli.js append \
  --folder "Jarvis" \
  --title "MAM RAG review" \
  --body "\n---\n2026-02-09: additional notes"
```

#### 6) Delete note (delete)

```bash
node cli.js delete --folder "Jarvis" --title "MAM RAG review"
```

#### 7) Search notes (search)

```bash
node cli.js search --query "MAM RAG" --folder "Jarvis"
```

This performs a simple text search over titles and bodies. For moderate-sized
note collections this is sufficient; it is not meant for tens of thousands of
notes.
