# Claude 환경 구조 감사 및 최적화 제안

**세션**: claude-env-optimization
**작성일**: 2026-04-20
**목표**: `~/.claude/` 하위 문서 구조 파악 → 토큰 최적화 / 서브에이전트 / 로컬 LLM 활용 관점의 개선안 도출

---

## 1. 현재 구조 통합도

### 1-1. 전체 계층

```
~/.claude/                              # Claude Code 런타임 디렉터리
├── CLAUDE.md → ~/my-claude-global/CLAUDE.md   # 심링크 (슬림 코어 137줄)
├── settings.json                       # MCP 서버 + 기본 권한 (39줄)
├── settings.local.json                 # 로컬 권한 ★ 262줄 (비대화)
├── agents/                             # ❌ 존재하지 않음
├── skills/                             # ★ 46개 (대부분 gstack 번들)
├── plans/                              # 8개 (과거 작업 플랜)
├── projects/                           # 프로젝트별 메모리 (24개 디렉터리)
│   ├── -Users-cjons/memory/            # 32 files (공통 메모리)
│   ├── -Users-cjons-Documents-tms/     # 5 files
│   ├── -Users-cjons-Documents-dev-macromalt/ # 5 files
│   ├── -Users-cjons-Documents-dev/     # 2 files
│   └── (worktree 잔재 15개 디렉터리)    # ⚠️ 사용 흔적 없음
├── session-env/ + sessions/            # 세션 상태 (31 + 9)
└── file-history/ + telemetry/ + ...    # 이력/텔레메트리

~/my-claude-global/                     # 단일 진실 소스 (GitHub 동기화)
├── CLAUDE.md                           # 137줄 (전역 규칙 코어)
├── docs/                               # 159줄 (7개 규칙 문서)
│   ├── comment-rules.md      (21줄)
│   ├── documentation-rules.md(13줄)
│   ├── git-rules.md          (11줄)
│   ├── design-guide.md       (35줄)
│   ├── error-handling.md     (24줄)
│   ├── api-standards.md      (28줄)
│   └── testing-standards.md  (27줄)
├── skills/                             # 2개 심링크 소스 (session-archive-*)
├── tools/                              # session-archive CLI (Python)
└── .github/workflows/                  # 9개 워크플로우 (paperclip-dispatch 등)
```

### 1-2. 세션 시작 시 자동 로딩되는 컨텍스트

