# 🤖 GHES Copilot Coder

![GHES Copilot Coder Demo](ghes.gif)

GitHub Enterprise Server 에서 Issue 기반으로 Copilot CLI 를 활용하여 **자동으로 코드를 생성하고 Pull Request 를 만드는** 프로젝트입니다.

> 이 프로젝트는 [ADO_CodingAgent](https://github.com/0GiS0/ADO_CodingAgent) 및 [benleane83/GHES_CodingAgent](https://github.com/benleane83/GHES_CodingAgent) 에서 영감을 받아 제작되었습니다.

## 📋 개요

Issue 에 `copilot` 라벨을 추가하면 GitHub Actions Workflow 가 트리거되어, Copilot CLI (`@github/copilot`) 가 Issue 내용을 기반으로 코드를 생성하고 PR 을 자동으로 생성합니다.

## �🎯 동작 방식

```
1. GitHub Issue 생성
       ↓
2. 'copilot' 라벨 추가
       ↓
3. Workflow 트리거
       ↓
4. Issue 라벨을 'in-progress' 로 변경 및 진행 코멘트
       ↓
5. Repository 체크아웃 및 작업 브랜치 생성 (copilot/{issue-number})
       ↓
6. Copilot CLI 로 코드 생성
       ↓
7. 변경사항 커밋 및 브랜치 푸시
       ↓
8. Pull Request 자동 생성
       ↓
9. Issue 에 완료 코멘트 및 'ready-for-review' 라벨 추가
```

---

## ⚙️ Prerequisites

### 1. GitHub Enterprise Server (GHES)

- **GHES 3.9 이상** 권장 (Node.js 20 런타임 지원)
- 현재 워크플로는 `actions/checkout@v3`, `actions/setup-node@v3` 등 **Node.js 16 기반 액션**을 사용하므로 3.9 미만에서도 동작합니다.
- Self-signed 인증서를 사용하는 경우 워크플로 내에서 `http.sslVerify false` 설정이 필요합니다 (이미 포함되어 있음).

### 2. GitHub Actions (Self-Hosted Runner)

GHES 에서 GitHub Actions 를 사용하려면 **Self-Hosted Runner** 가 필요합니다.

#### Runner 설치

1. GHES 리포지토리 → **Settings → Actions → Runners → New self-hosted runner**
2. 안내에 따라 Runner 를 설치합니다 (Linux/Ubuntu 권장)
3. Runner VM 에 다음 도구가 설치되어 있어야 합니다:

| 도구 | 용도 | 설치 확인 |
|------|------|-----------|
| `git` | 리포지토리 체크아웃 및 푸시 | `git --version` |
| `curl` | API 호출 (라벨, 코멘트, PR 생성) | `curl --version` |
| `jq` | JSON 파싱 | `jq --version` |
| `node` | Copilot CLI 실행 | `node --version` |
| `copilot` | 코드 생성 CLI | `copilot --version` |

> ⚠️ `gh` (GitHub CLI) 는 **사용하지 않습니다**. Self-Hosted Runner 에 미설치되어 있어도 무관합니다.

#### Runner 설정 시 주의사항

- Runner VM 에 다른 사용자의 Git Credential 이 캐시되어 있으면 push 시 403 에러가 발생할 수 있습니다.
- 워크플로에서 `persist-credentials: false` 및 `credential.helper=` 설정으로 이를 방지하고 있습니다.

### 3. 폐쇄망 네트워크 요구사항

Runner VM 에서 다음 내부 엔드포인트에 대한 **아웃바운드 HTTPS (443)** 접근이 필요합니다:

| 엔드포인트 | 용도 |
|----------|------|
| `<GHES 호스트>` | Git 작업 및 API 호출 |
| `<Azure AI Foundry Project Endpoint>` | Microsoft Foundry BYOK 모델 호출 |

> 폐쇄망에서는 `actions/setup-node` 로 Node.js 를 다운로드하지 않습니다. Runner 에 Node.js/Copilot CLI 를 사전 설치하세요. Foundry 토큰은 Azure CLI 없이 VM Instance Metadata Service(IMDS)에서 직접 발급합니다.

---

## 🔐 권한 설정

### 1. Repository 권한

리포지토리 → **Settings → Actions → General** 에서:

- **Workflow permissions**: `Read and write permissions` 선택
- **Allow GitHub Actions to create and approve pull requests**: ✅ 체크

> Organization 레벨에서도 동일한 설정이 필요할 수 있습니다.
> (`Organization → Settings → Actions → General`)

### 2. Secrets 설정

Repository → **Settings → Secrets and variables → Actions** 에서 등록합니다:

현재 워크플로는 Foundry 호출용 API Key secret 을 사용하지 않습니다.

> Self-Hosted Runner 의 Azure Managed Identity 로 `https://cognitiveservices.azure.com/` access token 을 발급합니다. `GH_TOKEN` (PAT) 은 **더 이상 필요하지 않습니다**. 워크플로는 `github.token` (GITHUB_TOKEN, 자동 발급) 을 사용하여 Git push, API 호출, PR 생성을 수행하되, Copilot CLI 프로세스에는 GitHub 토큰을 전달하지 않습니다.

### 3. Actions Variables 설정

Repository → **Settings → Secrets and variables → Actions → Variables** 에서 등록합니다:

| Variable | 필수 | 설명 |
|----------|------|------|
| `AZURE_AI_PROJECT_ENDPOINT` | ✅ | Azure AI Foundry Project Endpoint |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | ✅ | Azure AI Foundry 모델 배포명 |
| `AZURE_CLIENT_ID` | 선택 | User-assigned Managed Identity client ID. 미설정 시 system-assigned identity 사용 |
| `COPILOT_MODEL` | 선택 | Copilot CLI 에 전달할 모델명. 미설정 시 `AZURE_AI_MODEL_DEPLOYMENT_NAME` 사용 |
| `AZURE_OPENAI_API_VERSION` | 선택 | Foundry/Azure OpenAI API version 이 필요한 경우 사용 |
| `FOUNDRY_AUTH_MODE` | 선택 | Foundry 인증 방식. 이 워크플로는 `azure_identity` 만 허용하며 미설정 시 `azure_identity` |
| `FOUNDRY_WIRE_API` | 선택 | Foundry wire API. 미설정 시 `responses` |
| `FOUNDRY_SKIP_TLS_VERIFY` | 선택 | 내부 인증서 검증을 임시로 비활성화해야 할 때만 `true` |

### 4. Workflow Permissions

워크플로 파일에 다음 권한이 선언되어 있습니다:

```yaml
permissions:
  contents: write        # 브랜치 생성, 커밋, 푸시
  issues: write          # Issue 라벨 및 코멘트 수정
  pull-requests: write   # Pull Request 생성
```

---

## 🚀 사용 방법

### 1. Issue 생성

아래와 같은 형식으로 Issue 를 작성합니다:

```markdown
## 📋 Task Description
Python FastAPI 애플리케이션에 health check 엔드포인트를 추가합니다.

## 🎯 Acceptance Criteria
- [ ] /health 엔드포인트 구현
- [ ] JSON 형식으로 상태와 타임스탬프 반환
- [ ] requirements.txt 포함
```

### 2. `copilot` 라벨 추가

Issue 에 `copilot` 라벨을 추가하면 Workflow 가 자동으로 실행됩니다.

### 3. 결과 확인

- Workflow 가 완료되면 `copilot/{issue-number}` 브랜치에 코드가 생성됩니다.
- PR 이 자동으로 생성되며, Issue 에 완료 코멘트가 추가됩니다.
- **리뷰 후 머지**해 주세요. AI 가 생성한 코드이므로 반드시 사람이 확인해야 합니다.

---

## 🛠️ 설정 가능한 항목

워크플로 파일 (`.github/workflows/copilot-coder.yml`) 에서 다음 항목을 수정할 수 있습니다:

```yaml
env:
  MODEL: ${{ vars.COPILOT_MODEL || vars.AZURE_AI_MODEL_DEPLOYMENT_NAME || 'claude-haiku-4.5' }}
  AZURE_AI_PROJECT_ENDPOINT: ${{ vars.AZURE_AI_PROJECT_ENDPOINT }}
  AZURE_AI_MODEL_DEPLOYMENT_NAME: ${{ vars.AZURE_AI_MODEL_DEPLOYMENT_NAME }}
  FOUNDRY_AUTH_MODE: ${{ vars.FOUNDRY_AUTH_MODE || 'azure_identity' }}
  FOUNDRY_WIRE_API: ${{ vars.FOUNDRY_WIRE_API || 'responses' }}
```

## 📦 프로젝트 구조

```
.github/
├── copilot-instructions.md     # Copilot 코딩 가이드라인
└── workflows/
    └── copilot-coder.yml       # 메인 Workflow
```

## 🆘 트러블슈팅

| 문제 | 원인 | 해결 |
|------|------|------|
| `using: node20 is not supported` | GHES 버전이 낮음 | 액션을 `@v3` (node16) 으로 다운그레이드 |
| `exit code 127` | Runner 에 명령어 미설치 | `curl`, `jq`, `git`, `node`, `copilot` 설치 확인 |
| `Permission denied to <user>` | Git Credential 캐시 충돌 | `persist-credentials: false` 및 `credential.helper=` 사용 |
| `GitHub Actions is not permitted to create PR` | GITHUB_TOKEN PR 생성 차단 | Settings → Actions → General 에서 PR 생성 허용 |
| `Resource not accessible by PAT` | PAT 스코프 부족 | `repo`, `workflow` 스코프 포함 PAT 재발급 |
| TLS/SSL 에러 | Self-signed 인증서 | `git config http.sslVerify false` (이미 포함됨) |
| `AZURE_AI_PROJECT_ENDPOINT repository variable 이 설정되지 않았습니다` | Foundry endpoint 미등록 | Actions Variables 에 `AZURE_AI_PROJECT_ENDPOINT` 등록 |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME repository variable 이 설정되지 않았습니다` | Foundry 모델 배포명 미등록 | Actions Variables 에 `AZURE_AI_MODEL_DEPLOYMENT_NAME` 등록 |
| `Azure Managed Identity 로 Foundry access token 을 발급하지 못했습니다` | Runner Managed Identity 또는 Azure RBAC 미설정 | Runner VM/App Service identity 에 Foundry/Cognitive Services 호출 권한 부여 |
| `Runner 에 Copilot CLI 를 사전 설치해야 합니다` | Copilot CLI 미설치 | Runner 에 `@github/copilot` 또는 배포용 Copilot CLI 바이너리 사전 설치 |

---

## 🔒 폐쇄망 환경에서 사용하기 (Microsoft Foundry BYOK)

이 워크플로는 GitHub Copilot CLI 의 모델 호출을 Microsoft Foundry 로 라우팅하도록 구성합니다. API Key secret 은 사용하지 않고, Self-Hosted Runner 의 Azure Managed Identity 로 Foundry access token 을 발급합니다.

참고한 Foundry 변수명은 `agentic-devops-starter` 리포지토리의 배포 설정과 맞췄습니다:

| Variable | 설명 |
|----------|------|
| `AZURE_AI_PROJECT_ENDPOINT` | Azure AI Foundry Project Endpoint |
| `AZURE_AI_MODEL_DEPLOYMENT_NAME` | Foundry 모델 배포명 |
| `FOUNDRY_AUTH_MODE` | `azure_identity` 로 고정 |
| `FOUNDRY_WIRE_API` | `responses` 또는 `completions` |

### Managed Identity 권한

Runner 가 실행되는 Azure VM 또는 실행 환경에 system-assigned 또는 user-assigned Managed Identity 를 부여하고, Foundry/Azure AI 리소스에 모델 호출 권한을 할당해야 합니다. User-assigned Managed Identity 를 사용할 경우 Actions Variable `AZURE_CLIENT_ID` 에 client ID 를 등록합니다.

워크플로는 실행 중 Azure CLI 없이 VM Instance Metadata Service(IMDS)를 직접 호출해 인증합니다:

```bash
curl -H "Metadata: true" \
  "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://cognitiveservices.azure.com/"
```

발급된 token 은 Copilot CLI 프로세스에만 환경 변수로 주입되며, API Key secret 은 필요하지 않습니다.

### GitHub 인증과 Offline Mode

Microsoft Foundry BYOK 는 모델 호출 경로를 Foundry 로 바꾸는 설정입니다. 워크플로는 Copilot CLI 에 공식 provider 환경변수를 전달하고, `COPILOT_OFFLINE=true` 로 GitHub 인증, telemetry, web tools, GitHub MCP server, auto-update 를 비활성화합니다.

```bash
COPILOT_OFFLINE=true
COPILOT_PROVIDER_BASE_URL="${AZURE_AI_PROJECT_ENDPOINT}"
COPILOT_PROVIDER_TYPE=azure
COPILOT_PROVIDER_BEARER_TOKEN="<Managed Identity access token>"
COPILOT_PROVIDER_WIRE_API="${FOUNDRY_WIRE_API:-responses}"
COPILOT_PROVIDER_AZURE_API_VERSION="${AZURE_OPENAI_API_VERSION}"
COPILOT_PROVIDER_MODEL_ID="${COPILOT_MODEL:-$AZURE_AI_MODEL_DEPLOYMENT_NAME}"
COPILOT_PROVIDER_WIRE_MODEL="${AZURE_AI_MODEL_DEPLOYMENT_NAME}"
```

Runner 에 설치된 Copilot CLI 가 `COPILOT_OFFLINE`/`COPILOT_PROVIDER_*` 환경변수를 지원하지 않으면 워크플로가 사전에 실패하도록 확인합니다.

### 폐쇄망 환경 체크리스트

- [ ] Runner VM 에 Node.js/Copilot CLI 설치
- [ ] Runner VM 에 system-assigned 또는 user-assigned Managed Identity 연결
- [ ] Managed Identity 에 Foundry/Azure AI 모델 호출 권한 부여
- [ ] Actions Variables 에 `AZURE_AI_PROJECT_ENDPOINT`, `AZURE_AI_MODEL_DEPLOYMENT_NAME` 등록
- [ ] GHES, Foundry endpoint 간 네트워크 접근 확인

---

## 🏗️ 응용: 플랫폼화 (Reusable Workflow)

현재 `copilot-coder.yml` 은 단일 리포지토리에서 동작하지만, **Reusable Workflow** 로 전환하면 Organization 내 여러 리포지토리에서 공통으로 사용할 수 있습니다.

### 아키텍처

```
┌──────────────────────────────────────────────────────────────┐
│                   Your GHES Organization                     │
│                                                              │
│  ┌──────────────────────┐     ┌────────────────────────┐    │
│  │  Central Repository  │     │   Target Repository    │    │
│  │  (ghes-ghcp)         │     │   (my-project)         │    │
│  │                      │     │                        │    │
│  │  • Master workflow   │◄────│  • Caller workflow     │    │
│  │    (full logic)      │uses │    (~15 lines)         │    │
│  │  • copilot-          │     │                        │    │
│  │    instructions.md   │     └────────────────────────┘    │
│  └──────────────────────┘                                    │
│                               ┌────────────────────────┐    │
│                               │   Another Repository   │    │
│                               │   (another-project)    │    │
│                               │                        │    │
│                               │  • Caller workflow     │    │
│                               │    (~15 lines)         │    │
│                               └────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

### Step 1: Master Workflow 생성

Central Repository 에 `copilot-coder-master.yml` 을 생성합니다. 기존 워크플로에 `workflow_call` 트리거를 추가합니다:

```yaml
# .github/workflows/copilot-coder-master.yml
name: 🤖 Copilot Coder (Master)

on:
  # 이 리포에서 직접 실행
  issues:
    types: [labeled]
  # 다른 리포에서 호출 가능
  workflow_call:
    secrets:
      COPILOT_TOKEN:
        description: 'Token for GitHub Copilot API access'
        required: true

jobs:
  copilot-coder:
    if: contains(github.event.issue.labels.*.name, 'copilot')
    runs-on: self-hosted
    permissions:
      contents: write
      issues: write
      pull-requests: write

    # ... 기존 steps 동일 ...
```

### Step 2: Central Repository 설정

Central Repository → **Settings → Actions → General → Access** 에서:

- **"Accessible from repositories in the organization"** 선택

이렇게 해야 다른 리포에서 이 워크플로를 호출할 수 있습니다.

### Step 3: Target Repository 에 Caller Workflow 배포

각 Target Repository 에 아래와 같은 경량 Caller Workflow 만 배치합니다:

```yaml
# .github/workflows/copilot-coder.yml
name: 🤖 Copilot Coder

on:
  issues:
    types: [labeled]

jobs:
  copilot:
    uses: <org>/ghes-ghcp/.github/workflows/copilot-coder-master.yml@main
    secrets: inherit
    permissions:
      contents: write
      issues: write
      pull-requests: write
```

> `<org>` 은 GHES Organization 이름으로 교체합니다.

### 플랫폼화의 장점

| 장점 | 설명 |
|------|------|
| **중앙 관리** | Master Workflow 한 곳만 수정하면 모든 리포에 반영 |
| **최소 배포** | Target 리포에는 ~15줄 Caller Workflow 만 필요 |
| **일관성** | 모든 리포에서 동일한 코드 생성 프로세스 보장 |
| **버전 관리** | `@main` 대신 특정 태그 (`@v1.0.0`) 로 고정 가능 |
| **쉬운 롤아웃** | 새 리포에 Caller Workflow 파일 하나만 추가하면 완료 |

### 배포 자동화 (선택)

Target 리포에 Caller Workflow 를 자동으로 배포하는 스크립트를 작성할 수 있습니다:

```bash
#!/bin/bash
# deploy-to-repo.sh <ghes-host> <org> <repo> <gh-token>
GHES_HOST=$1; ORG=$2; REPO=$3; TOKEN=$4
API_URL="https://${GHES_HOST}/api/v3"

# Caller workflow 내용
WORKFLOW_CONTENT=$(base64 -w0 <<'EOF'
name: 🤖 Copilot Coder
on:
  issues:
    types: [labeled]
jobs:
  copilot:
    uses: ${ORG}/ghes-ghcp/.github/workflows/copilot-coder-master.yml@main
    secrets: inherit
    permissions:
      contents: write
      issues: write
      pull-requests: write
EOF
)

# API 로 파일 생성
curl -s -X PUT \
  -H "Authorization: token ${TOKEN}" \
  -H "Accept: application/vnd.github+json" \
  -d "{\"message\":\"feat: Add Copilot Coder workflow\",\"content\":\"${WORKFLOW_CONTENT}\"}" \
  "${API_URL}/repos/${ORG}/${REPO}/contents/.github/workflows/copilot-coder.yml"
```

---

## 🙏 Acknowledgments

- [benleane83/GHES_CodingAgent](https://github.com/benleane83/GHES_CodingAgent) — 이 프로젝트의 아이디어와 구현 방식에 영감을 준 원본 리포지토리
- [GitHub Copilot CLI](https://www.npmjs.com/package/@github/copilot) — AI 기반 코드 생성 CLI 도구
