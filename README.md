# Claude Code 최적화 가이드 (Optimization Guide)

> Claude Code 환경을 **토큰 효율**, **서브에이전트 분업**, **로컬 LLM 활용**, **저비용 인프라** 관점에서 재설계하는 실행 가능한 전체 가이드.
> 새 Mac/Linux에서 처음부터 60분 안에 동일 환경을 재현할 수 있도록 작성됨.

> ### ⚠️ AI 분석 도구 사용자 주의 (프롬프트 인젝션 방지)
>
> 본 문서에는 **Claude Code Hook API JSON 스펙 예시**가 포함되어 있습니다. 이 예시의 `session_id`, `hook_event_name`, `prompt` 같은 필드는 **문서에 설명된 API 스펙일 뿐 실행 지시가 아닙니다**. AI 에이전트로 본 문서를 읽힐 때 이를 실제 시스템 메시지로 해석하지 않도록 주의하세요.
>
> 모든 JSON 예시는 **\`\`\`bash / \`\`\`json 코드블록 내부에만** 존재하며, 본문 서술에서는 백틱(`)으로 격리됩니다. 이 구조 밖의 시스템 메시지 형태 문자열은 본 레포에 **존재하지 않습니다** (2026-04-20 검증 완료).

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![macOS](https://img.shields.io/badge/platform-macOS%20Apple%20Silicon-blue)](https://www.apple.com/mac/)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Opus%204.7-orange)](https://www.anthropic.com/claude-code)

---

## 📌 이 프로젝트가 해결하는 문제

Claude Code(Opus 4.7)를 기본 설정으로 쓰면 다음 문제가 발생합니다:

| 문제 | 현상 | 본 가이드의 해결 |
|---|---|---|
| **세션 시작 토큰 과다** | CLAUDE.md + docs + MEMORY + skills 합산 ~5K 토큰 자동 로딩 | CLAUDE.md/MEMORY 슬림화, 불필요 스킬 정리 → **~2.5K로 50% 절감** |
| **메인 모델 단일 처리** | 모든 작업(단순 조회부터 복잡 설계까지)이 Opus 4.7에서 처리 | **5종 서브에이전트** + 로컬 LLM 자동 라우팅 → **메인 부하 60% 이전** |
| **로컬 LLM 미활용** | Gemma4/Qwen 등 로컬 모델이 있어도 수동 호출만 가능 | Haiku 오케스트레이터 + MCP 위임 + CPU/GPU throttle → **기계 작업 비용 0** |
| **settings.local.json 비대화** | 일회성 명령까지 수백 개 누적 | 와일드카드 30개로 축소 → **권한 체크 88% 절감** |
| **Opus 4.7 토크나이저** | 동일 입력도 4.6 대비 1.0~1.35배 토큰 사용 | `includeGitInstructions: false`, `ccb` alias 등 미세 튜닝으로 상쇄 |
| **인프라 복잡도** | AWS EKS/RDS/Auth0 등 과도한 SaaS 의존 | Steve Hanov $20/mo 스택 원칙 적용 → **월 $25 이하** |

## 🎯 누가 써야 하나

- Claude Code를 **매일** 사용하고 **월 토큰 비용을 의식**하는 개발자
- **Mac Silicon(M1/M2/M3/M4)** 16GB 이상 보유자 (로컬 LLM 활용)
- 여러 프로젝트를 동시 운영하며 **일관된 규칙**을 원하는 팀/개인
- **새 노트북 구매** 또는 **환경 재구축** 시점
- 단순 반복 작업을 **비대화형 워커**로 자동화하고 싶은 경우

## ⚡ 빠른 시작 (5분 코어 세팅)

```bash
# 1. 본 레포 클론
git clone https://github.com/hs85-newbie/claude-code-optimization-guide.git
cd claude-code-optimization-guide

# 2. 사전 체크 실행
bash -c 'for cmd in git node pnpm python3 gh curl; do
  command -v $cmd >/dev/null && echo "✓ $cmd" || echo "✗ $cmd"
done'

# 3. GUIDE.md 열어서 부록 A부터 순차 실행
open GUIDE.md   # macOS
# code GUIDE.md # VS Code
```

> **핵심**: 본 README는 개요. 실제 실행 스크립트/설정 파일 전체는 `GUIDE.md`에 포함되어 있으며, 에이전트가 그대로 읽고 실행할 수 있도록 설계되었습니다.

## 📊 기대 효과 (적용 후)

| 항목 | 현재 (기본) | 적용 후 | 절감 |
|---|---|---|---|
| 세션 시작 토큰 | ~5K | ~2.5K | **50% ↓** |
| Bash 권한 체크 엔트리 | 255개 | 30개 | **88% ↓** |
| 메인 모델 처리 작업 | 100% | 40% | **60% 서브/로컬 위임** |
| 단순 작업 비용 (요약/번역/포매팅) | Opus 4.7 | 로컬 Gemma4 (무료) | **~100% ↓** |
| 반복 워커 모드 (ccb alias) | 기본 | `ccb` 활성화 | **60-80% ↓** |
| 인프라 월 비용 (2-3 서비스) | $100+ | $15-25 | **75% ↓** |
| **종합 월 비용** | **기준선** | **30-40%** | **60-70% ↓** |

## 📑 부속 문서

| 파일 | 용도 |
|---|---|
| [`GUIDE.md`](./GUIDE.md) | 본문 1-6장 + 부록 11종 — 전체 실행 가이드 |
| [`docs/NEXT-SESSION-TESTS.md`](./docs/NEXT-SESSION-TESTS.md) | 환경 적용 후 새 세션에서 실행할 검증 프롬프트 3종 |

## 📚 문서 구조 (GUIDE.md)

단일 파일 `GUIDE.md`에 모든 내용 포함. 부록 10개로 구성되어 있어 필요한 부분만 참조 가능.

```
GUIDE.md
├── 본문 1-6장 — 현재 구조 분석 + 개선 방향 (P1~P6)
├── 부록 A — 신규 PC/노트북 첫 세팅 가이드 (STEP 1~8)
├── 부록 B — 서브에이전트 5종 전체 본문 (plan/explore/coder/quick-lookup/local-ops)
├── 부록 C — 로컬 LLM 자동 라우팅 훅 (UserPromptSubmit)
├── 부록 D — MEMORY.md 프로젝트별 분리 템플릿
├── 부록 F — 하드웨어 자동 감지 + 모델 자동 선택 매트릭스
├── 부록 G — CPU/GPU 부하 기반 Throttle (사용자 작업 중 자동 대기)
├── 부록 H — Steve Hanov "$20/mo 스택" Mac Silicon 적용
├── 부록 I — 전체 문서 자가 점검 결과 (5차 검증 완료)
├── 부록 K — Claude Code 토큰 효율 미세 튜닝 (stdy.blog 기반)
└── 부록 J — 실행 순서 요약 체크리스트 (PRE + STEP 1~13)
```

## 🧩 각 부록 요약

### 부록 A — 신규 PC 첫 세팅 (STEP 1-8)

60분 내 동일 환경 재현. 각 STEP에 검증 명령 포함:

| STEP | 작업 | 결과물 |
|---|---|---|
| 1 | `my-claude-global` 클론 + CLAUDE.md 심링크 | 전역 규칙 적용 |
| 2 | gh 인증 + `~/.anthropic/env` 준비 | API 키 로드 |
| 3 | `~/.claude/settings.json` 배포 (토큰 효율 옵션 포함) | 30개 권한 + MCP 설정 |
| 4 | 서브에이전트 5종 배포 (부록 B) | `~/.claude/agents/*.md` |
| 5 | Skills 심링크 (선택) | session-archive 등 |
| 6 | 로컬 LLM 세팅 (부록 F 선행) | LM Studio + Gemma4 |
| 7 | 훅 배포 + hooks 자동 병합 | 라우팅 + Throttle |
| 8 | 최종 검증 체크리스트 | 환경 정상성 확인 |

### 부록 B — 서브에이전트 5종

| 에이전트 | 모델 | 트리거 | 용도 |
|---|---|---|---|
| `plan` | **opus** | "어떻게 구현", 2+ 파일 수정 | 설계/아키텍처 |
| `explore` | **sonnet** | "왜 동작", 3+ 검색 | 코드베이스 탐색 |
| `coder` | **sonnet** | 대상 파일 확정 | 코드 구현 |
| `quick-lookup` | **haiku** | 단일 파일 <200줄 | 단순 조회 |
| `local-ops` | **haiku** (오케스트레이터) → MCP `local-llm` | 요약/번역/포매팅 | 기계 작업 (무료) |

### 부록 C — UserPromptSubmit 훅

"요약해/번역해/포매팅" 등 키워드 감지 시 `local-ops` 사용 힌트 자동 출력. **stdin JSON 파싱** 방식으로 Claude Code 공식 훅 스펙 준수.

### 부록 F — 하드웨어 자동 감지

`sysctl` + `system_profiler`로 Mac Silicon 사양 자동 감지 → RAM 기준 최적 모델 추천:

| RAM | 권장 모델 |
|---|---|
| 8 GB | 로컬 LLM 비권장 (OpenRouter 폴백만) |
| 16 GB | Qwen3-4B / Gemma3-4B Q4 |
| **24 GB** | **Gemma-4-26B-A4B MoE Q4** ← 본 가이드 작성 기준 머신 |
| 32 GB | Qwen3-32B Q4 |
| 48 GB | Llama-3.3-70B Q3 |
| 64 GB+ | Llama-3.3-70B Q4/Q6 |
| 96 GB+ | Llama-3.3-70B Q8 (Claude Sonnet 근접) |

### 부록 G — CPU/GPU Throttle

사용자가 Xcode/영상 편집/빌드 등 무거운 작업 중이면 로컬 LLM 호출을 **자동 대기**시키고, 부하 해소 시 재개. 타임아웃(3분) 초과 시 클라우드 폴백.

- `top -l 1`로 CPU idle % 측정
- `sysctl vm.loadavg`로 부하 확인
- `memory_pressure`로 메모리 여유 확인
- **sudo 불필요** (보안 권한 낮음)

### 부록 H — Steve Hanov 원칙 (Mac Silicon 변환)

[원문](https://stevehanov.ca/blog/how-i-run-multiple-10k-mrr-companies-on-a-20month-tech-stack)을 Apple Silicon 환경에 맞게 재해석:

| Hanov 원칙 | Mac Silicon 적용 |
|---|---|
| SQLite WAL > 원격 Postgres | SQLite WAL 기본, Postgres는 멀티 인스턴스만 |
| RTX 3090 자가 보유 | Apple Silicon Unified Memory + LM Studio |
| Linode/DO $5 VPS | Railway $5 Hobby |
| Go 단일 바이너리 | `railway up` (git push 배포) |
| VS Code + Copilot | **Claude Code + VS Code** (Cursor 불필요) |

권장 스택 월 비용: **$6-10 (단일 서비스) / $15-25 (2-3 서비스)**

### 부록 K — 토큰 효율 미세 튜닝

[stdy.blog 원문](https://www.stdy.blog/increasing-token-efficiency-by-setting-adjustment-in-claude-and-codex/) 기반:

**settings.json 조정**:
```json
{
  "includeGitInstructions": false,
  "autoConnectIde": false,
  "attribution": { "commit": "", "pr": "" }
}
```

**`.zshrc` alias (워커 모드)**:
```bash
# ccb: 모든 자동 컨텍스트 제거, 필수 도구만 유지 (60-80% 절감)
alias ccb='ENABLE_CLAUDEAI_MCP_SERVERS=false CLAUDE_CODE_DISABLE_AUTO_MEMORY=1 CLAUDE_CODE_DISABLE_CLAUDE_MDS=1 CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=1 DISABLE_TELEMETRY=1 claude --tools "Bash,Edit,Glob,Grep,Read,Write" --disable-slash-commands --exclude-dynamic-system-prompt-sections'

alias ccby='ccb --dangerously-skip-permissions'
```

**사용 예**:
```bash
ccby -p "/tmp/logs/*.log 파일을 all.log로 합쳐"  # 워커 모드로 간단 작업
```

## 🏗️ 아키텍처 (최종 상태)

```
┌──────────────────────────────────────────────────────────────┐
│                     사용자 프롬프트                           │
└──────────────────────┬───────────────────────────────────────┘
                       │
           ┌───────────▼────────────┐
           │  UserPromptSubmit 훅   │  (부록 C, 요약/번역 키워드 감지)
           └───────────┬────────────┘
                       │
           ┌───────────▼────────────┐
           │   메인 모델 (Opus 4.7)  │  라우팅 판단 + 통합 응답
           └─┬────┬────┬────┬──────┘
             │    │    │    │
    ┌────────▼┐ ┌─▼──┐ ┌▼───┐ ┌▼─────────┐
    │ plan    │ │expl│ │coder│ │local-ops │
    │ (Opus)  │ │ore │ │    │ │(Haiku)   │
    └─────────┘ │Son)│ │Son)│ └────┬─────┘
                └────┘ └────┘      │
                                   │ MCP 위임
                                   ▼
                  ┌────────────────────────────┐
                  │  Throttle Gate (부록 G)    │  CPU/메모리 부하 체크
                  └─────────────┬──────────────┘
                                │
                  ┌─────────────▼──────────────┐
                  │  LM Studio (localhost:1234)│
                  │  Gemma-4-26B-A4B MoE       │  ← 로컬 무료
                  └────────────────────────────┘
```

## 📋 실행 순서 (부록 J 체크리스트)

신규 환경 적용 시 순차 진행:

```
[ ] PRE:     Preflight 체크 (필수 도구 + 디스크 + RAM + 기존 백업)    → 부록 I-3
[ ] STEP 1:  my-claude-global 클론 + CLAUDE.md 심링크               → 부록 A-STEP 1
[ ] STEP 2:  gh 인증 + ~/.anthropic/env 준비                         → 부록 A-STEP 2
[ ] STEP 3:  settings.json 배포 (토큰 효율 옵션 포함)                → 부록 A-STEP 3
[ ] STEP 4:  서브에이전트 5종 배포                                    → 부록 A-STEP 4 + B
[ ] STEP 5:  Skills 심링크 (선택)                                     → 부록 A-STEP 5
[ ] STEP 6:  로컬 LLM 세팅 (하드웨어 감지 → 모델 → 서버)              → 부록 A-STEP 6 + F
[ ] STEP 7:  라우팅 + Throttle 훅 배포 + hooks 자동 병합              → 부록 A-STEP 7 + C + G
[ ] STEP 8:  최종 검증 스크립트                                       → 부록 A-STEP 8
[ ] STEP 9:  "요약해줘" 테스트로 로컬 라우팅 확인                      → 부록 B-5 + C-3
[ ] STEP 10: MEMORY.md 프로젝트별 재구성 (기존 환경 이관 시)           → 부록 D
[ ] STEP 11: 신규 프로젝트 Hanov 체크리스트 적용                      → 부록 H-5
[ ] STEP 12: .zshrc에 ccb/ccby alias 추가                             → 부록 K-3/K-4
[ ] STEP 13: 상태 점검 프롬프트로 설정 효율성 확인                    → 부록 K-6
```

## 💻 하드웨어 요구사항

### 최소 (Claude Code만)
- macOS 13.0+ 또는 Linux
- RAM 8GB
- 디스크 10GB
- Python 3.11+, Node 20+

### 권장 (로컬 LLM 포함)
- **Apple Silicon M1 이상** (Unified Memory 활용)
- **RAM 24GB 이상** (Gemma-4-26B MoE Q4 로드 가능)
- 디스크 **30GB 이상**
- LM Studio 최신

### 최적 (프론티어 근접)
- **Apple Silicon M3 Pro/Max/Ultra**
- **RAM 64GB 이상** (Llama-3.3-70B Q4 가능)
- 디스크 **100GB 이상**

## 🔧 트러블슈팅 FAQ

<details>
<summary>Q. CLAUDE.md 내용이 세션에 적용 안 됨</summary>

심링크가 깨진 상태. 재생성:
```bash
ln -sf ~/my-claude-global/CLAUDE.md ~/.claude/CLAUDE.md
test -L ~/.claude/CLAUDE.md && echo "OK"
```
</details>

<details>
<summary>Q. 서브에이전트가 호출 안 됨</summary>

frontmatter 파싱 실패 가능성. 확인:
1. `---` 구분자가 파일 최상단과 YAML 끝에 있는지
2. `name`, `description`, `model` 필수 필드 존재
3. `model` 값은 `opus`, `sonnet`, `haiku`, `inherit` 중 하나
4. `claude doctor` 실행해서 에이전트 인식 여부 확인
</details>

<details>
<summary>Q. settings.json에 `$HOME` 리터럴이 남음</summary>

heredoc 따옴표 혼동. 수동 치환:
```bash
sed -i '' "s|\$HOME|$HOME|g" ~/.claude/settings.json
```
또는 heredoc 사용 시 `<<EOF` (따옴표 없음) 확인.
</details>

<details>
<summary>Q. 권한 프롬프트가 계속 뜸</summary>

자주 쓰는 명령을 콜론 와일드카드로 등록:
```json
"Bash(tool_name:*)"
```
주의: `Bash(tool *)` (스페이스) 스타일은 호환성 낮음.
</details>

<details>
<summary>Q. Gemma4 응답이 느림</summary>

GPU offload 조정 (부록 F-3):
```bash
# GPU 코어 수 × 1.25 값으로 재로드
lms load gemma-4-26b-a4b-it --gpu-offload 10 --context-length 16384
```
M2 8코어 → offload 10 / M3 Max 40코어 → offload 50
</details>

<details>
<summary>Q. 훅이 동작 안 함</summary>

Claude Code hook은 stdin으로 JSON을 받음 (환경변수 아님). 테스트:
```bash
echo '{"prompt":"이 로그 요약해줘"}' | ~/.claude/hooks/route-to-local.sh
```
"[라우팅 힌트] ..." 출력 기대. 출력 없으면 Python3 설치 여부 확인.
</details>

<details>
<summary>Q. 로컬 LLM이 응답하지 않음</summary>

LM Studio 서버 상태 확인:
```bash
curl -sf http://localhost:1234/v1/models
lms ps   # 로드된 모델 목록
```
서버가 꺼져 있으면 `lms server start --port 1234` 실행.
</details>

<details>
<summary>Q. macOS 26.x에서 top 명령 출력이 다름</summary>

`top -l 1 -n 0`의 "CPU usage" 포맷은 Darwin 전체 호환되지만, 필드 순서가 바뀔 수 있음. 수동 확인:
```bash
top -l 1 -n 0 | awk '/CPU usage/ {print}'
```
출력에 맞게 부록 G-2 스크립트의 `$7` 인덱스 조정.
</details>

<details>
<summary>Q. Throttle 게이트 훅(local-llm-gate.sh)이 "BUSY" 루프에 빠져 통과 안 함 ★ 실전 빈발</summary>

**증상**: `WAIT: BUSY ... mem_free=16%` 가 15초마다 반복되며 타임아웃까지 대기.

**원인**: 로컬 LLM(Gemma4 26B 등)이 LM Studio에 로드된 상태에서는 macOS 메모리 가용률이 구조적으로 8-15% 수준. 초기 임계값 20%는 절대 통과 불가. 또한 macOS 26.x의 `memory_pressure` 명령 출력 포맷이 변경되어 기존 파싱 실패.

**해결**: 가이드 v1.4에서 vm_stat 기반 계산 + 기본 임계 8%로 변경됨. 최신 스크립트 사용. 저사양/LLM 미사용 머신은 환경변수로 조정:
```bash
export LOCAL_LLM_MEM_FREE_THRESHOLD=5   # 매우 낮게 (5%)
```

**상세**: GUIDE.md 부록 L-1 참조.
</details>

## 📖 참고 자료

이 가이드는 다음 리소스를 종합하여 작성:

- **Claude Code 공식 문서**: https://docs.anthropic.com/claude-code
- **Claude 프롬프팅 베스트 프랙티스**: https://docs.anthropic.com/claude/docs/prompt-engineering
- **Steve Hanov의 $20/mo 스택**: https://stevehanov.ca/blog/how-i-run-multiple-10k-mrr-companies-on-a-20month-tech-stack (Mac Silicon 환경으로 재해석)
- **stdy.blog 토큰 효율 가이드**: https://www.stdy.blog/increasing-token-efficiency-by-setting-adjustment-in-claude-and-codex/ (한국어)
- **Claude Code 토큰 효율 Gist**: https://gist.github.com/spilist/c468cbf1ed0ffc91100f813aabdcd520
- **LM Studio**: https://lmstudio.ai/

## 🗺️ 로드맵

- [x] 초기 구조 감사 + 5차 점검 완료
- [x] Steve Hanov 원칙 Mac Silicon 적용
- [x] stdy.blog 토큰 효율 팁 통합
- [x] 하드웨어 자동 감지 매트릭스 (M1~M4)
- [x] CPU/GPU Throttle (sudo 없이)
- [ ] Windows/WSL 버전 추가
- [ ] Linux (Ubuntu/Fedora) 네이티브 가이드
- [ ] NVIDIA GPU 환경 (VLLM/Ollama) 보조 가이드
- [ ] 서브에이전트 성능 벤치마크 자동화
- [ ] Claude Agent SDK 버전 (TypeScript/Python)
- [ ] 월간 자동 환경 감사 GitHub Action

## 🤝 기여 방법

1. 이슈 등록 또는 개선안 제안 Issue 오픈
2. Fork → 브랜치 생성 (`feat/...`, `fix/...`)
3. 수정 후 PR 생성
4. PR 본문에 다음 포함:
   - 어떤 문제를 해결하는지
   - 어떤 환경에서 검증했는지 (macOS 버전, Mac 사양)
   - Before/After 수치 (가능하면)

기여 전에 `GUIDE.md`의 **부록 I (자가 점검 체크리스트)** 를 참고하여 본인의 변경이 5차 점검을 통과하는지 확인 부탁드립니다.

## 📝 변경 이력

| 날짜 | 변경 | 상세 |
|---|---|---|
| 2026-04-20 | v1.0 초안 릴리스 | 본문 1-6장 + 부록 A-D |
| 2026-04-20 | v1.1 확장 | 부록 F/G/H/I 추가, 하드웨어 감지, Hanov 원칙 |
| 2026-04-20 | v1.2 미세 튜닝 | 부록 K 추가 (stdy.blog 토큰 효율 팁) |
| 2026-04-20 | v1.3 검증 완료 | 5차 자가 점검, 스크립트 실행 테스트 통과, 오류 0건 |
| 2026-04-20 | v1.4 실전 적용 피드백 | 본 머신(M2/24GB) 적용 중 발견한 Throttle 게이트 영구 BUSY 이슈 수정 (vm_stat 기반 + 임계 8%). 부록 L 신설. |

## ⚠️ 면책

- 본 가이드는 **Claude Code 2.1.x + Opus 4.7** 기준으로 작성됨. 향후 버전에서 일부 설정 키/플래그가 변경될 수 있음.
- 로컬 LLM 모델명과 HuggingFace 경로는 커뮤니티 상태에 따라 변동됨. 부록 F-2 자동 감지 스크립트를 우선 실행 후 출력을 따르기 바람.
- `ccby` alias (권한 확인 생략)는 **신뢰 가능한 자동화 스크립트 내**에서만 사용 권장. 사람이 직접 대화하는 세션에서는 `ccb`만 사용.
- 설정 변경 전 **반드시 기존 환경 백업** (`cp -Rp ~/.claude ~/.claude.backup-$(date +%Y%m%d)`).

## 📄 라이선스

MIT License - 자유롭게 포크/수정/재배포 가능. 상업적 사용 가능.

## 👤 작성자

**hs85-newbie** — Claude Code 전역 환경 최적화 실험 중

- GitHub: [@hs85-newbie](https://github.com/hs85-newbie)
- 관련 레포: [my-claude-global](https://github.com/hs85-newbie/my-claude-global) (전역 규칙 + Paperclip 파이프라인)
- 관련 레포: [gemma4-bench](https://github.com/hs85-newbie/gemma4-bench) (로컬 LLM 하이브리드 오케스트레이션)

---

**이 문서만 읽고 60분 내 재현 불가능하다면 이슈를 남겨주세요. 가이드의 목표는 "에이전트가 읽고 그대로 실행 가능한 수준"입니다.**