| 항목 | 라인 수 | 로딩 시점 | 비고 |
|---|---|---|---|
| CLAUDE.md | 137 | 매 세션 | @import로 docs 7개 추가 로드 |
| docs/*.md (7개) | 159 | 매 세션 | CLAUDE.md가 @import |
| MEMORY.md | 52 | 매 세션 | 상위 200줄 제한 |
| settings.json | 39 | 매 세션 | MCP 서버 + 기본 권한 |
| settings.local.json | 262 | Bash 호출 시 참조 | ★ 과다 |
| **합계** | **~650줄** | | 실제 토큰 ≈ 5K |

### 1-3. MCP 서버 구성 (settings.json)

| 서버 | 용도 | 상태 |
|---|---|---|
| `openspace` | 외부 워크스페이스 스킬 디스커버리 | 활성 |
| `local-llm` | Gemma-4-26B (LM Studio, localhost:1234) | 활성, **자동 라우팅 없음** |

### 1-4. Skills 카테고리 (46개)

| 카테고리 | 개수 | 대표 |
|---|---|---|
| gstack 번들 (외부 도입) | ~32 | anti-slop, autoplan, browse, ship, qa, design-*, plan-*, codex, investigate |
| OpenSpace | 2 | delegate-task, skill-discovery |
| session-archive (내부) | 2 | session-archive-ingest, session-archive-stats |
| 필수 유틸 | ~10 | context-engineering, incremental-implementation, checkpoint, careful, guard |

---

## 2. 문제점 진단 (토큰·병목·로컬LLM 관점)

### 2-1. 토큰 낭비 요소

#### A. `settings.local.json` 비대화 (262줄, 255개 권한 엔트리)
- **현상**: 일회성 디버그 명령 (`Bash(kill 23474)`, `Bash(echo "EXIT:$?")`, `python3 -c "..."` 복합 명령 등)까지 누적
- **영향**: 매 Bash 호출 권한 체크 시 전량 파싱 → 권한 매칭 지연 + 파일 I/O
- **심각도**: ★★★★☆

#### B. MEMORY.md 인덱스 비대화 (52줄, 28개 항목)
- **현상**: 현재 작업 중이 아닌 프로젝트(ai-curation, gemma4-bench, tms-stt 등)까지 매 세션 로드
- **영향**: 세션 시작 시 관련성 낮은 context 로드
- **심각도**: ★★★☆☆

#### C. worktree 잔재 디렉터리 (`~/.claude/projects/` 내 15개)
- **현상**: `-Users-cjons-Documents-dev--claude-worktrees-*` 형태로 과거 worktree 메타데이터 잔재
- **영향**: 세션 디스커버리 지연, 디스크 사용
- **심각도**: ★★☆☆☆

### 2-2. 서브에이전트 / 하네스 병목

#### A. **`~/.claude/agents/` 디렉터리 부재** ★★★★★
- 내장 에이전트(Explore, Plan, general-purpose)만 사용 가능
- "분석/설계/코딩/기본탐색/단순조회" 구분이 불가능한 구조
- 결과: **모든 판단과 실행이 메인 모델(Opus 4.7) 컨텍스트에서 처리**되어 토큰 소비 최대화

#### B. Skills 과다로 인한 선택 피로
- 46개 Skills description이 매 세션 로드 → 시스템 프롬프트 증가
- gstack 번들 중 실사용 빈도 낮은 스킬 다수 (design-shotgun, office-hours, retro 등)

#### C. 하네스 vs 멀티에이전트 선택 규칙이 CLAUDE.md에 있으나 **자동 적용 장치 없음**
- 규칙은 존재: "멀티에이전트 = 독립 서브태스크 2개↑"
- 그러나 description 레벨 트리거 없음 → 사용자 명시 요청에 의존

### 2-3. 로컬 LLM 미활용

#### A. MCP 서버 `local-llm` 연결되어 있으나 **자동 라우팅 없음**
- Gemma-4-26B가 LM Studio에 상시 로드되어 있음
- 현재는 사용자가 명시적으로 "로컬 LLM으로 해줘"라고 해야만 사용
- **단순 요약/포매팅/번역 등 Haiku 이하 작업을 로컬에서 처리 가능하지만 미활용**

#### B. `gemma4-bench` 디스패처 스크립트 존재하나 통합 안 됨
- `~/gemma4-bench/scripts/dispatch.sh`로 라우팅 가능 (router.yaml)
- Claude Code 에이전트에서 호출 연동 미구현

---

## 3. 개선안 (우선순위 순)

### P1. `~/.claude/agents/` 5종 에이전트 신설 (최우선)

#### 설계 원칙
- description 필드 첫 줄 = **트리거 명세** (Use PROACTIVELY when..., NOT for...)
- 수치 기준 (파일 수, 쿼리 수, 라인 수) 명시 → 판단 일관성
- 모델 계층: Opus(전략) → Sonnet(코딩/탐색) → Haiku(단순·로컬 오케스트레이션)

#### 5종 구조

| 에이전트 | 트리거 | 모델 | 금지 |
|---|---|---|---|
| `plan` | "어떻게 구현", 2+ 파일 수정 예상, 설계 리뷰 | opus | 파일 수정 |
| `explore` | "왜 동작", "어디서 사용", 3+ 검색 필요 | sonnet | 파일 수정 |
| `coder` | 대상 파일 확정 + 변경 범위 명확 | sonnet | 설계 결정 |
| `quick-lookup` | 단일 파일 < 200줄, 경로 명시, 사실 확인 | haiku | 다중 파일 검색 |
| `local-ops` | 요약·포매팅·번역 등 기계 작업 | haiku (오케스트레이터) → MCP `local-llm` 위임 | 판단·코드 수정 |

> **전체 frontmatter/본문**: 부록 B-1 ~ B-5 참조 (본 섹션과 중복 방지 위해 단일 진실 소스는 부록).

### P2. `settings.local.json` 대청소

#### 조치
- 255개 → **와일드카드 규칙 30개 이내**로 축소
- 일회성 명령(`Bash(kill 23474)`, `Bash(echo "EXIT:$?")` 등) 삭제
- 반복 패턴은 `Bash(python3 -c:*)` 형태 와일드카드로 통합
- `Bash(awk:*)`, `Bash(find:*)`, `Bash(grep:*)` 같은 범용 명령은 `settings.json`으로 이관하여 전역화

> **전체 settings.json**: 부록 A-STEP 3 참조 (콜론 문법 통일 + mkdir/cp/mv/rm 포함 전체 30여 항목).

### P3. MEMORY.md 프로젝트별 분리 로딩

#### 현 문제
전역 MEMORY.md에 28개 프로젝트 메모리 모두 인덱싱 → 세션마다 전량 로드

#### 개선
- 공통(user_profile, feedback_*) 만 전역 MEMORY.md에 유지 (10줄 이내)
- 프로젝트별 메모리는 해당 프로젝트 디렉터리의 `projects/-Users-cjons-Documents-{프로젝트}/memory/MEMORY.md`로 이관
- Claude Code는 현재 cwd 기반으로 자동 분리 로딩 (이미 지원됨)

### P4. Skills 정리

#### 조치
- **사용 빈도 분석**: `~/.claude/telemetry/` 데이터로 최근 30일 호출 이력 확인
- **비활성화 대상 후보** (사용 빈도 낮을 가능성):
  - `office-hours`, `design-shotgun`, `design-consultation`, `retro`, `canary`, `benchmark`, `plan-devex-review`, `devex-review`, `pair-agent`, `open-gstack-browser`, `connect-chrome`
- **유지 확정**: `investigate`, `ship`, `review`, `codex`, `checkpoint`, `anti-slop`, `context-engineering`, `delegate-task`, `skill-discovery`, `session-archive-*`, `careful`, `guard`
- **Skills 디렉터리 심볼릭 링크**로 교체: 로컬 설치 대신 `~/my-claude-global/skills/` 또는 원본 레포에서 link → Git 동기화

### P5. 로컬 LLM 자동 라우팅 통합

#### 방식
1. `local-ops` 에이전트 (P1에 포함) - 메인 모델이 description 매칭으로 자동 위임
2. Hook 활용: 특정 명령 패턴(예: "요약해", "번역해") 감지 시 `local-ops` 사용 힌트 출력
3. 시스템 부하 기반 throttle (사용자 무거운 작업 중 자동 대기)

> **전체 hooks 구조 및 스크립트**: 부록 C (라우팅 힌트) + 부록 G (throttle) 참조.

### P6. worktree 잔재 정리

```bash
# 미사용 worktree 디렉터리 삭제 (백업 후)
cp -Rp ~/.claude/projects ~/.claude/backups/projects-$(date +%Y%m%d)
ls ~/.claude/projects/ | grep "claude-worktrees" | xargs -I{} rm -rf ~/.claude/projects/{}
```

---

## 4. 효과 추정 (적용 후)

| 항목 | 현재 | 적용 후 | 절감 |
|---|---|---|---|
| 세션 시작 토큰 | ~5K | ~2.5K | 50% ↓ |
| Bash 권한 체크 부하 | 255 엔트리 | 30 엔트리 | 88% ↓ |
| 메인 모델 처리 작업 | 100% | 40% (60%는 서브/로컬 위임) | 60% ↓ |
| 단순 작업 비용 | Opus 4.7 | 로컬 무료 | ≈100% ↓ |
| **종합 월 비용 추정** | 기준선 | **30-40%** | **60-70% ↓** |

---

## 5. 적용 순서 (단계별)

| 단계 | 작업 | 소요 | 리스크 |
|---|---|---|---|
| 1 | `~/.claude/agents/` 5종 md 작성 | 30분 | 낮음 |
| 2 | settings.local.json 백업 후 슬림화 | 20분 | 중 (권한 누락 시 프롬프트 증가) |
| 3 | MEMORY.md 프로젝트별 분리 | 40분 | 낮음 |
| 4 | Skills 사용 빈도 조사 후 정리 | 60분 | 낮음 |
| 5 | 로컬 LLM 라우팅 훅 추가 | 30분 | 중 (hook 오발동 테스트 필요) |
| 6 | worktree 잔재 정리 | 10분 | 낮음 (백업 필수) |

---

## 6. 후속 작업

- [ ] 에이전트 md 5종 템플릿 합의 후 생성
- [ ] settings.local.json 슬림화 안 검토
- [ ] 로컬 LLM 라우팅 훅 PoC 테스트 (gemma4-bench 디스패처 활용)
- [ ] Skills 텔레메트리 분석 (`~/.claude/telemetry/` 구조 파악 필요)
- [ ] 매월 1회 환경 감사 자동화 (monthly-review.yml에 통합)

---

## 검증 기준

- 세션 시작 시 로드 라인 수 ≤ 320줄 (현재 ~650줄)
- 단일 "요약해줘" 요청이 자동으로 local-llm 에이전트로 위임되는지 확인
- 복잡도 중간 코딩 요청이 `coder` 에이전트(Sonnet)로 위임되는지 확인
- 메인 모델 직접 처리는 "라우팅 판단 + 통합 응답"에만 제한됨을 텔레메트리로 검증

---

# 부록 A. 신규 PC/노트북 첫 세팅 가이드

> **대상**: 새 Mac(또는 Linux)에서 처음부터 동일 환경을 구축하는 경우
> **전제**: Claude Code CLI가 이미 설치되어 있음 (`brew install anthropic/tap/claude-code` 또는 공식 인스톨러)
> **총 소요**: 약 60분 (네트워크 속도에 따라 변동)

## A-1. 사전 요구 사항

| 도구 | 확인 명령 | 없을 시 설치 |
|---|---|---|
| git | `git --version` | `brew install git` |
| node (≥ 20) | `node --version` | `brew install node` |
| pnpm | `pnpm --version` | `npm i -g pnpm` |
| bun | `bun --version` | `curl -fsSL https://bun.sh/install \| bash` |
| python3 (≥ 3.11) | `python3 --version` | `brew install python@3.12` |
| gh CLI | `gh --version` | `brew install gh` |
| LM Studio (선택, 로컬 LLM용) | `lms --version` | https://lmstudio.ai/ 에서 다운로드 |

## A-2. 부트스트랩 스크립트 (에이전트용 순차 실행)

아래 명령을 **순서대로** 실행. 각 블록이 독립적이므로 실패 시 해당 블록부터 재시작 가능.

### [STEP 1] 전역 규칙 레포 클론 + 심링크

```bash
# 1-1. my-claude-global 클론 (단일 진실 소스)
cd ~
git clone https://github.com/hs85-newbie/my-claude-global.git
cd my-claude-global && git pull

# 1-2. Claude Code 디렉터리 초기화
mkdir -p ~/.claude/{agents,skills,backups}
mkdir -p ~/docs/reports

# 1-3. CLAUDE.md 심링크 (전역 규칙 적용)
ln -sf ~/my-claude-global/CLAUDE.md ~/.claude/CLAUDE.md

# 검증
test -L ~/.claude/CLAUDE.md && echo "OK: CLAUDE.md 심링크" || echo "FAIL"
head -5 ~/.claude/CLAUDE.md
```

### [STEP 2] GitHub 인증 + 전역 시크릿 로드

```bash
# 2-1. gh 인증 (브라우저 열림 — 사용자 개입 필요)
gh auth login

# 2-2. Anthropic API 키 파일 준비 (수동 — 비밀번호 매니저에서 복사)
mkdir -p ~/.anthropic
# 아래 파일을 직접 편집:
# vi ~/.anthropic/env
# 내용: ANTHROPIC_API_KEY=sk-ant-...
chmod 600 ~/.anthropic/env 2>/dev/null || true
```

### [STEP 3] MCP 서버 `settings.json` 배포

아래 내용을 `~/.claude/settings.json`에 **그대로** 작성:

```bash
# heredoc `<<EOF` (따옴표 없음)을 사용하여 $HOME을 쉘이 자동 확장
cat > ~/.claude/settings.json <<EOF
{
  "includeGitInstructions": false,
  "autoConnectIde": false,
  "attribution": {
    "commit": "",
    "pr": ""
  },
  "permissions": {
    "allow": [
      "Bash(git:*)",
      "Bash(pnpm:*)",
      "Bash(bun:*)",
      "Bash(npm:*)",
      "Bash(npx:*)",
      "Bash(node:*)",
      "Bash(python3:*)",
      "Bash(pip:*)",
      "Bash(pip3:*)",
      "Bash(ls:*)",
      "Bash(grep:*)",
      "Bash(find:*)",
      "Bash(cat:*)",
      "Bash(head:*)",
      "Bash(tail:*)",
      "Bash(mkdir:*)",
      "Bash(cp:*)",
      "Bash(mv:*)",
      "Bash(rm:*)",
      "Bash(ln:*)",
      "Bash(chmod:*)",
      "Bash(test:*)",
      "Bash(gh:*)",
      "Bash(docker:*)",
      "Bash(psql:*)",
      "Bash(railway:*)",
      "Bash(lms:*)",
      "Bash(curl:*)",
      "Bash(ffmpeg:*)",
      "Bash(ffprobe:*)",
      "Bash(make:*)",
      "Bash(sysctl:*)",
      "Bash(system_profiler:*)",
      "WebFetch(domain:github.com)",
      "WebFetch(domain:docs.anthropic.com)",
      "WebFetch(domain:developers.openai.com)",
      "WebFetch(domain:lmstudio.ai)",
      "WebFetch(domain:huggingface.co)",
      "WebSearch"
    ]
  },
  "extraKnownMarketplaces": {
    "claude-plugins-official": {
      "source": {
        "source": "github",
        "repo": "anthropics/claude-plugins-official"
      }
    }
  },
  "mcpServers": {
    "local-llm": {
      "command": "node",
      "args": ["${HOME}/gemma4-bench/scripts/mcp-local-llm.mjs"],
      "env": {
        "LM_STUDIO_URL": "http://localhost:1234/v1",
        "LOCAL_LLM_MODEL": "gemma-4-26b-a4b-it"
      }
    }
  }
}
EOF

# 검증 (JSON 구조 + 실제 경로 확장 확인)
python3 -m json.tool ~/.claude/settings.json > /dev/null && echo "OK: settings.json 유효"
grep -q "/Users/" ~/.claude/settings.json && echo "OK: \$HOME 확장됨" || echo "FAIL: \$HOME 미확장"
```

> **참고**: `local-llm` MCP는 `~/gemma4-bench/` 레포가 필요. STEP 6 완료 후 자동 활성.

### [STEP 4] 서브에이전트 5종 배포 (부록 B에 전체 본문)

```bash
# 부록 B의 각 md 파일을 ~/.claude/agents/ 에 작성 (아래 명령 5회 반복)
# cat > ~/.claude/agents/plan.md <<'EOF' ... EOF
# cat > ~/.claude/agents/explore.md <<'EOF' ... EOF
# cat > ~/.claude/agents/coder.md <<'EOF' ... EOF
# cat > ~/.claude/agents/quick-lookup.md <<'EOF' ... EOF
# cat > ~/.claude/agents/local-ops.md <<'EOF' ... EOF

# 검증 (5개 파일 존재 확인)
ls ~/.claude/agents/*.md | wc -l  # 기대값: 5
```

### [STEP 5] 필수 Skills 심링크 (선택)

```bash
# my-claude-global 내부 스킬 2종 링크
ln -sf ~/my-claude-global/skills/session-archive-ingest ~/.claude/skills/session-archive-ingest
ln -sf ~/my-claude-global/skills/session-archive-stats ~/.claude/skills/session-archive-stats

# OpenSpace 스킬 (선택 — OpenSpace 사용 시)
# git clone https://github.com/openspace/openspace ~/openspace
# cp -r ~/openspace/openspace/host_skills/delegate-task ~/.claude/skills/
# cp -r ~/openspace/openspace/host_skills/skill-discovery ~/.claude/skills/
```

### [STEP 6] 로컬 LLM 세팅 (선택 — **부록 F의 하드웨어 감지 스크립트로 RAM 기준 모델 결정**)

> RAM 16GB 이상이면 로컬 LLM 적용 가능 (부록 F-1 매트릭스 기준). 본 머신(M2/24GB)은 Gemma-4-26B-A4B(MoE) 최적.

```bash
# 6-1. 부록 F-2 하드웨어 감지 스크립트 배포 (선행 필수)
mkdir -p ~/.claude/setup
# 부록 F-2의 detect-hardware.sh 전문을 ~/.claude/setup/detect-hardware.sh로 저장
chmod +x ~/.claude/setup/detect-hardware.sh

# 6-2. 하드웨어 감지 실행 → 권장 모델 확인
~/.claude/setup/detect-hardware.sh
# 출력된 권장 모델명과 GPU offload 값 메모 → 아래에서 사용

# 6-3. gemma4-bench 레포 클론 (로컬 LLM 디스패처 및 MCP 스크립트 포함)
cd ~
git clone https://github.com/hs85-newbie/gemma4-bench.git
cd gemma4-bench && (pnpm install 2>/dev/null || npm install)

# 6-4. LM Studio CLI 심링크 (LM Studio.app 설치 전제)
mkdir -p ~/bin
ln -sf "/Applications/LM Studio.app/Contents/Resources/app/.webpack/lms" ~/bin/lms
# PATH에 ~/bin 추가 (이미 있으면 생략)
grep -q 'HOME/bin' ~/.zshrc 2>/dev/null || echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
export PATH="$HOME/bin:$PATH"

# 6-5. LM Studio 서버 기동 (백그라운드)
open -g -a "LM Studio"
sleep 3
lms server start --port 1234

# 6-6. 추천 모델 다운로드 및 로드 (STEP 6-2 출력 참조)
# 예시 (본 머신 M2/24GB):
# lms get lmstudio-community/gemma-4-26b-a4b-it
# lms load gemma-4-26b-a4b-it --gpu-offload 10 --context-length 16384
#
# 실제로는 STEP 6-2 스크립트가 출력하는 명령어를 그대로 복사 실행.

# 검증
curl -s http://localhost:1234/v1/models | python3 -m json.tool | head -20
```

### [STEP 7] 로컬 LLM 자동 라우팅 훅 + Throttle 훅 (선택)

```bash
# 7-1. 훅 디렉터리
mkdir -p ~/.claude/hooks

# 7-2. 라우팅 힌트 훅 (부록 C-1 전문을 이 경로에 저장)
# ~/.claude/hooks/route-to-local.sh
chmod +x ~/.claude/hooks/route-to-local.sh

# 7-3. Throttle 게이트 훅 (부록 G-2 전문을 이 경로에 저장)
# ~/.claude/hooks/local-llm-gate.sh
chmod +x ~/.claude/hooks/local-llm-gate.sh

# 7-4. settings.json에 hooks 섹션 자동 병합 (수동 편집 대체)
python3 <<'PYEOF'
import json, os
path = os.path.expanduser("~/.claude/settings.json")
with open(path) as f:
    cfg = json.load(f)
cfg["hooks"] = {
    "UserPromptSubmit": [
        {
            "matcher": "(요약해|번역해|포매팅|포맷팅|정렬해|변환해|추출해)",
            "hooks": [
                {"type": "command", "command": os.path.expanduser("~/.claude/hooks/route-to-local.sh")}
            ]
        }
    ]
}
with open(path, "w") as f:
    json.dump(cfg, f, indent=2, ensure_ascii=False)
print("✓ hooks 섹션 병합 완료")
PYEOF

# 7-5. 훅 검증 (stdin JSON 주입)
echo '{"session_id":"test","hook_event_name":"UserPromptSubmit","prompt":"이 로그 요약해줘","cwd":"/tmp"}' \
  | ~/.claude/hooks/route-to-local.sh
```

### [STEP 8] 최종 검증

```bash
# 체크리스트 스크립트
cat <<'EOF' | bash
echo "=== Claude 환경 검증 ==="
test -L ~/.claude/CLAUDE.md && echo "✓ CLAUDE.md 심링크" || echo "✗ CLAUDE.md 누락"
test -f ~/.claude/settings.json && echo "✓ settings.json" || echo "✗ settings.json 누락"
count=$(ls ~/.claude/agents/*.md 2>/dev/null | wc -l | tr -d ' ')
[ "$count" = "5" ] && echo "✓ agents 5종" || echo "✗ agents 불완전 ($count/5)"
curl -sf http://localhost:1234/v1/models > /dev/null && echo "✓ LM Studio 서버 응답" || echo "- LM Studio 미기동 (선택)"
echo "=== 완료 ==="
EOF

# Claude 세션 시작 테스트
claude --version
```

## A-3. 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| CLAUDE.md 내용 안 보임 | 심링크 깨짐 | `ln -sf ~/my-claude-global/CLAUDE.md ~/.claude/CLAUDE.md` 재실행 |
| 서브에이전트 호출 안 됨 | frontmatter 파싱 실패 | `---` 구분자 + YAML 들여쓰기 점검, `claude doctor` 실행 |
| local-llm MCP 기동 실패 | `~/gemma4-bench/` 레포 미존재 | STEP 6-1 먼저 실행, 또는 `settings.json`에서 MCP 항목 임시 제거 |
| settings.json에 `$HOME` 리터럴 남음 | heredoc 따옴표 혼동 | `<<EOF`(따옴표 없음) 사용 확인, 또는 `sed -i '' "s\|\\$HOME\|$HOME\|g" ~/.claude/settings.json` |
| 권한 프롬프트 많이 뜸 | settings.json 권한 부족 | 자주 쓰는 명령을 `allow`에 콜론 와일드카드(`Bash(tool:*)`)로 추가 |
| Gemma4 응답 느림 | GPU offload 설정 낮음 | 부록 F-3 권장값(GPU 코어 × 1.25)으로 `lms load --gpu-offload N` 재실행 |
| 훅이 힌트 출력 안 함 | stdin 입력 파싱 실패 | 부록 C-3 테스트로 JSON 주입 확인, `python3` 존재 여부 점검 |

---

# 부록 B. 서브에이전트 5종 전체 본문 (즉시 배포용)

> 아래 내용을 각각 `~/.claude/agents/<이름>.md` 파일로 **그대로** 저장.
> 각 파일은 YAML frontmatter + 본문 구조. `---` 구분자 및 줄 끝 공백 주의.

## B-1. `~/.claude/agents/plan.md` — 설계 전용 (Opus)

```markdown
---
name: plan
description: Use PROACTIVELY when user asks "어떻게 구현", "설계 리뷰", "아키텍처 검토", or when the task touches 2+ files with unclear approach. NOT for bug fixes with a known cause, single-file edits, or simple renames. Returns step-by-step plan with file list, trade-offs, and test criteria.
model: opus
tools: Glob, Grep, Read, WebFetch, WebSearch
---

당신은 구현 계획 전담 에이전트입니다. 코드를 **직접 수정하지 않습니다**.

## 역할
- 요구사항 분석 → 영향 파일 목록 작성 → 단계별 구현 순서 도출
- 2개 이상 대안이 있으면 트레이드오프 비교표 제시
- 구현 전 검증 기준(테스트/성공 조건)을 먼저 정의

## 출력 형식
1. **목표 (1문장)**
2. **영향 파일** (경로:역할)
3. **단계** (각 단계 = 논리적 커밋 1개 단위)
4. **트레이드오프** (선택 시)
5. **검증 기준** (테스트 or 수동 확인 조건)
6. **리스크** (있을 경우)

## 금지
- 파일 생성/수정/삭제
- 테스트 실행 (계획만)
- 외부 API 호출 (조사용 WebFetch 제외)

## 규모 가이드
- 파일 3개 이하 → 간략 플랜 (100자 이내)
- 파일 4-10개 → 표준 플랜
- 파일 11개 이상 → 단계 분할 제안 + 증분 배포 원칙 적용
```

## B-2. `~/.claude/agents/explore.md` — 탐색 전용 (Sonnet)

```markdown
---
name: explore
description: Use when user asks "왜 이렇게 동작", "어디서 사용됨", "이 함수 호출처", or when search needs 3+ queries across unknown locations. Use PROACTIVELY for "코드베이스 분석" requests. NOT when the file path is already specified (use quick-lookup instead) or when the task requires modifying code.
model: sonnet
tools: Glob, Grep, Read, WebFetch
---

당신은 코드베이스 탐색 전담 에이전트입니다. **파일을 수정하지 않습니다**.

## 역할
- 키워드/패턴으로 관련 파일 식별
- 심볼 정의/사용처 추적
- 아키텍처 흐름 설명

## 탐색 전략
1. Glob으로 파일 후보 범위 축소 (`**/*.ts`, `src/**/*.tsx` 등)
2. Grep으로 키워드 hit 수집 (패턴 2-3회 조정)
3. Read로 핵심 파일 원문 확인 (한 번에 최대 300줄)
4. 발견을 `경로:라인` 형식으로 보고

## 출력 형식
- **발견 요약** (3-5 bullet)
- **핵심 경로** (`파일:라인 — 역할` 목록)
- **흐름 다이어그램** (필요 시 텍스트 ASCII)
- **후속 조사 제안** (불확실한 지점)

## 금지
- Edit/Write/NotebookEdit (모두 차단)
- 파일 수정 제안 (탐색 보고만)

## 효율 원칙
- 동일 파일 재읽기 금지 — 첫 Read에 범위 충분히 확보
- Grep `-A/-B/-C` 컨텍스트로 추가 Read 최소화
- 결론 명확하면 추가 탐색 생략
```

