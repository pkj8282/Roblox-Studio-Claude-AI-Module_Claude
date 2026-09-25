# AI용 튜토리얼: Claude 구독 사용량 직접 확인하기

> 대상: Claude Code 안에서 동작하는 AI 에이전트
> 목적: 사용자가 `/usage`를 입력하지 않아도, AI가 스스로 **구독 요금제의 실제 한도 사용률**을 조회하고 판단에 활용한다.
> 검증일: 2026-09-25 (Windows 11, Pro 요금제에서 동작 확인)

---

## 1. 핵심 개념

| 방법 | 성격 | 용도 |
|---|---|---|
| **OAuth 사용량 API** (`/usage`와 동일 데이터) | 서버 정답값 | 한도 여유·초기화 시각 판단 ← **판단 기준** |
| `ccusage` | 로컬 로그 기반 추정 | 어느 날/모델에서 많이 썼는지 원인 분석 |
| 절대 한도 역산 (ccusage 비용 ÷ 사용률) | 오차 큼 | 참고용, 판단 근거로 쓰지 말 것 |

- `/usage`는 CLI 내장 명령이라 AI가 도구로 실행할 수 없다. 대신 `/usage`가 호출하는 **동일한 API**를 직접 호출한다.
- API는 **사용률(%)과 초기화 시각**만 준다. 절대 한도(달러/토큰)는 공개되지 않는다.

---

## 2. 보안 규칙 (반드시 준수)

1. **토큰(`accessToken`, `refreshToken`)을 절대 출력·로그·파일·커밋에 남기지 않는다.** 스크립트 내부에서만 사용한다.
2. 자격 증명 파일 전체를 `cat`/`Read`로 열지 않는다. 필요한 필드만 프로그램에서 읽는다.
3. 토큰을 `api.anthropic.com` 외의 곳으로 보내지 않는다.
4. 조회는 읽기 전용(GET)만 한다.

---

## 3. 준비 사항

| 항목 | 위치 |
|---|---|
| 자격 증명 (Windows / Linux) | `~/.claude/.credentials.json` → `claudeAiOauth.accessToken` |
| 자격 증명 (macOS) | 파일 대신 Keychain 항목 `Claude Code-credentials` (`security find-generic-password -s "Claude Code-credentials" -w`) |
| 런타임 | Node.js 18+ (내장 `fetch` 사용) 또는 PowerShell 5.1+ |

API 요청 형식:

```
GET https://api.anthropic.com/api/oauth/usage
Authorization: Bearer <accessToken>
anthropic-beta: oauth-2025-04-20
```

---

## 4. 실행 방법

### 방법 A — Node.js (Bash 도구, 크로스 플랫폼 권장)

```bash
node -e '
const fs=require("fs"),os=require("os"),path=require("path");
const c=JSON.parse(fs.readFileSync(path.join(os.homedir(),".claude",".credentials.json"),"utf8")).claudeAiOauth;
fetch("https://api.anthropic.com/api/oauth/usage",{headers:{Authorization:"Bearer "+c.accessToken,"anthropic-beta":"oauth-2025-04-20"}})
.then(async r=>{if(!r.ok){console.log("ERROR",r.status,await r.text());process.exit(1)}return r.json()})
.then(u=>{const f=t=>t?new Date(t).toLocaleString("ko-KR",{timeZone:"Asia/Seoul"}):"-";
console.log("plan:",c.subscriptionType);
for(const l of u.limits||[])console.log(`${l.kind}: ${l.percent}% (${l.severity}) reset ${f(l.resets_at)}`);
if(u.extra_usage)console.log("extra_usage: enabled="+u.extra_usage.is_enabled+" used="+u.extra_usage.used_credits);})
.catch(e=>{console.log("ERROR",e.message);process.exit(1)});
'
```

출력 예시:

```
plan: pro
session: 2% (normal) reset 2026. 9. 25. 오전 7:09:59
weekly_all: 4% (normal) reset 2026. 9. 26. 오후 6:59:59
extra_usage: enabled=true used=0
```

### 방법 B — PowerShell 도구 (Windows)

