# LLM Gateway 프로젝트 계획서

## 1. 배경

두나무 DevOps Engineer 공고(AI Gateway 및 플랫폼 운영)에 지원하기 위한 포트폴리오 프로젝트다.

공고의 핵심 요구사항과 현재 경험을 비교하면 다음과 같다.

| 공고 요구사항 | 현재 경험 | 이 프로젝트에서 채울 부분 |
|---|---|---|
| AI Gateway 구축·운영 (Bedrock, Vertex, Foundry) | 없음 | Bedrock 앞단 LiteLLM 게이트웨이 구축과 운영 |
| MCP/CLI로 사내 자산 추상화 | Jira·Confluence MCP 연동, Claude Code 플러그인 팀 배포(30명) | MCP 서버 직접 개발 |
| Claude Code 등 AI 도구 활용 | 실무 사용 중 | 게이트웨이 경유 실사용 데이터 |
| Kubernetes 운영 | EKS 실무, CKA/CKAD | Helm 기반 배포 구성 |
| IaC (Terraform, Helm, Packer) | Helm 실무 / Terraform 없음 | Terraform으로 AWS 리소스 관리 |
| 보안 관점의 인프라 고민 (우대) | CloudFront+WAF | 최소 권한 IAM, SSO, 비밀 값 관리 |
| FinOps | 약함 | 팀별 예산, 비용 알림, 모델 라우팅 |
| 모니터링 | Prometheus·Grafana 실무 | 게이트웨이 사용량·비용·에러 대시보드 |

## 2. 목표

**한 줄 목표:** Amazon Bedrock 앞에 LiteLLM 게이트웨이를 두고, 팀별 키·예산·사용량 제한과 모니터링을 적용한 뒤, 실제 Claude Code 트래픽을 통과시켜 2주간 운영한다.

**보여주고 싶은 것**
- AI 플랫폼을 운영 관점(권한, 비용, 장애, 감사)에서 설계할 수 있다.
- 검증된 오픈소스를 골라 운영 가능한 형태로 구성할 수 있다.
- 처음 쓰는 도구(Terraform)도 근거를 가지고 학습해 적용할 수 있다.
- 만들고 끝이 아니라 직접 운영하며 문제를 발견하고 개선한다.

**사이드 프로젝트의 한계를 정직하게 표현한다.** 이 프로젝트는 "운영 경력"이 아니라 "실무에서 아직 해보지 못한 영역을 이해하고 직접 구성해 본 경험"으로 소개한다.

## 3. 아키텍처

```
[사용자 A: Claude Code]  ──키 A──┐
                                 ├──> [LiteLLM Gateway] ──IAM(최소 권한)──> [Amazon Bedrock] ──> Claude Haiku / Sonnet
[사용자 B: 영어 학습 앱] ──키 B──┘          │
                                            ├── PostgreSQL (키, 팀, 사용량 저장)
                                            └── Prometheus → Grafana (대시보드, 알람)

[직접 만든 MCP 서버] ── Claude Code에서 클러스터 상태 / 게이트웨이 사용량 조회

전부 로컬 kind 클러스터 위에 Helm으로 배포
AWS 리소스(IAM, Budgets)는 Terraform으로 관리
```

## 4. 범위

**포함**
- Terraform: 게이트웨이 전용 최소 권한 IAM, AWS Budgets 예산 알림
- kind 클러스터 + Helm 배포: LiteLLM, PostgreSQL, Prometheus, Grafana
- 운영 정책: 팀별 키, 월 예산, 분당 요청 제한, 모델 fallback, 작업별 모델 라우팅
- Claude Code를 게이트웨이 경유로 연결
- MCP 서버 직접 개발
- README와 블로그 시리즈
- 2주 실사용 운영 기록

**제외 (README에 "운영 전환 시 고려사항"으로 기록)**
- EKS 배포: 비용 문제로 로컬 kind 사용. 전환 시 IRSA/Pod Identity로 자격증명 교체
- 개인정보·민감정보 마스킹: 2단계 과제
- 멀티 클라우드(Vertex, Foundry) 연동: 구조상 확장 가능하다는 점만 설명
- 고가용성 구성 (다중 레플리카, 외부 관리형 DB)

## 5. 단계별 계획

일차가 아니라 단계 기준으로 진행한다. 퇴근 후 작업이라 단계당 기간은 유동적이다.

### Phase 0. 환경 준비
- 도구 설치: OrbStack(Docker), AWS CLI, Terraform, kubectl, kind, Helm, Claude Code
- IAM Identity Center 작업용 사용자 + `aws configure sso`
- Git 저장소, .gitignore, docs 기록 체계
- **산출물:** 동작하는 로컬 환경, 첫 커밋