## B-3. `~/.claude/agents/coder.md` — 코딩 실행 (Sonnet)

```markdown
---
name: coder
description: Use when target files are identified AND change scope is clear (plan exists or task is simple). Triggers on "이 파일 수정", "X 함수 구현", "버그 수정" with known cause. NOT for architectural decisions (use plan), codebase-wide search (use explore), or trivial one-liners already clear to the main session.
model: sonnet
tools: Read, Edit, Write, Bash, Glob, Grep
---

당신은 코드 구현 전담 에이전트입니다. **설계 결정을 하지 않습니다** — 받은 계획을 충실히 수행.

## 역할
- 지정된 파일의 변경 사항 구현
- 타입 체크 / 린트 / 테스트 실행으로 검증
- 실패 시 원인 파악 후 수정 (최대 2회 재시도)

## 작업 흐름
1. 대상 파일 Read
2. Edit/Write로 변경 적용
3. 프로젝트 기본 검증 실행:
   - TypeScript: `pnpm tsc --noEmit` or `npx tsc --noEmit`
   - Lint: `pnpm lint` or `eslint .`
   - Test: `pnpm test` (변경 관련 파일만)
4. 통과 시 결과 보고 / 실패 시 수정 반복

## 출력 형식
- **변경 파일** (경로 + 한 줄 요약)
- **검증 결과** (통과/실패 + 로그 핵심)
- **후속 필요 사항** (리팩터링 제안, 문서 갱신 등)

## 금지
- 설계 변경 (범위 이탈 시 플랜 업데이트 요청 후 중단)
- 파일 4개 이상 변경 시 단일 커밋으로 묶기 (반드시 논리적 단위로 분할)
- `git push` / PR 생성 (반드시 사용자 승인)

## 커밋 규칙
- 논리적 단위 1개 = 커밋 1개
- 메시지: `feat/fix/chore/refactor/docs: 한국어 요약`
- 사용자 승인 없으면 커밋만, push 금지
```

## B-4. `~/.claude/agents/quick-lookup.md` — 단순 조회 (Haiku)