```powershell
$c = (Get-Content "$env:USERPROFILE\.claude\.credentials.json" -Raw | ConvertFrom-Json).claudeAiOauth
$h = @{ Authorization = "Bearer " + $c.accessToken; "anthropic-beta" = "oauth-2025-04-20" }
try {
  $u = Invoke-RestMethod -Uri "https://api.anthropic.com/api/oauth/usage" -Headers $h -Method Get
  "plan: " + $c.subscriptionType
  foreach ($l in $u.limits) { "{0}: {1}% ({2}) reset {3}" -f $l.kind, $l.percent, $l.severity, ([datetime]$l.resets_at).ToLocalTime() }
} catch { "ERROR: " + $_.Exception.Message }
```

> 전체 원본 JSON이 필요하면 `$u | ConvertTo-Json -Depth 5` (토큰은 응답에 포함되지 않으므로 출력해도 안전).

---

## 5. 응답 필드 해석

| 필드 | 의미 |
|---|---|
| `limits[]` | **판단에 쓸 핵심 목록.** 활성 한도만 정리되어 있음 |
| `limits[].kind` | `session` = 5시간 창, `weekly_all` = 주간 전체, 요금제에 따라 모델별 주간 한도가 추가될 수 있음 |
| `limits[].percent` | 사용률(정수 %) |
| `limits[].severity` | `normal` / 경고 단계 — 서버가 판단한 위험도 |
| `limits[].resets_at` | 초기화 시각 (**UTC**, 한국 시간은 +9시간) |
| `five_hour`, `seven_day` | 위와 같은 정보의 개별 필드 (`utilization` = 사용률) |
| `extra_usage` | 한도 초과 시 쓰는 추가 크레딧 상태 |
| `seven_day_breakdown.rows` | 주간 사용량의 출처 비율 (Claude Code / Chat 등) |
| 기타 이름이 난해한 필드 (`nimbus_quill` 등) | 내부 기능용. `null`이면 무시 |

---

## 6. 판단 규칙 (AI 행동 지침)

| 상태 | AI 행동 |
|---|---|
| `session` < 70% | 정상 진행 |
| `session` 70–90% | 대규모 작업(다수 서브에이전트, 전체 코드베이스 탐색) 전에 사용자에게 알림 |
| `session` ≥ 90% 또는 `severity` ≠ `normal` | 무거운 작업 시작 전 사용자 확인 필수, 초기화 시각 안내 |
| `weekly_all` ≥ 80% | 남은 기간·초기화 시각을 사용자에게 보고하고 작업 우선순위 조율 제안 |

- 사용자에게 보고할 때는 **사용률 + 초기화 시각(한국 시간)** 두 가지를 함께 알린다.
- 너무 자주 호출하지 않는다: 긴 작업의 시작 전, 또는 사용자가 요청할 때만 조회.

---

## 7. 보조 도구: ccusage (원인 분석용)

```bash
npx -y ccusage@latest daily --since 20260901   # 일별
npx -y ccusage@latest blocks --active          # 현재 5시간 블록
npx -y ccusage@latest session                  # 세션별
```

- 달러 금액은 **API 단가 기준 환산값**이며 구독 청구액이 아니다.
- 블록 시작 시각은 UTC 기준으로 표시될 수 있다.
- `blocks --active`의 "Projected Usage"는 초반 속도를 선형 외삽한 값이라 과대평가되기 쉽다.

---

## 8. 문제 해결

| 증상 | 원인 / 조치 |
|---|---|
| `401` / `403` | 토큰 만료 → Claude Code를 재시작하거나 `/login` 후 재시도 (사용자에게 요청) |
| 자격 증명 파일 없음 | macOS(Keychain 사용) 또는 API 키 로그인 상태 — API 키 사용자는 구독 한도가 없음 |
| `fetch is not defined` | Node 18 미만 → 방법 B 사용 또는 Node 업그레이드 |
| 응답 필드 누락/변경 | **비공식 엔드포인트**라 형식이 바뀔 수 있음 → 원본 JSON을 출력해 구조 재확인 |

---

## 9. 한계

- 비공식(문서화되지 않은) API이므로 예고 없이 바뀌거나 막힐 수 있다.
- 절대 한도는 제공되지 않으며, 모델별 가중치도 공개되어 있지 않다.
- 사용률은 정수 반올림 값이라 낮은 구간에서는 해상도가 낮다.