### Phase 1. AWS 기반 (Terraform)
- Bedrock 모델 접근 확인, 추론 프로필 ID로 Haiku 호출 성공
- Terraform 기초 학습 (provider, resource, plan/apply, state)
- 게이트웨이 전용 IAM 사용자 + 지정 모델만 호출 가능한 정책
- AWS Budgets 월 예산과 50/80/100% 알림
- 액세스 키는 CLI로 발급 (state에 비밀 값이 남지 않도록)
- **산출물:** `terraform/` 코드, 최소 권한 검증 기록 (허용 모델은 성공, 그 외는 거부)

### Phase 2. 게이트웨이 배포와 운영 정책
- kind 클러스터 생성
- Helm으로 LiteLLM + PostgreSQL 배포, AWS 키와 마스터 키는 Secret으로 분리
- team-a, team-b 키 발급, 팀별 월 예산과 분당 요청 제한
- 한도 초과 시 실제 차단되는지 테스트
- **산출물:** `helm/` values, 정책 설정, 한도 초과 테스트 결과

### Phase 3. 관측성
- Prometheus, Grafana Helm 배포
- 대시보드: 팀별 요청 수, 토큰 사용량, 비용, 에러율, 응답 시간
- 알람: 에러율, 응답 시간, 예산 소진율
- LiteLLM의 메트릭 제공 범위는 버전·라이선스별로 다르므로 시작 시 문서 확인
- **산출물:** 대시보드 JSON, 알람 규칙, 스크린샷

### Phase 4. 실사용 연결
- Claude Code를 게이트웨이 경유로 연결 (Pro 계정과 전환 가능하게)
- 모델 장애 대비 fallback (대체 모델 또는 리전)
- 작업 종류별 라우팅 (가벼운 작업은 Haiku)
- 장애 상황 재현: 한도 초과, 모델 오류, 게이트웨이 중단
- **산출물:** 연결 설정 가이드, 장애 재현 기록

### Phase 5. MCP 서버 개발
- 게이트웨이 사용량 또는 kind 클러스터 상태를 조회하는 MCP 서버 개발
- Claude Code에 연결해 자연어로 조회
- 읽기 전용 권한으로 제한하고 그 이유를 기록
- **산출물:** `mcp-server/` 코드와 사용 예시

### Phase 6. 문서화
- README: 아키텍처 그림, 기술 선택 이유, 운영 관점 의사결정과 트레이드오프, 실행 방법, 운영 전환 시 고려사항
- **산출물:** 지원서에 첨부할 수 있는 수준의 README

### Phase 7. 2주 운영
- 사이드 프로젝트·개인 학습 작업만 게이트웨이 경유로 사용
- 기록 항목: 일별 요청 수·토큰·비용, 에러와 원인, 설정 변경과 이유, "실제 회사라면 더 필요한 것"
- **산출물:** README의 "운영하며 배운 점" 섹션, 블로그 회고 글

### Phase 8. (선택) 영어 학습 앱
- 영작 → 자연스러운 표현 추천 → 유사 문장 연습 → 피드백
- 게이트웨이의 두 번째 사용자로 별도 키와 예산 적용, Haiku 라우팅
- 멀티 테넌트 상황 재현 용도

## 6. 저장소 구조 (예정)

```
llm-gateway/
├── CLAUDE.md
├── README.md
├── .gitignore
├── docs/            # 계획서, 작업 기록, 의사결정 기록 (블로그 재료)
├── terraform/       # IAM, Budgets
├── helm/            # LiteLLM, PostgreSQL, 모니터링 values
├── k8s/             # kind 설정, 기타 매니페스트
├── dashboards/      # Grafana 대시보드 JSON
├── mcp-server/      # 직접 만든 MCP 서버
└── scripts/         # 키 발급, 테스트 스크립트
```

## 7. 의사결정 기록