```markdown
---
name: quick-lookup
description: Use PROACTIVELY for single-file reads under 200 lines, known-path content retrieval, simple fact checks ("이 함수 시그니처가 뭐야", "이 파일 수정일이 언제야"), and single-keyword Grep. NOT for multi-file search (use explore), not for any code modification.
model: haiku
tools: Read, Glob, Grep
---

당신은 단순 조회 전담 에이전트입니다. **해석을 최소화**하고 원문 우선으로 반환합니다.

## 대응 범위
- 경로가 명시된 단일 파일 읽기 (≤ 200줄)
- 단일 키워드 Grep (1회로 종결)
- 특정 함수/심볼 시그니처 확인
- 파일 존재 여부 / 메타데이터 확인

## 출력 형식
- **결과**: 원문 또는 매칭 라인
- **경로**: `파일:라인` (있을 경우)
- **추가 조사 필요**: "범위 초과 — explore 에이전트 권장" (있을 경우)

## 금지
- 2회 이상 탐색 시도 (즉시 중단하고 explore 권장)
- 코드 해석/요약 (원문 우선)
- 파일 수정

## 효율 원칙
- 첫 시도로 답이 안 나오면 바로 중단 + 상위 에이전트 제안
- 긴 파일 (> 200줄)은 읽지 않고 범위 재지정 요청
```

## B-5. `~/.claude/agents/local-ops.md` — 로컬 LLM 기계 작업 (Haiku 오케스트레이터)

> **수정 이력 (2026-04-20)**: `model: inherit` → `model: haiku`로 변경.
> Claude Code 서브에이전트 `model` 필드는 Claude 모델만 지원합니다. 이 에이전트는 **Haiku(최저 비용)가 오케스트레이터로 실행되며, 실제 생성 작업은 `local-llm` MCP 서버(Gemma-4-26B, 로컬 무료)에 위임**합니다.

```markdown
---
name: local-ops
description: Use PROACTIVELY for text-only mechanical tasks with NO judgment required — summarization of provided text, JSON/YAML reformatting, Korean↔English translation, list extraction from logs, template filling, markdown table conversion. NOT for code modification, architecture decisions, debugging, or anything requiring understanding of context beyond the literal input.
model: haiku
tools: Read, Bash, mcp__local-llm__*
---

당신은 로컬 LLM 오케스트레이터입니다. **당신(Haiku)은 판단만** 하고, 실제 생성은 `local-llm` MCP 툴(LM Studio + Gemma-4-26B)로 위임합니다.

> **실행 흐름**: 사용자 요청 수신 → 판단·포맷 결정 (Haiku) → `mcp__local-llm__*` 호출 또는 `~/gemma4-bench/scripts/dispatch.sh`로 위임 (Gemma4, 로컬 무료) → 결과 반환
> **비용 모델**: Haiku 입력/출력 토큰만 과금, 실제 생성 작업은 0원.

## 대응 범위 (판단 불필요 작업만)
- 제공된 텍스트 요약 (길이 제약 준수)
- JSON/YAML/TOML 포맷 변환 및 정렬
- 언어 번역 (한↔영, 한↔일)
- 로그에서 특정 필드 추출
- 템플릿에 값 채우기
- 마크다운 표 ↔ JSON 배열 변환
- 파일명 일괄 규칙 변환 (snake_case → camelCase 등)

## 금지 (반드시 거부하고 상위 에이전트 제안)
- 코드 로직 수정
- 버그 원인 분석
- 설계 결정
- 외부 API 호출
- 여러 파일 간 상관관계 판단

## 호출 방법 (우선순위 순)
1. **MCP 툴**: `mcp__local-llm__*` 도구 사용 (settings.json에 `local-llm` MCP 서버 등록 전제)
2. **디스패처 스크립트** (대안):
   ```bash
   ~/gemma4-bench/scripts/dispatch.sh "작업 지시문"
   ```
3. **직접 LM Studio API** (최후):
   ```bash
   curl -s http://localhost:1234/v1/chat/completions \
     -H "Content-Type: application/json" \
     -d '{"model":"'"$LOCAL_LLM_MODEL"'","messages":[{"role":"user","content":"..."}]}'
   ```

## 선행 체크
호출 전 반드시 실행:
1. LM Studio 서버 응답 확인: `curl -sf http://localhost:1234/v1/models > /dev/null`
2. 시스템 부하 확인: `~/.claude/hooks/local-llm-gate.sh` (부록 G)
3. 실패 시 폴백: 상위 에이전트에 "로컬 LLM 비가용 — 클라우드로 처리 필요" 보고

## 출력 형식
- 원문 변환 결과만 반환 (설명 최소화)
- 판단 필요 작업 감지 시: "이 작업은 판단이 필요 — [적절한 에이전트] 권장"
- 로컬 LLM 호출 실패 시: "LOCAL_LLM_UNAVAILABLE" 접두로 상위 보고
```

---

# 부록 C. 로컬 LLM 자동 라우팅 훅 (선택)

## C-1. `~/.claude/hooks/route-to-local.sh`

사용자 프롬프트에 특정 키워드(요약, 번역, 포매팅 등)가 감지되면 로컬 LLM으로 우선 처리 힌트를 주는 훅.

> **입력 방식 주의**: Claude Code `UserPromptSubmit` 훅은 **stdin으로 JSON**을 받습니다. 환경 변수 `CLAUDE_USER_PROMPT`는 존재하지 않습니다. JSON은 `{"session_id":..., "hook_event_name":"UserPromptSubmit", "prompt":"사용자 입력", "cwd":..., ...}` 형식.

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/route-to-local.sh
# UserPromptSubmit 훅 — 로컬 LLM으로 위임 가능한 작업 감지 시 힌트 출력
# 입력: stdin으로 JSON (prompt 필드)
# 출력: stdout (Claude에 추가 컨텍스트로 전달됨)

set -euo pipefail

# stdin JSON에서 prompt 추출 (jq가 없어도 python3로 파싱)
INPUT=$(cat)
PROMPT=$(printf '%s' "$INPUT" | python3 -c "
import sys, json
try:
    data = json.load(sys.stdin)
    print(data.get('prompt', ''), end='')
except Exception:
    pass
")

MATCHERS=("요약해" "번역해" "포맷팅" "포매팅" "정렬해" "변환해" "추출해")

for keyword in "${MATCHERS[@]}"; do
  if [[ "$PROMPT" == *"$keyword"* ]]; then
    # LM Studio 서버 상태
    if curl -sf http://localhost:1234/v1/models > /dev/null 2>&1; then
      STATUS="활성"
    else
      STATUS="비활성 — 클라우드 폴백"
    fi
    cat <<HINT
[라우팅 힌트] 기계적 변환 작업으로 판단됩니다.
→ local-ops 에이전트(Gemma-4-26B, 로컬 무료) 사용을 권장합니다.
→ LM Studio 서버: $STATUS
HINT
    exit 0
  fi
done

# 매칭 없으면 조용히 종료 (출력 없음)
exit 0
```

## C-2. `settings.json`에 훅 등록 (STEP 3 이후 수정)

기존 `settings.json`에 `hooks` 섹션 추가:

```json
{
  "permissions": { "...": "..." },
  "mcpServers": { "...": "..." },
  "hooks": {
    "UserPromptSubmit": [
      {
        "matcher": "(요약해|번역해|포매팅|포맷팅|정렬해|변환해|추출해)",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/route-to-local.sh"
          }
        ]
      }
    ]
  }
}
```

## C-3. 훅 검증

```bash
# 훅 단독 실행 테스트 (stdin으로 JSON 주입)
echo '{"session_id":"test","hook_event_name":"UserPromptSubmit","prompt":"이 로그 파일 요약해줘","cwd":"/tmp"}' \
  | ~/.claude/hooks/route-to-local.sh

# 기대 출력 (3줄):
# [라우팅 힌트] 기계적 변환 작업으로 판단됩니다.
# → local-ops 에이전트(Gemma-4-26B, 로컬 무료) 사용을 권장합니다.
# → LM Studio 서버: 활성 (또는 비활성)

# 매칭 없는 경우 출력 없음 (정상)
echo '{"prompt":"새 기능 설계해줘"}' | ~/.claude/hooks/route-to-local.sh
```

---

# 부록 D. MEMORY.md 프로젝트별 분리 템플릿

## D-1. 전역 MEMORY.md (최소화)

`~/.claude/projects/-Users-cjons/memory/MEMORY.md`:

```markdown
## 공통 (모든 프로젝트 공유)
- [사용자 프로필](user_profile.md) — 역할, 스택, 작업 방식
- [커밋/보고 규칙](feedback_commit_state.md) — 논리적 단위 커밋 + 리포트 포맷
- [파괴적 작업 백업](feedback_backup_before_destructive.md) — cp -Rp 선행 필수

## 파이프라인 공통
- [Paperclip 워크플로우](reference_paperclip_workflows.md) — dispatch/cleanup/reset/split
- [토큰 최적화 원칙](feedback_token_optimization.md) — 프롬프트 분기, 재시도 카운터
```

프로젝트별 메모리는 해당 디렉터리로 이관:
- `~/.claude/projects/-Users-cjons-Documents-dev-macromalt/memory/MEMORY.md` (macromalt 전용)
- `~/.claude/projects/-Users-cjons-Documents-tms/memory/MEMORY.md` (tms 전용)

## D-2. 이관 스크립트

```bash
# 백업
cp -Rp ~/.claude/projects ~/.claude/backups/projects-$(date +%Y%m%d)

# macromalt 관련 메모리 이관
MACRO_SRC=~/.claude/projects/-Users-cjons/memory
MACRO_DST=~/.claude/projects/-Users-cjons-Documents-dev-macromalt/memory
mkdir -p "$MACRO_DST"
mv "$MACRO_SRC"/project_macromalt*.md "$MACRO_DST/" 2>/dev/null
mv "$MACRO_SRC"/reference_macromalt*.md "$MACRO_DST/" 2>/dev/null

# 각 프로젝트에 MEMORY.md 생성 (프로젝트별 인덱스)
# ... (프로젝트별 수동 편집)

# 전역 MEMORY.md 간소화
# vi ~/.claude/projects/-Users-cjons/memory/MEMORY.md
# → 공통 + 파이프라인 공통만 남기고 삭제
```

---

# 부록 F. 하드웨어 자동 감지 + 로컬 LLM 모델 자동 선택 (Mac Silicon)

> **목표**: 신규 노트북 세팅 시 CPU/RAM/GPU 코어를 자동 감지하여 해당 머신에 최적인 모델을 추천·다운로드.

## F-1. Apple Silicon 사양 → 모델 매트릭스

Apple Silicon은 **Unified Memory Architecture**로 RAM = VRAM 공유. 따라서 RAM 기준으로 모델을 선택.

