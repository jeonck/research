# notebooklm-py 스킬 설치·사용 가이드

[notebooklm-py](https://github.com/teng-lin/notebooklm-py)(비공식 Google NotebookLM 자동화 CLI/Python 라이브러리)와, Claude Code에 설치된 `notebooklm` 스킬을 정리한 문서입니다.

- 기준 버전: `notebooklm-py` **0.8.5**
- `skill/` 폴더: 현재 설치된 `~/.claude/skills/notebooklm/` 사본 (원저작자 Teng Lin, MIT)

> 비공식 API라서 Google 쪽 변경으로 동작이 깨질 수 있습니다. 중요한 작업엔 쓰지 마세요.

---

## 1. 설치

Python 3.10+ 필요.

```bash
# 패키지 (CLI + 브라우저 로그인)
pip install "notebooklm-py[browser]"

# 선택: 브라우저 쿠키 추출 로그인 / 헤드리스(서버·CI)용
pip install "notebooklm-py[cookies]"
pip install "notebooklm-py[headless]"
```

`externally-managed-environment` 오류가 나면 `--break-system-packages`를 쓰지 말고 CLI 전용 설치를 쓰세요.

```bash
uv tool install "notebooklm-py[browser]"   # 또는
pipx install "notebooklm-py[browser]"
```

설치 확인:

```bash
notebooklm --version     # NotebookLM CLI, version 0.8.5
```

## 2. 스킬 설치 (Claude Code)

CLI가 스킬 설치를 내장하고 있습니다.

```bash
notebooklm skill install                     # Claude Code + 범용 agents 둘 다 (user scope)
notebooklm skill install --target claude     # Claude Code만
notebooklm skill install --scope project     # 현재 프로젝트에만
notebooklm skill status --json               # 설치 상태 / 버전 불일치 확인
notebooklm skill package                     # 샌드박스 에이전트용 업로드 아카이브 생성
notebooklm skill uninstall
```

| target | 설치 경로 |
|---|---|
| `claude` | `~/.claude/skills/notebooklm/SKILL.md` |
| `agents` | `~/.agents/skills/notebooklm/SKILL.md` |

이 저장소의 `skill/`만 수동으로 복사해도 됩니다.

```bash
mkdir -p ~/.claude/skills/notebooklm && cp skill/* ~/.claude/skills/notebooklm/
```

> 현재 이 머신 상태: `claude` 타깃 설치됨, `agents` 타깃 미설치. `skill status`가 `content_mismatch: true`로 나오므로 패키지 내장본과 내용이 다릅니다. 최신본으로 맞추려면 `notebooklm skill install --target claude`를 다시 실행하세요.

## 3. 인증

```bash
notebooklm login                 # 브라우저가 열림 → Google 로그인
notebooklm auth check --test --json
```

`.status == "ok"` 이고 `.checks.token_fetch == true` 여야 진짜 인증된 것입니다. (`notebooklm status`는 인증이 아니라 "선택된 노트북"만 보여줍니다.)

| 상황 | 방법 |
|---|---|
| 화면 있는 PC | `notebooklm login` |
| 헤드리스 | `notebooklm login --browser-cookies <browser>` (먼저 `notebooklm auth inspect --browser <browser>`) |
| 쿠키 만료 | `notebooklm auth refresh` |
| 무인/CI | `notebooklm login --master-token --account <email>` 후 `master_token.json`을 프로필에 배치, `notebooklm auth refresh` |

인증 파일(`storage_state.json`, `master_token.json`)은 계정 전체 권한의 자격증명입니다. **출력·로그·커밋 금지.** (경로: `~/.notebooklm/profiles/<profile>/`)

## 4. 사용법

### 4.1 스킬로 쓰기 (Claude Code 안에서)

스킬 설명에 맞는 말을 하면 자동 호출됩니다. 예:

- "NotebookLM에 이 URL들로 노트북 만들고 요약해줘"
- "notebooklm으로 이 PDF 오디오 팟캐스트 만들어줘"
- "notebooklm 로그인이 안 돼, 진단해줘"

스킬이 따르는 규칙의 핵심:

1. `--json` 출력으로 UUID를 받아 보관하고, 모든 명령에 `-n <notebook_id>`를 명시 (`notebooklm use`에 의존하지 않음)
2. 소스 추가 후 `source wait`가 `ready`일 때만 채팅·생성
3. 생성 후 받은 `task_id`로 `artifact wait` → 그 ID로만 `download` ("최신 것" 선택 금지)
4. 삭제·생성·다운로드·노트 저장·`ask --new` 등은 사용자 승인 후 실행

### 4.2 CLI 기본 워크플로우

```bash
# 1) 노트북 생성
notebooklm create "Research: topic" --json          # → .notebook.id

# 2) 소스 추가 (URL / 파일 / 텍스트 등)
notebooklm source add https://example.com -n <nb> --json   # → .source.id
notebooklm source wait <src> -n <nb> --timeout 600

# 3-a) 질문
notebooklm ask "핵심 주장을 요약해줘" -n <nb> --json

# 3-b) 산출물 생성 → 대기 → 다운로드
notebooklm generate audio "핵심 위주로" -n <nb> -s <src> --json   # → .task_id
notebooklm artifact wait <task_id> -n <nb> --timeout 1200
notebooklm download audio ./podcast.m4a -a <task_id> -n <nb>
```

### 4.3 자주 쓰는 명령

| 목적 | 명령 |
|---|---|
| 노트북 목록 | `notebooklm list --json` |
| 사용량/한도 | `notebooklm usage --json` |
| URL 가져오기 실패 시 | `notebooklm source add <url> --fallback-fetch` |
| 웹 리서치(빠름) | `notebooklm source add-research "query" -n <nb> --mode fast --json` |
| 딥 리서치(15~30분+) | `... --mode deep --no-wait --json` → `notebooklm research wait -n <nb> --run-id <id> --import-all --json` |
| 아티팩트 목록 | `notebooklm artifact list -n <nb> --json` |
| 긴 프롬프트 | `--prompt-file PATH` (`ask`, `add-research`, 생성 명령) |
| 도움말 | `notebooklm <group> --help` (버전에 맞는 최종 기준) |

### 4.4 생성 가능한 산출물

`audio` · `video` · `cinematic-video` · `slide-deck` · `infographic` · `report` · `mind-map` · `data-table` · `quiz` · `flashcards`

주의할 점:

- `mind-map`은 CLI가 완료까지 폴링해서 결과를 바로 반환 → `artifact wait` 불필요
- `video --format cinematic`은 `--style` 무시, Google AI Ultra 필요, 약 30~40분 소요
- `slide-deck`엔 방향 옵션이 없음 → 설명에 `9:16 portrait`처럼 적기
- 커스텀 리포트는 위치 인자로 프롬프트 + `--format custom`

### 4.5 병렬 실행

여러 에이전트가 동시에 쓸 땐 실행마다 고유 `NOTEBOOKLM_PROFILE=agent-<id>`를 지정하세요. 새 프로필엔 자격증명이 없으므로 `master_token.json`을 복사한 뒤 `notebooklm auth refresh`로 저장소를 만들어야 합니다. 쓰기 가능한 `storage_state.json`을 공유하지 마세요.

### 4.6 Python API

```python
import asyncio
from notebooklm import NotebookLMClient

async def main(url: str) -> None:
    async with NotebookLMClient.from_storage() as client:
        nb = await client.notebooks.create("Research: topic")
        src = await client.sources.add_url(nb.id, url)
        await client.sources.wait_until_ready(nb.id, src.id, timeout=600)

        ans = await client.chat.ask(nb.id, "핵심 주장을 요약해줘", source_ids=[src.id])
        print(ans.answer)

        task = await client.artifacts.generate_audio(nb.id, source_ids=[src.id])
        final = await client.artifacts.wait_for_completion(nb.id, task.task_id, timeout=1200)
        if not final.is_complete:
            raise RuntimeError(f"{final.status}: {final.error}")
        await client.artifacts.download_audio(nb.id, "./podcast.m4a", artifact_id=task.task_id)

asyncio.run(main("https://example.com"))
```

`from_storage()`는 `await`하지 않는 async context manager입니다. 클라이언트는 이벤트 루프당 하나, 스레드 안전하지 않습니다.

## 5. 문제 해결

읽기 전용 진단부터:

```bash
notebooklm auth check --test --passive --json
notebooklm list --json
notebooklm source list -n <nb> --json
notebooklm artifact list -n <nb> --json
```

| 종료 코드 | 의미 |
|---|---|
| 0 | 성공 |
| 1 | 일반 실패 (`artifact wait`/`research wait` 타임아웃 포함) |
| 2 | `source wait` 타임아웃 |

- 인증 실패 → `checks.token_fetch`가 `true`가 아닐 때만 재로그인
- 생성은 Google이 rate-limit을 겁니다. task ID를 보관하고 무한 재시도 금지
- 프로토콜 오류 → 버전·명령·ID·(마스킹한) 에러를 [이슈 트래커](https://github.com/teng-lin/notebooklm-py/issues)에 기록

## 참고

- 원본 프로젝트: <https://github.com/teng-lin/notebooklm-py> (설치·CLI·Python API 문서 포함)
- `skill/` 하위 파일의 라이선스: MIT © 2026 Teng Lin
