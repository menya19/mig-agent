# EC2 마이그레이션 Agent 기술 스택 및 시스템 아키텍처 설계서

본 문서는 EC2 마이그레이션 실행 및 무결성 검증 Agent 시스템을 구축하기 위해 필요한 기술 스택, 라이브러리, 인프라 구성 요소를 정의합니다.

---

## 1. 전체 기술 스택 요약

| 구분 | 추천 기술 / 도구 | 선정 이유 |
| :--- | :--- | :--- |
| **Agent Core Framework** | LangGraph / LangChain | Complex Multi-step Workflow 제어 및 Stateful 에이전트 구현 최적화 |
| **LLM Engine** | Anthropic Claude 3.5 Sonnet / OpenAI GPT-4o | 뛰어난 코드/로그 해석 능력 및 Structured Output 지원 |
| **AWS SDK & Automation** | Boto3, AWS CLI, Paramiko / Fabric | AWS 인프라 제어(MGN, EC2, Route53) 및 에이전트리스 SSH 명령 실행 |
| **Infra Management** | AWS Systems Manager (SSM), Terraform | Agentless 방식의 안전한 명령 실행 및 EC2 프로비저닝 |
| **Backend Framework** | FastAPI, Python 3.11+ | 비동기 처리(Asyncio) 기반의 API 서버 구축 및 빠른 실행 속도 |
| **Database & Cache** | PostgreSQL, Redis | 작업 이력 관리, 상태 저장(Checkpointing) 및 메시지 큐 |
| **Monitoring & Logging** | AWS CloudWatch, OpenTelemetry, Slack SDK | 실시간 진행 상황 알림 및 통합 모니터링 |

---

## 2. 레이어별 상세 설계를 위한 구성 요소

### 1) Agent Framework & LLM Layer
* **LangGraph:** 마이그레이션 실행 ➔ 검증 ➔ 롤백 ➔ 완료로 이어지는 유한 상태 기계(Finite State Machine)의 흐름 제어.
* **Structured Outputs (Pydantic):** LLM이 검증 결과 및 RCA 보고서를 일정한 JSON 형태로 출력하도록 강제하여 시스템 파이프라인과 완벽한 연동 지원.

### 2) Infrastructure & AWS Integration Layer
* **AWS SDK (Boto3):**
  * `boto3.client('mgn')`: MGN 복제 상태 확인 및 Cutover 실행.
  * `boto3.client('route53')` / `boto3.client('elbv2')`: 트래픽 절체.
  * `boto3.client('ssm')`: Target EC2에 SSH 포트 개방 없이 안전한 Shell Script 투입.
* **Paramiko / Fabric:** SSM 에이전트 미설치 환경을 위한 Fallback SSH 커넥터.

### 3) Validation & Testing Tooling
* **System Checker Scripts:** Bash / Python 스크립트 모듈 (OS 파라미터, 프로세스, 파일 체크섬 수집용).
* **HTTP/API Tester:** `httpx` 또는 `aiohttp` 기반의 고성능 Async API Health Checker.

### 4) Storage & Message Queue Layer
* **Redis:** 마이그레이션 진행 중 수집되는 실시간 동기화율(%) 및 로그 스트리밍 샌드박스.
* **PostgreSQL:** 마이그레이션 수행 이력, Audit 로그, 시스템 Diff 결과 및 RCA 보고서 영구 저장.

---

## 3. 시스템 보안 및 접근 권한 설계

* **IAM Role (Least Privilege 원칙 적용):**
  * `AWSApplicationMigrationFullAccess` (또는 MGN 전용 세부 권한)
  * `AmazonSSMFullAccess` (Target EC2 내 검증 스크립트 실행용)
  * `AmazonRoute53FullAccess` / `ElasticLoadBalancingFullAccess` (Cutover용)
* **Secret Management:**
  * AWS Secrets Manager를 통해 Target/Source 서버의 SSH Key 및 API Access Key 보안 관리.