| RAM | 권장 로컬 LLM | 양자화 | 디스크 | 비고 |
|---|---|---|---|---|
| ≤ 8 GB | **로컬 LLM 비권장** — OpenRouter 폴백 전용 | — | — | 시스템 여유 부족 |
| 9-16 GB | Qwen3-4B-Instruct / Gemma3-4B-IT | Q4_K_M | 3-4 GB | 경량 기계 작업만 |
| 17-24 GB | **Gemma-4-26B-A4B-IT** (MoE) / Qwen3-8B | Q4_K_M | 14-15 GB | **본 머신 (M2/24GB) 최적** |
| 25-36 GB | Qwen3-32B / Mixtral-8x7B-Instruct | Q4_K_M | 19-26 GB | 코딩 작업 가능 |
| 37-48 GB | Llama-3.3-70B-Instruct | Q3_K_M | 32 GB | 프론티어 근접 |
| 49-64 GB | Llama-3.3-70B-Instruct | Q4_K_M | 40 GB | 코딩/추론 우수 |
| 65-96 GB | Llama-3.3-70B / DeepSeek-V3 distill | Q6_K / Q8 | 55-75 GB | Claude Sonnet 근접 |
| 97-128 GB | Llama-3.3-70B Q8 + 보조 모델 다중 로드 | Q8 | 75 GB+ | 최고 구성 |

> **여유 RAM 규칙**: 모델 크기의 **1.5배** RAM 필요 (OS/앱/캐시 포함). 예: 16GB 모델 → 24GB RAM.

## F-2. 자동 감지 스크립트

`~/.claude/setup/detect-hardware.sh`:

```bash
#!/usr/bin/env bash
# Mac Silicon 하드웨어 감지 + 최적 로컬 LLM 추천
set -euo pipefail

CHIP=$(sysctl -n machdep.cpu.brand_string)
RAM_GB=$(( $(sysctl -n hw.memsize) / 1024 / 1024 / 1024 ))
CPU_CORES=$(sysctl -n hw.ncpu)
P_CORES=$(sysctl -n hw.perflevel0.physicalcpu 2>/dev/null || echo "?")
E_CORES=$(sysctl -n hw.perflevel1.physicalcpu 2>/dev/null || echo "?")
GPU_CORES=$(system_profiler SPDisplaysDataType | awk '/Total Number of Cores/ {print $NF; exit}')
MACOS_VER=$(sw_vers -productVersion)

# 모델 추천
if [ "$RAM_GB" -le 8 ]; then
  REC_MODEL="none"
  REC_NOTE="로컬 LLM 비권장 — OpenRouter 폴백만 사용"
elif [ "$RAM_GB" -le 16 ]; then
  REC_MODEL="qwen/qwen3-4b-instruct"
  REC_NOTE="경량 기계 작업 전용"
elif [ "$RAM_GB" -le 24 ]; then
  REC_MODEL="lmstudio-community/gemma-4-26b-a4b-it"
  REC_NOTE="MoE 26B (활성 파라미터 4B) — 기계 작업 최적"
elif [ "$RAM_GB" -le 36 ]; then
  REC_MODEL="lmstudio-community/qwen3-32b-instruct"
  REC_NOTE="코딩 작업 가능"
elif [ "$RAM_GB" -le 48 ]; then
  REC_MODEL="lmstudio-community/llama-3.3-70b-instruct-Q3_K_M"
  REC_NOTE="70B Q3 — 프론티어 근접"
elif [ "$RAM_GB" -le 96 ]; then
  REC_MODEL="lmstudio-community/llama-3.3-70b-instruct-Q4_K_M"
  REC_NOTE="70B Q4 — 코딩/추론 우수"
else
  REC_MODEL="lmstudio-community/llama-3.3-70b-instruct-Q8_0"
  REC_NOTE="70B Q8 — 최고 구성"
fi

cat <<REPORT
=== Mac Silicon 하드웨어 감지 결과 ===
Chip       : $CHIP
CPU        : ${CPU_CORES} cores (P:$P_CORES + E:$E_CORES)
GPU        : ${GPU_CORES:-?} cores
RAM        : ${RAM_GB} GB (Unified Memory)
macOS      : $MACOS_VER

=== 권장 로컬 LLM ===
모델       : $REC_MODEL
비고       : $REC_NOTE

=== 다음 단계 ===
REPORT

if [ "$REC_MODEL" != "none" ]; then
  cat <<NEXT
1. LM Studio 실행: open -g -a "LM Studio"
2. 서버 시작: lms server start --port 1234
3. 모델 다운로드:
   lms get "$REC_MODEL"
4. 모델 로드 (GPU offload 권장값은 GPU 코어 수 × 1.25):
   lms load "$REC_MODEL" --gpu-offload $(( ${GPU_CORES:-10} * 5 / 4 )) --context-length 16384
NEXT
else
  echo "로컬 LLM 생략 — OpenRouter API 키만 준비 (부록 H 참조)"
fi
```

## F-3. LM Studio 설정 체크

LM Studio 파라미터 튜닝 가이드 (`~/.claude/projects/-Users-cjons/memory/feedback_lm_studio_gemma4_config.md` 기반):

| 항목 | 권장값 | 주의 |
|---|---|---|
| Context length | 16384 | Q8 KV 캐시 비권장 (지연 증가) |
| GPU Offload | GPU 코어 × 1.25 | M2 8코어 → 10 layers |
| Flash Attention | On | Apple Silicon 기본 지원 |
| K/V Cache Type | fp16 | Q8 사용 시 느려짐 |
| Timeout | 240s | 초기 로드 + 긴 추론 대응 |

## F-4. 실행

```bash
mkdir -p ~/.claude/setup
# F-2 스크립트를 ~/.claude/setup/detect-hardware.sh로 저장
chmod +x ~/.claude/setup/detect-hardware.sh
~/.claude/setup/detect-hardware.sh
```

---

# 부록 G. 백그라운드 작업 CPU/GPU 사용량 기반 Throttle

> **목표**: 사용자가 Xcode/영상 편집/빌드 등 무거운 작업 중일 때 로컬 LLM 호출을 **자동 대기**시키고, 부하가 내려가면 재개.

## G-1. 시스템 부하 측정 전략 (Apple Silicon)

Apple Silicon의 GPU 사용률은 `sudo powermetrics`로만 정확히 측정 가능 (보안 제한). sudo 없이 **근사 측정**:

| 지표 | 명령 | 해석 |
|---|---|---|
| CPU idle % | `top -l 1 -n 0 \| awk '/CPU usage/ {print $7}'` | ≥ 40% = 여유 있음 |
| Load average (1m) | `sysctl -n vm.loadavg \| awk '{print $2}'` | ≤ CPU 코어 수의 70% = 여유 |
| 메모리 가용률 | `vm_stat` 페이지 기반 계산 `(free+inactive)/total` | ≥ 8% = 여유 (LLM 로드 상태 고려) |
| LM Studio 프로세스 존재 | `pgrep -f "LM Studio"` | 서버 기동 확인 |

> **참고**: GPU 사용률 정확 측정은 `sudo powermetrics --samplers gpu_power -i 1000 -n 1`이 필요하나, sudo 없이는 CPU/load/memory로 대체.
>
> **메모리 측정 실전 이슈 (2026-04-20 발견, 본 가이드 부록 L-1)**: `memory_pressure` 명령은 macOS 버전별로 출력 포맷이 다르고 macOS 26.x에서는 `System-wide memory free percentage` 필드가 **없음**. 또한 로컬 LLM(Gemma4 26B)이 로드된 상태에서 메모리 가용률은 **8-15% 수준**이 정상. 20% 임계는 **영구 BUSY 상태를 유발**하므로 본 가이드는 **vm_stat 페이지 계산 + 8% 임계**로 확정.

## G-2. Throttle 훅 스크립트

`~/.claude/hooks/local-llm-gate.sh`:

```bash
#!/usr/bin/env bash
# 로컬 LLM 호출 전 시스템 부하 검사 — 여유 생길 때까지 대기, 타임아웃 시 클라우드 폴백
# 메모리 측정: vm_stat 기반 (memory_pressure 명령은 macOS 버전별 포맷 불일치)
set -euo pipefail

THRESHOLD_CPU_IDLE=${LOCAL_LLM_CPU_IDLE_THRESHOLD:-40}   # CPU idle% 기준
THRESHOLD_MEM_FREE=${LOCAL_LLM_MEM_FREE_THRESHOLD:-8}    # 기본 8% (LLM 로드 상태 고려)
MAX_WAIT_SEC=${LOCAL_LLM_MAX_WAIT:-180}                   # 최대 대기 3분
WAIT_INTERVAL=${LOCAL_LLM_WAIT_INTERVAL:-15}              # 재검사 주기
CORE_COUNT=$(sysctl -n hw.ncpu)
LOAD_LIMIT=$(awk -v c="$CORE_COUNT" 'BEGIN{print c*0.7}')

check_load() {
  # CPU idle (top 한 번 샘플링)
  local idle_pct
  idle_pct=$(top -l 1 -n 0 2>/dev/null | awk '/CPU usage/ {gsub("%",""); print $7}')
  local idle_int="${idle_pct%.*}"
  [ -z "$idle_int" ] && idle_int=100

  # Load average (1분)
  local load1
  load1=$(sysctl -n vm.loadavg | awk '{print $2}')

  # 메모리 가용률 — vm_stat 페이지 기반 계산 (macOS 버전 무관)
  # 가용 = free + inactive (inactive는 즉시 회수 가능)
  local free_p inact_p act_p wired_p total mem_free
  free_p=$(vm_stat | awk '/Pages free/ {gsub("\\.",""); print $3+0; exit}')
  inact_p=$(vm_stat | awk '/Pages inactive/ {gsub("\\.",""); print $3+0; exit}')
  act_p=$(vm_stat | awk '/Pages active/ {gsub("\\.",""); print $3+0; exit}')
  wired_p=$(vm_stat | awk '/Pages wired down/ {gsub("\\.",""); print $4+0; exit}')
  total=$(( ${free_p:-0} + ${inact_p:-0} + ${act_p:-0} + ${wired_p:-0} ))
  if [ "$total" -gt 0 ]; then
    mem_free=$(( (${free_p:-0} + ${inact_p:-0}) * 100 / total ))
  else
    mem_free=50
  fi

  # 판정: CPU 여유 + 부하 낮음 + 메모리 여유
  awk -v i="$idle_int" -v t="$THRESHOLD_CPU_IDLE" \
      -v l="$load1" -v ll="$LOAD_LIMIT" \
      -v m="$mem_free" -v mt="$THRESHOLD_MEM_FREE" '
    BEGIN {
      if (i >= t && l <= ll && m >= mt) {
        printf "OK idle=%s%% load=%s mem_free=%s%%\n", i, l, m
        exit 0
      } else {
        printf "BUSY idle=%s%% load=%s mem_free=%s%%\n", i, l, m
        exit 1
      }
    }'
}

# LM Studio 응답 확인 (폴백 판단)
if ! curl -sf http://localhost:1234/v1/models > /dev/null 2>&1; then
  echo "LOCAL_LLM_DOWN: LM Studio 서버 응답 없음 — 클라우드로 폴백"
  exit 2
fi

elapsed=0
while [ "$elapsed" -lt "$MAX_WAIT_SEC" ]; do
  if result=$(check_load); then
    echo "GO: $result (waited ${elapsed}s)"
    exit 0
  else
    echo "WAIT: $result (elapsed=${elapsed}s, next check in ${WAIT_INTERVAL}s)"
    sleep "$WAIT_INTERVAL"
    elapsed=$((elapsed + WAIT_INTERVAL))
  fi
done

echo "TIMEOUT: 시스템 부하 지속 ${MAX_WAIT_SEC}s — 클라우드로 폴백 권장"
exit 3
```