| 결정 | 이유 | 대안과 트레이드오프 |
|---|---|---|
| 모델 공급처로 Bedrock | IAM 인증과 모델 단위 최소 권한, CloudTrail 감사, AWS 청구 체계 통합, 공고 환경(AWS)과 일치 | Anthropic API 직접 호출은 간단하지만 키가 분산되고 감사·비용 관리가 분리됨 |
| 게이트웨이로 LiteLLM | 검증된 오픈소스, 팀별 키·예산·제한·fallback 기본 제공. 시간을 운영 영역에 집중 | 직접 구현은 인증·집계·장애 처리를 모두 떠안음. 다른 게이트웨이 비교는 Phase 2에서 짧게 기록 |
| Bedrock 위에 게이트웨이를 별도로 둠 | IAM만으로는 팀별 예산·사용량 제한, 팀 단위 비용 집계, fallback이 어렵고, 사용자에게 AWS 자격증명을 직접 배포하지 않기 위해 | 구성 요소가 늘어 운영 대상이 하나 더 생김 |
| EKS 대신 로컬 kind | 클러스터 상시 비용 회피. EKS 실무 경험은 이미 있음 | 24시간 운영 불가, IRSA 사용 불가 |
| 게이트웨이 자격증명은 액세스 키 + Secret | kind에서는 IRSA/Pod Identity 불가 | 장기 키 사용. 권한 최소화, CLI 발급으로 state 노출 방지, EKS 전환 시 교체 |
| 로컬 작업은 IAM Identity Center SSO | 장기 키가 파일에 남지 않음, 세션 만료, MFA 적용 | 초기 설정이 액세스 키보다 번거로움 |
| Docker 런타임으로 OrbStack | Mac에서 가볍고 빠름 (배터리·메모리 절약) | Docker Desktop이 더 범용적 |
| 저장소를 GitHub에 공개 | 포트폴리오로 코드·의사결정·작업 기록을 근거로 제시 | 비밀 값 유출 위험 → `.gitignore`(state, tfvars, .env, kubeconfig) + gitleaks pre-commit hook으로 차단 |
| 비밀 값 검사는 gitleaks + `.githooks/` | hook 스크립트를 저장소에 포함해 버전 관리, 별도 프레임워크(pre-commit) 의존 없음 | 클론 후 `git config core.hooksPath .githooks`를 직접 실행해야 함 |
| 저장소 이름 `bedrock-llm-gateway-poc` | 공고 키워드(Bedrock)를 드러내고 `-poc`로 프로젝트 성격을 정직하게 표현 | `-poc`가 완성도가 낮아 보일 수 있음 |
| GitHub Push protection 추가 | 로컬 hook은 설정 누락 시 우회되므로 서버 쪽 이중 방어 | 없음(public 저장소 무료) |

새 결정이 생기면 이 표에 추가한다.

## 8. 비용 계획

- 인프라: 로컬 kind라 0원
- Bedrock: 쓴 토큰만큼 과금. 테스트는 Haiku 위주
- 상한: AWS Budgets 알림과 LiteLLM 예산으로 월 $20 내외
- 신규 계정 크레딧이 있다면 우선 소진
- 회사 코드나 회사 업무는 게이트웨이로 보내지 않음

## 9. 블로그 시리즈 (velog)

| 편 | 주제 | 대응 단계 |
|---|---|---|
| 1 | 왜 AI Gateway인가, 그리고 왜 Bedrock + LiteLLM인가 | 계획 |
| 2 | 처음 써보는 Terraform으로 최소 권한 IAM 만들기 | Phase 0~1 |
| 3 | kind 위에 LLM 게이트웨이 올리기, 팀별 예산과 한도 걸기 | Phase 2 |
| 4 | LLM 게이트웨이 모니터링: 토큰과 비용을 대시보드로 | Phase 3 |
| 5 | Claude Code를 내 게이트웨이로 연결하고 장애 재현해보기 | Phase 4 |
| 6 | MCP 서버 직접 만들기 | Phase 5 |
| 7 | 2주 운영 회고 | Phase 7 |

글은 `docs/` 작업 기록을 바탕으로 쓰되, 막힌 이유와 판단 근거는 직접 쓴다.

## 10. 지원서 활용

- **포트폴리오:** GitHub 저장소 링크 + 블로그 시리즈 링크
- **경력기술서:** 실무 경험(플러그인 마켓플레이스, MCP 연동)은 "AI 플랫폼 운영 경험"에, 이 프로젝트는 "개인 프로젝트"로 분리해서 기재
- **면접 대비:** 의사결정 기록 표와 2주 운영 기록이 답변 재료

지원 시점: Phase 6까지 완료하고 2주 운영 기록이 쌓인 뒤. 블로그 4~5편 이상 발행 상태 목표.

## 11. 리스크

| 리스크 | 대응 |
|---|---|
| Bedrock 모델 접근이나 사용 목적 양식에서 지연 | Phase 0에서 미리 신청 |
| Terraform 학습에 예상보다 시간 소요 | Phase 1 범위를 IAM과 Budgets로 제한 |
| LiteLLM 설정, 버전 차이로 막힘 | 공식 문서 기준 버전 고정, 막힌 내용은 블로그 소재로 기록 |
| Helm v4와 v3 문서 차이 | 명령어 차이 발생 시 기록 |
| 퇴근 후 작업이라 흐름이 끊김 | CLAUDE.md 체크리스트와 docs 기록으로 재개 지점 관리 |
| 비용 초과 | Budgets 알림 + LiteLLM 예산 이중 장치 |
