전체 계획과 단계별 산출물, 의사결정 기록은 @docs/project-plan.md 참고.

# LLM Gateway 사이드 프로젝트

## 목적
두나무 DevOps Engineer(AI Gateway) 포지션 지원용 포트폴리오.
공고 핵심 업무인 "AWS Bedrock 기반 AI Gateway 설계·운영", "IaC(Terraform, Helm)", "MCP로 사내 자산 추상화"를 작게 재현한다.
실무에서 Kubernetes(EKS, CKA/CKAD), ArgoCD, Helm, Prometheus/Grafana 경험은 있으나 Terraform과 AI Gateway는 처음이다.

## 아키텍처
Claude Code, 영어 학습 앱(후순위) → LiteLLM 게이트웨이(kind 클러스터, Helm 배포) → Amazon Bedrock → Claude 모델

## 핵심 의사결정 (요약)
Bedrock + LiteLLM, 로컬 kind(운영 전환 시 EKS), 게이트웨이 자격증명은 CLI 발급 액세스 키 + Secret, 로컬 AWS 작업은 SSO, 테스트는 Haiku 위주.
상세 이유·대안·트레이드오프는 `docs/project-plan.md` 7장 의사결정 기록 표에만 기록한다(새 결정도 그 표에 추가).

## 진행 체크리스트
단계별 상세 내용과 산출물은 `docs/project-plan.md` 5장 참고.

### Phase 0. 환경 준비
- [x] 도구 설치: OrbStack(Docker), AWS CLI v2, Terraform, kubectl, kind, Helm, Claude Code
- [x] Git 저장소(main), .gitignore, gitleaks pre-commit hook, docs 기록 체계
- [ ] 첫 커밋 + GitHub 원격 저장소
- [ ] IAM Identity Center 작업용 사용자 + `aws configure sso`
- [ ] Bedrock 모델 접근 신청(승인 지연 대비 미리)

### Phase 1. AWS 기반 (Terraform)
- [ ] Bedrock 호출 성공(추론 프로필 ID 확인 후 Haiku로 테스트)
- [ ] 게이트웨이 전용 IAM 사용자 + 지정 모델만 호출 가능한 정책
- [ ] AWS Budgets 월 예산 + 50/80/100% 알림
- [ ] 액세스 키 CLI 발급, 최소 권한 검증(허용 모델 성공, 그 외 거부)

### Phase 2. 게이트웨이 배포와 운영 정책
- [ ] kind 클러스터 생성
- [ ] Helm으로 LiteLLM + PostgreSQL 배포, AWS 키·마스터 키는 Secret으로 분리
- [ ] team-a, team-b 키 발급, 팀별 월 예산·분당 요청 제한
- [ ] 한도 초과 차단 테스트

### Phase 3. 관측성
- [ ] Prometheus·Grafana 배포
- [ ] 대시보드(팀별 요청 수, 토큰, 비용, 에러율, 응답 시간)
- [ ] 알람(에러율, 응답 시간, 예산 소진율)

### Phase 4. 실사용 연결
- [ ] Claude Code를 게이트웨이 경유로 연결
- [ ] fallback, 작업별 모델 라우팅
- [ ] 장애 재현(한도 초과, 모델 오류, 게이트웨이 중단)

### Phase 5. MCP 서버
- [ ] 게이트웨이 사용량 또는 클러스터 상태 조회 MCP 서버(읽기 전용)

### Phase 6. 문서화
- [ ] README: 아키텍처 그림, 의사결정과 트레이드오프, 실행 방법, 운영 전환 시 고려사항

### Phase 7. 2주 운영
- [ ] 2주간 실사용하며 사용량·비용·에러·설정 변경 이유 기록

### Phase 8. (선택) 영어 학습 앱

## 작업 규칙
- 회사 코드나 회사 업무 트래픽은 이 게이트웨이로 보내지 않는다(개인 프로젝트·학습용만).
- 비밀 값(액세스 키, 게이트웨이 마스터 키)은 Git에 커밋하지 않는다.
- 로컬 AWS 작업은 SSO 임시 자격증명만 사용하고 루트 계정은 쓰지 않는다.
- Terraform은 처음이므로 코드를 작성할 때 각 리소스와 주요 줄의 의미를 설명해 준다.
- 작업이 끝날 때마다 `docs/`에 그날 한 일, 막힌 부분, 내린 결정을 기록한다(블로그 시리즈 재료).
- 비용 상한: AWS Budgets 및 LiteLLM 예산으로 월 $20 내외.