배포:
```bash
mkdir -p ~/.claude/hooks
# 위 스크립트를 ~/.claude/hooks/local-llm-gate.sh로 저장
chmod +x ~/.claude/hooks/local-llm-gate.sh

# 테스트
~/.claude/hooks/local-llm-gate.sh
# 출력 예: "GO: OK (waited 0s)" 또는 "WAIT: BUSY idle=28% ..."
```

## G-3. 로컬 LLM 디스패처에 Gate 통합

`~/gemma4-bench/scripts/dispatch.sh` 상단에 삽입 (또는 래퍼 작성):

```bash
#!/usr/bin/env bash
# dispatch-with-gate.sh — gate → 통과 시 dispatch / 실패 시 폴백
set -euo pipefail

GATE_RESULT=$(~/.claude/hooks/local-llm-gate.sh 2>&1) || GATE_EXIT=$?
GATE_EXIT=${GATE_EXIT:-0}

case "$GATE_EXIT" in
  0) # 통과
    exec ~/gemma4-bench/scripts/dispatch.sh "$@"
    ;;
  2|3) # 서버 다운 또는 타임아웃
    echo "[FALLBACK] 로컬 LLM 비가용 — 상위 에이전트에 클라우드 처리 요청 필요"
    echo "사유: $GATE_RESULT"
    exit 1
    ;;
  *) # 기타
    echo "[ERROR] gate 실패: $GATE_RESULT"
    exit 1
    ;;
esac
```

## G-4. Throttle 튜닝 환경 변수

개별 머신 성능에 맞게 조정:

```bash
# ~/.zshrc 또는 프로젝트별 .envrc
export LOCAL_LLM_CPU_IDLE_THRESHOLD=30   # 저사양 머신은 낮춤 (20-30)
export LOCAL_LLM_MEM_FREE_THRESHOLD=8    # LLM 상시 로드 상태면 낮춤 (5-10)
export LOCAL_LLM_MAX_WAIT=60             # 빠른 폴백 원하면 짧게
export LOCAL_LLM_WAIT_INTERVAL=10        # 체크 주기
```

---

# 부록 H. Steve Hanov "$20/mo 스택" 원칙 (Mac Silicon 적용)

> **출처**: https://stevehanov.ca/blog/how-i-run-multiple-10k-mrr-companies-on-a-20month-tech-stack
> **핵심 철학**: "비용을 0에 가깝게 유지하면 수백만 달러 투자와 같은 런웨이를 복잡도·스트레스 없이 얻는다."

## H-1. Hanov 원칙 5가지 → Mac Silicon 번역

| # | Hanov 원칙 (원본) | 본 환경 적용 (Mac Silicon) |
|---|---|---|
| 1 | **SQLite + WAL이 원격 Postgres보다 빠르다** (로컬 파일 I/O < TCP hop) | 신규 프로젝트 DB는 **SQLite WAL 기본**. 멀티 인스턴스 필요 시에만 Postgres. |
| 2 | **RTX 3090 자가 보유 + VLLM 로컬 추론** ($900 일회성) | **Apple Silicon Unified Memory + LM Studio (Metal)**. 이미 보유 — 추가 비용 0. |
| 3 | **OpenRouter로 프론티어 폴백 라우팅** | Anthropic 직접 + **OpenRouter 백업 경로** (일시 장애 대비). |
| 4 | **Linode/DigitalOcean VPS $5/mo, Go 단일 바이너리 scp 배포** | **Railway $5/mo Hobby + `railway up`** 또는 Fly.io free tier. 복잡한 K8s 금지. |
| 5 | **VS Code + Copilot per-request 요금** (Cursor 회피) | **Claude Code + VS Code** (Cursor 불필요). 서브에이전트/로컬 LLM으로 per-token 비용 절감. |

## H-2. 피해야 할 것 (Hanov 리스트 + 한국 개발자 맥락)

| ❌ 회피 | ✅ 대안 |
|---|---|
| AWS EKS / RDS / NAT Gateway | Railway / Fly.io / Cloudflare Workers |
| Cursor / 과도한 AI IDE 구독 | Claude Code + VS Code 무료 |
| Datadog / New Relic | Sentry 무료 + Railway 기본 로그 |
| Auth0 / Clerk 유료 티어 | NextAuth.js / 직접 구현 (@smhanov/auth 참조) |
| CloudWatch 복잡 모니터링 | Cron + 슬랙 웹훅 단순 알림 |
| Kubernetes / Docker Swarm | 단일 VPS + systemd (또는 Railway) |
| Postgres (소규모 단일 노드) | SQLite + WAL + 백업 스크립트 |

## H-3. 본 환경 권장 스택 (월 예산 $20-40)

| 계층 | 선택 | 월 비용 | 근거 |
|---|---|---|---|
| 개발 IDE | VS Code + Claude Code | $0 (Claude 정액) | Cursor 대비 per-request 저렴 |
| 로컬 LLM | LM Studio + Gemma-4-26B (M2 24GB) | $0 | 기계 작업 전담 |
| AI 프론티어 | Anthropic Claude + OpenRouter 폴백 | 사용량 기반 | 복잡 판단만 |
| 호스팅 (개발) | 로컬 Mac + ngrok (필요 시) | $0 | 개발 단계 |
| 호스팅 (운영) | Railway Hobby | $5 | 단일 서비스당 |
| DB | SQLite + WAL (단일 노드) / Railway Postgres (멀티) | $0 / $5 | 규모에 따라 |
| DNS/CDN/SSL | Cloudflare (free tier) | $0 | 기본 전부 무료 |
| 도메인 | Namecheap / Porkbun | $1/mo (연 $10-12) | `.com` 기준 |
| 모니터링 | Sentry Free + Railway 로그 + 슬랙 웹훅 | $0 | 10K 이벤트/mo 충분 |
| CI/CD | GitHub Actions (public 무제한, private 2,000분) | $0 | 본 환경 `my-claude-global` 이미 활용 |
| 인증 | NextAuth.js + GitHub/Google OAuth | $0 | 사용자 수 무관 |
| 결제 (필요 시) | Stripe (과금 시점만 수수료) | $0 기본 | 사용량 기반 |
| **합계 (소규모)** | | **$6-10/mo** | |
| **합계 (2-3 서비스)** | | **$15-25/mo** | |

## H-4. 본 환경 파이프라인 이미 적용된 Hanov 원칙

| 원칙 | 현재 상태 | 추가 개선 |
|---|---|---|
| 복잡도 최소화 | `my-claude-global` 단일 레포, Paperclip 중앙 디스패치 | ✅ 이미 적용 |
| 로컬 LLM 활용 | Gemma-4-26B MCP 연결됨 | 서브에이전트로 자동 라우팅 (부록 B) |
| CI/CD 무료 | GitHub Actions 9 workflow | ✅ 이미 적용 |
| 배포 단순화 | Paperclip → dispatch → PR 머지 | ✅ 이미 적용 |
| SQLite 우선 | `session-archive` SQLite 사용 | 신규 프로젝트 기본 원칙 추가 |

## H-5. 신규 프로젝트 체크리스트 (Hanov 스타일)

```
[ ] Postgres 대신 SQLite WAL로 시작 가능한가? (단일 노드이면 YES)
[ ] 인증은 NextAuth + OAuth (GitHub/Google)로 충분한가? (SaaS 초기 YES)
[ ] 배포는 Railway git push 한 번으로 끝나는가? (YES면 K8s 도입 금지)
[ ] 모니터링은 Sentry free + 슬랙 알림으로 충분한가? (초기 YES)
[ ] 프론티어 LLM 호출은 꼭 필요한가? 로컬 Gemma로 대체 가능한가?
[ ] 도메인은 1개로 통합 가능한가? (서브도메인으로 분리)
[ ] 백업은 Railway Artifact + GitHub Actions cron으로 충분한가? (YES)
```

7개 항목 중 5개 이상 YES → Hanov 스택 적합. 그 이하 → 요구사항 재검토.

---

# 부록 I. 전체 문서 점검 결과 (Audit)

> **2026-04-20 점검 수행** — 이 문서를 에이전트가 읽고 실행할 때 문제가 없도록 전체 재검토.

## I-1. 수정 완료 항목 (본문 반영)

| # | 이슈 | 영향 | 수정 내용 |
|---|---|---|---|
| 1 | `local-ops.md`의 `model: inherit` | 치명 — 메인(Opus) 상속으로 비용 절감 반대 효과 | `model: haiku` + MCP 툴 위임 구조로 재설계 (부록 B-5) |
| 2 | settings.json 기본 권한 부족 | 중 — mkdir/cp/mv/rm 누락으로 프롬프트 다발 | 파일 조작 명령 10종 추가 (부록 A STEP 3) |
| 3 | `/Users/cjons/` 하드코딩 | 중 — 타 PC 이관 불가 | `$HOME` 변수화 + heredoc 확장 (부록 A STEP 3) |
| 4 | 권한 문법 혼재 (스페이스/콜론) | 소 — 일부 머신에서 패턴 매칭 실패 가능 | `Bash(tool:*)` 콜론 형식으로 통일 (부록 A STEP 3) |
| 5 | 모델 다운로드 명령 예시 오류 | 중 — 신규 사용자 혼란 | 하드웨어 감지 스크립트로 동적 추천 (부록 F-2) |

## I-2. 남은 제약/주의사항 (사용자 확인 필요)

