# 새 세션 검증 프롬프트 (Next Session Tests)

> **목적**: `claude-code-optimization-guide` v1.5 적용 직후, **새 Claude Code 세션**을 열어 서브에이전트 5종 + hooks + 로컬 LLM 라우팅이 실제로 동작하는지 검증.
> **작성 시점**: 2026-04-20 (본 머신 M2/24GB 환경)
> **사용 방법**: 새 터미널에서 `claude` 실행 후, 각 섹션의 프롬프트를 **그대로 복사하여 입력**.

---

## 0. 세션 시작 프롬프트

첫 입력으로 아래를 그대로 붙여넣어 세션 컨텍스트를 지정:

```
세션 이름: claude-code-optimization-guide 실제 검증

컨텍스트:
- 레포: https://github.com/hs85-newbie/claude-code-optimization-guide (v1.5)
- 로컬 클론: ~/claude-code-optimization-guide/
- 본 머신(M2/24GB) 적용 완료. Gemma-4-26B LM Studio 로드 중.
- 메모리 참조: project_claude_code_optimization_guide.md

이번 세션 목표: 서브에이전트 5종과 hooks의 실제 동작 검증.

아래 3가지를 순차 테스트하고, 이슈 발생 시 GUIDE.md 부록 L에 추가
(에러/원인/해결 형식) → 커밋 → 푸시 → 전체 점검 반복.
```

---

## 1. 테스트 1 — 서브에이전트 자동 인식

### 프롬프트

```
/agents 명령을 실행해서 플랜/익스플로어/코더/퀵룩업/로컬옵스 5종이
사용자 에이전트로 모두 인식되는지 확인해줘. 누락이나 형식 에러가 있으면
~/.claude/agents/*.md의 frontmatter 문제를 진단하고 GUIDE.md 부록 B
본문을 확인해서 수정안 제안.
```

### 기대 결과
- `plan`, `explore`, `coder`, `quick-lookup`, `local-ops` 5종 모두 "User Agents" 섹션에 표시
- 각 에이전트의 model 필드가 올바른 값 (`opus` / `sonnet` / `haiku`)으로 표시

### 실패 시 조치
- 에이전트 누락 → `~/.claude/agents/<이름>.md` 파일 존재 확인
- 파싱 에러 → YAML frontmatter `---` 구분자 및 들여쓰기 검증
- 잘못된 model → GUIDE.md 부록 B-1 ~ B-5 스펙과 비교 후 수정

---

## 2. 테스트 2 — 라우팅 훅 실제 발동

### 프롬프트 (그대로 입력)

```
이 로그 한 줄로 요약해줘:
[13:30] deploy start
[13:31] build ok
[13:32] cache miss
[13:33] deploy done 180s
```

### 기대 결과
- Claude 응답 **앞**에 다음 형식 블록이 자동으로 뜸:
  ```
  [라우팅 힌트] 기계적 변환 작업으로 판단됩니다.
  → local-ops 에이전트(Gemma-4-26B, 로컬 무료) 사용을 권장합니다.
  → LM Studio 서버: 활성
  ```
- "요약해"가 matcher 정규식에 매칭되어 `~/.claude/hooks/route-to-local.sh` 실행

### 실패 시 조치
- 힌트 미출력 → `~/.claude/settings.json`의 `hooks.UserPromptSubmit.matcher` 정규식 확인
- 실행 권한 → `chmod +x ~/.claude/hooks/route-to-local.sh`
- stdin 파싱 실패 → `python3 --version` 확인, 부록 C-1 스크립트와 로컬 비교
- 단독 실행 테스트:
  ```bash
  echo '{"prompt":"요약해줘"}' | ~/.claude/hooks/route-to-local.sh
  ```

---

## 3. 테스트 3 — local-ops 에이전트 직접 위임

### 프롬프트

```
local-ops 에이전트로 아래 JSON을 키 알파벳 순으로 정렬해줘:
{"charlie":3, "alpha":1, "bravo":2}
```

### 기대 결과
- Haiku(오케스트레이터)가 작업 판단 후 **실제 생성은 Gemma-4-26B에 위임**
- 위임 경로:
  1. `~/gemma4-bench/scripts/dispatch.sh` 호출, 또는
  2. `curl http://localhost:1234/v1/chat/completions` 직접 호출
- 결과: `{"alpha":1, "bravo":2, "charlie":3}` 반환

### 실패 시 조치
- Haiku가 직접 답함 → GUIDE.md 부록 B-5 `## 호출 방법` 규칙 재확인
- LM Studio 응답 없음 → `curl -sf http://localhost:1234/v1/models`로 서버 상태 체크
- 디스패처 부재 → `ls ~/gemma4-bench/scripts/dispatch.sh` 확인 → 필요 시 레포 clone

---

## 4. 이슈 발생 시 핸들링 루프 (자동 수행)

테스트 중 에러/예상과 다른 동작 발견 시 **별도 지시 없이도** 아래 루프 실행:

1. `~/claude-code-optimization-guide/GUIDE.md` 부록 L에 `L-N` 섹션 추가
   - **발생 단계** / **증상** / **근본 원인** / **해결** / **검증** / **교훈** / **변경 파일**
2. 필요 시 해당 부록 (A/B/C/G 등) 본문 수정
3. `README.md` 트러블슈팅 FAQ에도 항목 추가
4. 커밋 메시지: `fix: <한 줄 요약> (v1.N)` 형식으로 커밋 + push
5. **커밋 전 전체 스캔 재실행**
   - JSON 블록 유효성 (`json.loads`)
   - Bash 구문 (`bash -n`)
   - 부록 순서 (A → B → C → D → F → G → H → I → K → L → J)
   - `<system-reminder>` 0건 재확인
6. **동일 오류 2회 이상 반복 시 멈추고 보고** (사용자 글로벌 규칙)

---

## 5. 최종 보고 형식 (채팅)

각 테스트 결과를 아래 표로 채팅에 요약:

```
| 테스트 | 결과 | 이슈 | 조치 |
|---|---|---|---|
| /agents 인식 | ✓/✗ | ... | ... |
| 라우팅 훅 발동 | ✓/✗ | ... | ... |
| local-ops 위임 | ✓/✗ | ... | ... |
```

상세본은 GUIDE.md 부록 L에 기록, 채팅에는 **커밋 URL만 링크**.

---

## 6. 참고 — 현재 세션 마무리 상태

| 항목 | 상태 |
|---|---|
| 레포 버전 | v1.5 (`41b45b6`) |
| `~/.claude` 적용 | 완료 (백업: `~/.claude.backup-20260420-1333` 427MB) |
| 서브에이전트 5종 | 배포 완료 |
| hooks 2종 | 배포 완료 (stdin JSON 파싱, vm_stat throttle) |
| LM Studio | Gemma-4-26B-A4B 로드 중 (localhost:1234) |
| 해결된 에러 | #1 Throttle 게이트 영구 BUSY (부록 L-1) |

---

## 7. 복구 방법 (문제 발생 시)

설정 전체를 백업 시점으로 되돌리려면:

```bash
# 현재 상태 한번 더 백업
cp -Rp ~/.claude ~/.claude.broken-$(date +%Y%m%d-%H%M)

# 원복
rm -rf ~/.claude
cp -Rp ~/.claude.backup-20260420-1333 ~/.claude
```
