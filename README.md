# .github-automation

Jira ↔ GitHub 연동 및 PR 리마인드를 중앙에서 관리하는 자동화 레포입니다.
조직 내 모든 레포에 개별 설정 없이, 이 레포 하나로 자동화를 운영합니다.

---

## 아키텍처

```
Jira Automation (SubTask 생성/수정)
  │
  ▼  repository_dispatch
.github-automation (이 레포)
  │
  ├─ GitHub App Token 발급
  ├─ 대상 레포에 Issue 생성/수정 (gh CLI)
  └─ (실패 시 Slack 알림)
```

```
Schedule (매 2시간)
  │
  ▼
.github-automation (이 레포)
  │
  ├─ GitHub App Token 발급
  ├─ 조직 전체 Open PR 조회 (Search API)
  ├─ 비활성 PR 라벨 부착/제거
  └─ 통합 Slack 리마인드 발송
```

---

## 워크플로우

### 1. Jira → GitHub Issue 생성 (`jira-create-dispatch.yml`)

Jira에서 SubTask가 생성되면 대상 레포에 GitHub Issue를 자동 생성합니다.

| 항목 | 내용 |
|------|------|
| **트리거** | `repository_dispatch` (event_type: `create_issue`) |
| **동작** | Jira payload에서 repoName, title, body, labels, assignee 추출 → 중복 체크 → 라벨 생성 → Issue 생성 |
| **특징** | Jira wiki markup → Markdown 자동 변환, 중복 Issue 방지, 담당자 라벨 자동 생성 |

**Dispatch Payload:**

```json
{
  "event_type": "create_issue",
  "client_payload": {
    "repoName": "web-core",
    "issueKey": "DABOM-42",
    "title": "[DABOM-42] 로그인 오류 수정",
    "body": "Issue 본문 (Markdown)",
    "labels": ["feat", "backend"],
    "assignee": "홍길동"
  }
}
```

### 2. Jira → GitHub Issue 수정 (`jira-update-dispatch.yml`)

Jira SubTask의 summary 또는 description이 수정되면 GitHub Issue를 동기화합니다.

| 항목 | 내용 |
|------|------|
| **트리거** | `repository_dispatch` (event_type: `update_issue`) |
| **동작** | Jira Issue Key로 GitHub Issue 검색 → 제목 업데이트 + "상세 내용" 섹션만 교체 |
| **특징** | Issue body의 다른 섹션(기타 사항 등)은 유지하면서 상세 내용만 갱신 |

**Dispatch Payload:**

```json
{
  "event_type": "update_issue",
  "client_payload": {
    "repoName": "web-core",
    "issueKey": "DABOM-42",
    "summary": "수정된 제목",
    "description": "수정된 본문"
  }
}
```

### 3. PR 리마인드 (`pr-reminder.yml`)

조직 내 모든 레포의 Open PR을 조회하여, 비활성 PR을 리마인드합니다.

| 항목 | 내용 |
|------|------|
| **트리거** | 매 2시간 (cron) / 수동 실행 |
| **동작** | 조직 전체 Open PR 조회 → 4시간 이상 비활성 PR에 `remind-needed` 라벨 부착 → 활성화된 PR에서 라벨 제거 → Slack 통합 발송 |
| **제외 대상** | Draft PR, `wip` / `blocked` / `do-not-remind` 라벨이 있는 PR |

**Slack 메시지 예시:**

```
👀 PR Reminder

최근 업데이트가 없는 PR이 3건 있어요 ⏳

━━━━━━━━━━━━━━
📂 web-core
  - #12 로그인 버그 수정  👤 user1
  - #15 API 연동 개선     👤 user2

📂 backend-api
  - #8 DB 마이그레이션    👤 user3
━━━━━━━━━━━━━━
```

---

## Issue 템플릿

`templates/github/ISSUE_TEMPLATE/` 디렉토리에 조직 공용 Issue 템플릿이 포함되어 있습니다.
각 레포의 `.github/ISSUE_TEMPLATE/`에 복사하여 사용합니다.

| 파일 | 설명 |
|------|------|
| `issue-form.yml` | 이슈 생성 폼 (Jira 티켓, 브랜치명, 상세 내용, 기타 사항) |
| `config.yml` | blank issue 비활성화 설정 |

---

## 설정 방법

### 1. GitHub App 생성 및 설치

조직에 GitHub App을 생성하고 설치합니다. 필요 권한:

| 권한 | 수준 | 용도 |
|------|------|------|
| Issues | Read & Write | Issue 생성/수정 |
| Pull Requests | Read & Write | PR 라벨 부착/제거 (리마인드) |
| Metadata | Read | 레포 접근 기본 권한 |

### 2. Secrets & Variables 설정

이 레포의 **Settings → Secrets and variables → Actions** 에서 설정합니다.

**Secrets (필수):**

| 이름 | 설명 |
|------|------|
| `APP_ID` | GitHub App ID (숫자) |
| `PRIVATE_KEY` | GitHub App Private Key 전체 (`-----BEGIN RSA PRIVATE KEY-----` 포함) |

**Secrets (선택):**

| 이름 | 설명 |
|------|------|
| `SLACK_WEBHOOK_URL` | Slack Incoming Webhook URL (미설정 시 Slack 알림 스킵) |

**Variables (필수):**

| 이름 | 설명 |
|------|------|
| `ORG_NAME` | GitHub Organization 이름 (예: `da-bom`) |

### 3. Jira Automation 연동

Jira 프로젝트에서 Automation Rule을 생성하여, SubTask 생성/수정 시 이 레포에 `repository_dispatch`를 보내도록 설정합니다.

---

## 디렉토리 구조

```
.github-automation/
├── .github/
│   └── workflows/
│       ├── jira-create-dispatch.yml   ← Jira SubTask → GitHub Issue 생성
│       ├── jira-update-dispatch.yml   ← Jira SubTask 수정 → GitHub Issue 동기화
│       └── pr-reminder.yml            ← 조직 전체 PR 리마인드
└── templates/
    └── github/
        └── ISSUE_TEMPLATE/
            ├── issue-form.yml         ← 이슈 생성 폼 템플릿
            └── config.yml             ← blank issue 비활성화
```