| # | 항목 | 영향 | 권장 조치 |
|---|---|---|---|
| A | Claude Code 서브에이전트 `model` 필드는 Claude 모델만 지원 | — | 부록 B-5에 명시 (Haiku 오케스트레이터 + MCP 위임) |
| B | GPU 사용률 정확 측정은 `sudo powermetrics` 필요 | 소 | 부록 G-1에 CPU idle + load + memory 근사법 사용 명시 |
| C | Hook matcher가 과도 매칭 가능 (`분석해` 등 판단 필요 작업 포함) | 중 | 부록 C의 matcher는 **힌트 제공만** — 실제 위임은 에이전트 판단으로 제한됨 |
| D | LM Studio 모델명은 허깅페이스 상태에 따라 변동 | 소 | 부록 F-2 스크립트 실행 후 `lms get` 실패 시 `huggingface.co` 검색 필요 |
| E | `my-claude-global` 레포가 hs85-newbie 소유 (개인 계정) | 소 | 신규 사용자는 fork 후 자체 URL로 clone 필요 |
| F | macOS 26.x 최신 환경 호환성 미검증 | 소 | `top -l 1` 및 `sysctl` 은 Darwin 전체 호환 — 문제 없음 |

## I-3. 실행 전 반드시 확인할 Preflight

신규 세팅 시작 전 아래를 차례로 확인:

```bash
# Preflight 체크 스크립트
cat <<'EOF' | bash
echo "=== Preflight Check ==="
# 1. 필수 도구
for cmd in git node pnpm bun python3 gh curl; do
  command -v $cmd >/dev/null && echo "✓ $cmd" || echo "✗ $cmd 없음 — brew install 필요"
done
# 2. 디스크 공간 (최소 30GB 권장 for 로컬 LLM)
free_gb=$(df -g $HOME | awk 'NR==2 {print $4}')
[ "${free_gb:-0}" -ge 30 ] && echo "✓ 디스크 여유 ${free_gb}GB" || echo "⚠ 디스크 여유 ${free_gb}GB (30GB 권장)"
# 3. RAM
ram_gb=$(( $(sysctl -n hw.memsize) / 1024 / 1024 / 1024 ))
[ "$ram_gb" -ge 16 ] && echo "✓ RAM ${ram_gb}GB" || echo "⚠ RAM ${ram_gb}GB (로컬 LLM 비권장)"
# 4. 기존 ~/.claude 백업 필요 여부
[ -d ~/.claude ] && echo "⚠ 기존 ~/.claude 존재 — 백업 필수 (cp -Rp ~/.claude ~/.claude.backup-$(date +%Y%m%d))"
echo "=== 완료 ==="
EOF
```

## I-4. 문서 내 상호 참조 검증

| 참조 | 대상 | 상태 |
|---|---|---|
| 부록 A STEP 4 | → 부록 B (에이전트 5종) | ✓ 본문 완전 |
| 부록 A STEP 6 | → 부록 F-2 (하드웨어 감지) 선행 필수 | ✓ 본문 명시 |
| 부록 A STEP 7 | → 부록 C (라우팅 훅) + 부록 G (throttle 훅) | ✓ 본문 완전 |
| 부록 A STEP 8 | → 부록 A-3 트러블슈팅 | ✓ 본문 완전 |
| 부록 B-5 `local-ops` | → `~/gemma4-bench/scripts/dispatch.sh` | ⚠ 외부 레포 의존 — 없으면 직접 API 사용 |
| 부록 B-5 MCP 툴 | → `local-llm` MCP 서버 (settings.json) | ✓ 부록 A STEP 3에서 정의 |
| 부록 C hook | → `~/.claude/hooks/route-to-local.sh` | ✓ 본문 완전 |
| 부록 D 이관 스크립트 | → 기존 `~/.claude/projects/` 구조 | ✓ 실제 구조와 일치 |
| 부록 F 하드웨어 감지 | → F-2 스크립트, F-4 배포 | ✓ 본문 완전 |
| 부록 G Throttle | → G-2 스크립트, G-3 디스패처 | ✓ 본문 완전 |
| 부록 H Hanov | → H-5 체크리스트 | ✓ 본문 완전 |
| 부록 I-3 Preflight | → PRE 단계 (부록 J) | ✓ 체크리스트 반영 |
| 부록 K 토큰 효율 | → STEP 12/13 (부록 J) | ✓ 체크리스트 반영 |
| 부록 J (체크리스트) | → 모든 부록 종합 | ✓ 문서 최종 — 알파벳 순 어긋남은 의도적 (체크리스트가 말미 배치) |

## I-5. 리스크 매트릭스

| 적용 단계 | 실패 시 영향 | 복구 방법 |
|---|---|---|
| STEP 1 (심링크) | 전역 규칙 미적용 | `ln -sf` 재실행 |
| STEP 3 (settings.json) | 권한 프롬프트 폭증 | 기존 `settings.local.json` 백업본으로 롤백 |
| STEP 4 (에이전트 5종) | 위임 실패 → 메인 모델 처리 | 개별 md 삭제 → 기본 내장 에이전트로 폴백 |
| STEP 6a (하드웨어 감지) | 모델 추천 오류 | F-1 매트릭스 수동 참조 |
| STEP 6b (모델 로드) | MCP 서버 미기동 | 클라우드 폴백 자동 (부록 G로 처리) |
| STEP 6c (gemma4-bench) | 디스패처 부재 | MCP 직접 호출 또는 curl API |
| STEP 7a (throttle 훅) | 무한 대기 | `LOCAL_LLM_MAX_WAIT=60` 로 조정 |
| STEP 7b (라우팅 힌트 훅) | 힌트만 출력, 실행 방해 안 함 | 훅 파일 삭제 |

## I-6. 문서 개정 이력

| 날짜 | 변경 | 사유 |
|---|---|---|
| 2026-04-20 | 초안 작성 (본문 1-6장, 부록 A-E) | 최초 구조 감사 |
| 2026-04-20 | 부록 F/G/H/I 추가 + B-5 수정 + STEP 3 권한 보완 | 하드웨어 감지, Hanov 원칙, 전체 점검 |
| 2026-04-20 | 전체 점검 11건 수정 (E-01 ~ E-11) + 부록 K 추가 | stdy.blog 토큰 효율 팁 반영, hook stdin 파싱 수정 |

---

# 부록 K. Claude Code 토큰 효율 미세 튜닝

> **출처**: https://www.stdy.blog/increasing-token-efficiency-by-setting-adjustment-in-claude-and-codex/ (2026-04-19)
> **목표**: 설정/환경변수/플래그 조정으로 부록 A의 기본 세팅 위에 추가로 토큰 절약.
> **배경**: Opus 4.7은 토크나이저 변경으로 동일 입력이 **1.0~1.35배 더 많은 토큰**을 소비. 대화형 모드에서 turn 증가 시 추가 사고 토큰(thinking) 사용. 캐시 TTL도 1시간 → 5분으로 축소됨.

## K-1. 토큰이 새는 3대 경로

1. **매 세션/턴 자동 주입 컨텍스트**: CLAUDE.md, MEMORY, 빌트인 에이전트/스킬 description, Git 지침, IDE 상태
2. **툴 호출 출력 잔존**: Bash/Read/MCP 출력이 히스토리에 누적
3. **외부 커넥터**: 검색, IDE 연동, 미사용 MCP 서버

## K-2. `settings.json` 추가 옵션 (부록 A STEP 3에 이미 반영됨)

| 키 | 값 | 효과 |
|---|---|---|
| `includeGitInstructions` | `false` (기본: `true`) | Git 관련 시스템 프롬프트 섹션 제거. 짧지만 전 세션 누적 효과 큼 |
| `autoConnectIde` | `false` (기본: `false`) | 외부 터미널에서 실행 시 IDE 컨텍스트 자동 주입 차단 |
| `attribution.commit` | `""` (기본: 자동 서명) | "Co-Authored-By: Claude" 등 커밋 서명 제거 |
| `attribution.pr` | `""` (기본: 자동 서명) | PR 본문의 "Generated with Claude Code" 제거 |

> IDE 컨텍스트를 자주 쓰는 경우(드래그 선택 → 수정 등)는 `autoConnectIde: true` 유지 권장.

## K-3. 환경 변수로 추가 절감

`~/.zshrc`에 추가 (항상 적용할 것만 선별):

```bash
# === 상시 적용 권장 ===
export DISABLE_TELEMETRY=1                          # Anthropic 아웃바운드 트래픽 차단 (토큰 무관이지만 권장)
export CLAUDE_CODE_GLOB_NO_IGNORE=false             # .gitignore 파일을 Glob 결과에서 제외 (모노리포 효과 큼)

# === 출력 상한 (초대형 응답 방지 / 트레이드오프 있음) ===
# export BASH_MAX_OUTPUT_LENGTH=30000                # Bash 출력 최대 문자 (비공식 기본 30K)
# export CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS=25000  # Read 툴 최대 토큰
# export MAX_MCP_OUTPUT_TOKENS=25000                 # MCP 출력 최대 토큰
```

> **주의 (K-2 상한 트레이드오프)**: 상한을 낮추면 꼬리가 잘리며, 잘린 부분을 복구하려고 Claude가 `tail`/`grep`/재실행 등 **추가 툴 호출**을 시도해 **오히려 토큰이 더 소모**될 수 있음. 긴 출력의 꼬리(에러 요약, 최신 스택 트레이스)가 중요하므로 상한 조정은 신중히.

## K-4. 비대화형 워커 전용 alias (대폭 절감)

간단한 반복 작업(일괄 리네임, 파일 이동, 템플릿 적용)을 시킬 때만 컨텍스트를 모두 끄는 경량 모드:

아래 블록을 **그대로** `~/.zshrc`에 추가 (alias는 한 줄로 유지 — 복사 시 들여쓰기 공백 삽입 주의):

```bash
# ccb: Claude Code Bare — MCP/메모리/CLAUDE.md/빌트인 에이전트 모두 제외
alias ccb='ENABLE_CLAUDEAI_MCP_SERVERS=false CLAUDE_CODE_DISABLE_AUTO_MEMORY=1 CLAUDE_CODE_DISABLE_CLAUDE_MDS=1 CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=1 DISABLE_TELEMETRY=1 claude --tools "Bash,Edit,Glob,Grep,Read,Write" --disable-slash-commands --exclude-dynamic-system-prompt-sections'

# ccby: ccb + 권한 확인 생략 (신뢰 가능한 자동화 스크립트에서만)
alias ccby='ccb --dangerously-skip-permissions'
```

적용:
```bash
source ~/.zshrc
# 검증
type ccb   # 기대: "ccb is an alias for ..."
ccb --version
```

각 플래그/환경변수 의미:

| 플래그/변수 | 효과 |
|---|---|
| `ENABLE_CLAUDEAI_MCP_SERVERS=false` | Anthropic 제공 기본 MCP 서버 비활성화 |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` | `~/.claude/projects/*/memory/` 자동 로드 차단 |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS=1` | `~/.claude/CLAUDE.md` 및 프로젝트 CLAUDE.md 무시 |
| `CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=1` | 빌트인 서브에이전트/스킬 description을 시스템 프롬프트에서 제외 |
| `--tools "..."` | 네이티브 툴 허용 목록. 빈 값(`""`)이면 모두 비활성화 |
| `--disable-slash-commands` | `/help`, `/clear` 등 슬래시 커맨드 정의 제거 |
| `--exclude-dynamic-system-prompt-sections` | 머신별 동적 섹션 제외 → 프롬프트 캐시 재사용률 ↑ (반복 워커에서 효과 큼) |
| `--strict-mcp-config` | CLI로 명시한 MCP만 사용 (전역 무시) |
| `--no-session-persistence` | 세션 기록 저장 안 함 (일회성 실행) |
| `--system-prompt "..."` | 시스템 프롬프트 완전 교체 |

> **워커 사용 예**:
> ```bash
> ccby -p "이 디렉터리의 *.log 파일을 all.log로 합쳐" --cwd /tmp/logs
> ```

## K-5. 본 환경 상세 적용 권장

| 시나리오 | 권장 명령 | 절감 기대 |
|---|---|---|
| 대화형 일상 개발 | `claude` (표준 설정, K-2 반영) | 기본 절감 10-15% |
| 반복 포매팅/리네임 워커 | `ccb -p "작업 지시문"` | 60-80% (MCP/메모리/에이전트 모두 제외) |
| 신뢰 자동화 배치 | `ccby -p "..."` | 60-80% + 승인 오버헤드 0 |
| 긴 대화 세션 후반 | `--exclude-dynamic-system-prompt-sections` 추가 | 캐시 히트율 ↑ |

## K-6. 상태 점검 프롬프트

저자가 공유한 Gist(https://gist.github.com/spilist/c468cbf1ed0ffc91100f813aabdcd520)를 Claude에 붙여넣으면 현재 설정의 토큰 효율성을 자동 분석.

```
https://gist.github.com/spilist/c468cbf1ed0ffc91100f813aabdcd520?#file-token-efficiency-analysis-prompt-md 를 읽고 그대로 실행해줘
```

> **팁**: Anthropic/OpenAI 공식 문서 URL에 `.md`를 붙이면 마크다운 원문으로 받을 수 있어 (`https://code.claude.com/docs/en/settings.md`) 에이전트 컨텍스트 효율이 좋음.

## K-7. Codex CLI 쓰는 경우 (참고)

OpenAI Codex CLI 병행 사용 시:

```toml
# ~/.codex/config.toml
[features]
apps = false                    # ChatGPT 연결 앱/커넥터 시스템 프롬프트 제외

[apps._default]
enabled = false                 # 기본 앱 자동 주입 차단 (features.apps=false면 no-op이지만 보험용)

web_search = "disabled"         # 캐시된 웹 검색 비활성화 (로컬 작업 한정 시)
```

> Codex는 Claude 대비 토큰 여유가 큰 편 — 세밀 튜닝 우선순위 낮음.

## K-8. 트레이드오프 요약

| 설정 | 절감 | 손실 |
|---|---|---|
| `includeGitInstructions: false` | 중 | Git 관련 자동 워크플로우 일부 |
| `autoConnectIde: false` | 소 | IDE 드래그 선택/진단 자동 주입 |
| `CLAUDE_CODE_GLOB_NO_IGNORE=false` | 중-대 (모노리포) | `.gitignore` 파일 직접 접근 시 불편 |
| `BASH_MAX_OUTPUT_LENGTH` 축소 | 가변 | 꼬리 잘림 → 추가 재호출로 오히려 악화 가능 |
| `ccb` (bare 모드) | 대 | CLAUDE.md/메모리/에이전트 전부 차단 — 반복 워커 전용 |
| `attribution` 비우기 | 소 | 팀 정책에 "AI 표시" 있으면 충돌 |

---

# 부록 L. 실제 적용 기록 (본 머신 기준)

> **목적**: 본 문서의 가이드를 실제 하드웨어(Apple M2 / 24GB / macOS 26.3.1)에서 적용하며 발생한 에러, 원인, 해결 과정을 기록. 향후 독자가 동일 이슈를 빠르게 해결할 수 있도록 축적.

## L-0. 적용 환경

| 항목 | 값 |
|---|---|
| 머신 | MacBook Air (Apple M2) |
| CPU | 8 cores (P:4 + E:4) |
| GPU | 8 cores |
| RAM | 24 GB (Unified Memory) |
| macOS | 26.3.1 |
| LM Studio 모델 | `gemma-4-26b-a4b-it` (MoE, 14.82 GB, 로드 상태) |
| 기존 ~/.claude | 존재 (백업: `~/.claude.backup-20260420-1333`, 427MB) |

## L-1. 에러 #1 — Throttle 게이트 훅 영구 BUSY 루프

### 발생 단계
부록 A STEP 7 (hooks 배포) 의 `local-llm-gate.sh` 검증 테스트 중.

### 증상
```
WAIT: BUSY idle=43% load=2.10 mem_free=16% (elapsed=0s, next check in 15s)
WAIT: BUSY idle=76% load=2.08 mem_free=16% (elapsed=15s, next check in 15s)
WAIT: BUSY idle=77% load=2.16 mem_free=16% (elapsed=30s, next check in 15s)
...
```
CPU idle과 load는 여유 있는데 `mem_free=16%`가 임계 20% 미만으로 BUSY 판정 → 영원히 대기.

### 근본 원인 (2가지 복합)

**(A) `memory_pressure` 명령 출력 포맷 불일치**

macOS 26.3.1에서 `memory_pressure` 명령 출력:
```
The system has 25769803776 (1572864 pages with a page size of 16384).

Stats:
Pages free: 3840
Pages purgeable: 2
Pages purged: 81676266
...
```
→ 가이드 초안이 기대한 `System-wide memory free percentage` 필드 **부재**.
→ awk 파싱 결과가 기본값 `50`으로 떨어지거나, 잘못된 값(16)이 잡힘.

**(B) 임계값 20% 자체가 비현실적**

본 머신에서 `vm_stat` 기반 실측 결과:
- Pages free: 3,843
- Pages inactive: 124,408
- Pages active: 125,651
- Pages wired down: 922,325
- 가용 = (free + inactive) / total = **10.9%**

Gemma4-26B(14.82GB)가 wired로 상주하므로 메모리 가용률이 **구조적으로 20% 미달**. 임계 20%는 로컬 LLM 사용 시나리오와 모순.

### 해결

1. 메모리 측정 방식을 `memory_pressure` → **`vm_stat` 페이지 계산**으로 교체 (macOS 버전 무관)
2. 임계값 기본: `20%` → **`8%`** (LLM 상시 로드 상태 고려)
3. 환경변수 `LOCAL_LLM_MEM_FREE_THRESHOLD` 추가 → 사용자 머신별 조정 가능

### 수정 후 결과
```
GO: OK idle=75% load=2.63 mem_free=10% (waited 0s)
```
즉시 통과 확인.

### 교훈
- **외부 도구 출력은 OS 버전별로 깨질 수 있다** → 원시 시스템 콜(`sysctl`, `vm_stat`) 사용이 안전
- **임계값은 실제 사용 시나리오 기반** → 로컬 LLM 로드 전제로 기본값 재조정
- **환경변수로 override 가능**하게 설계 → 사용자 머신별 튜닝

### 관련 변경 파일
- GUIDE.md 부록 G-1 (측정 전략 표 + 주의사항 추가)
- GUIDE.md 부록 G-2 (스크립트 전문 교체)
- GUIDE.md 부록 G-4 (환경변수 목록에 `LOCAL_LLM_MEM_FREE_THRESHOLD` 추가)
- `~/.claude/hooks/local-llm-gate.sh` (실제 배포본)

---

# 부록 J. 실행 순서 요약 (에이전트용 체크리스트)

새 환경에서 Claude 에이전트가 이 문서를 읽고 **순차 실행**할 항목 (2026-04-20 업데이트).

> **번호 규칙**: STEP 1-8은 부록 A의 세팅 블록과 1:1 대응. STEP 9-13은 부록 A 이후 선택 단계 (각 부록 참조).

```
[ ] PRE:     부록 I-3 Preflight 체크 실행 (필수 도구 + 디스크 + RAM + 기존 백업)
[ ] STEP 1:  my-claude-global 클론 + CLAUDE.md 심링크                → 부록 A-STEP 1
[ ] STEP 2:  gh 인증 + ~/.anthropic/env 준비 (사용자 개입 필요)        → 부록 A-STEP 2
[ ] STEP 3:  ~/.claude/settings.json 배포 (토큰 효율 옵션 포함)       → 부록 A-STEP 3
[ ] STEP 4:  ~/.claude/agents/{plan,explore,coder,quick-lookup,local-ops}.md 생성 → 부록 A-STEP 4 + 부록 B
[ ] STEP 5:  Skills 심링크 (선택 — my-claude-global 내부 2종)          → 부록 A-STEP 5
[ ] STEP 6:  로컬 LLM 세팅 (하드웨어 감지 → 모델 결정 → 서버 기동)     → 부록 A-STEP 6 + 부록 F
[ ] STEP 7:  라우팅 + Throttle 훅 배포 + hooks 섹션 자동 병합          → 부록 A-STEP 7 + 부록 C + 부록 G
[ ] STEP 8:  최종 검증 스크립트 실행                                   → 부록 A-STEP 8
[ ] STEP 9:  Claude 세션 시작 → "이 로그 요약해줘"로 로컬 라우팅 확인   → 부록 B-5 / 부록 C-3
[ ] STEP 10: MEMORY.md 프로젝트별 재구성 (기존 환경 이관 시)            → 부록 D
[ ] STEP 11: 신규 프로젝트 체크리스트 적용 (Hanov 원칙)                 → 부록 H-5
[ ] STEP 12: .zshrc에 ccb/ccby alias + 토큰 효율 환경변수 추가          → 부록 K-3/K-4
[ ] STEP 13: 상태 점검 프롬프트로 설정 효율성 확인                      → 부록 K-6
```

완료 시 달성 목표:
- 세션 시작 토큰 ≤ 2.5K (Opus 4.7 토크나이저 기준 실측)
- 단순 작업 로컬 위임률 ≥ 40%
- 시스템 부하 높을 때 자동 대기 (부록 G)
- 월 인프라 비용 ≤ $25 (부록 H-3 기준)
- `ccb` 워커 모드 토큰 60-80% 절감 (부록 K-5)
