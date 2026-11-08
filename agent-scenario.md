# EC2 마이그레이션 Agent 사용자 시나리오 (User Scenario)

본 문서는 엔지니어(사용자)가 마이그레이션 실행 및 무결성 검증 Agent를 활용하여 실제로 작업을 수행하는 E2E(End-to-End) 시나리오를 설명합니다.

---

## 1. 페르소나 및 배경

* **사용자:** 클라우드 인프라 엔지니어 (System Architect)
* **목표:** 온프레미스 레거시 서버 `10.0.1.50` (웹 및 API 서버)을 신규 AWS EC2 (`m7g.large`)로 이전하고 무결성을 검증한 후 전환 완료.
* **시간대:** 야간 작업 (서비스 영향 최소화 시간대)

---

## 2. 단계별 상세 시나리오

```
[ Step 1. 사전 확인 및 작업 명령 ]
                 │
                 ▼
[ Step 2. 마이그레이션 실행 (Cutover) ]
                 │
                 ▼
[ Step 3. Post-Migration 무결성 검증 ]
       ├── (성공) ──────► [ Step 5. 승인 및 완결 ]
       └── (실패) ──────► [ Step 4. 자동 롤백 & RCA ]
```

### Step 1: 사전 확인 및 작업 요청 (Pre-flight Check)
1. 엔지니어는 Chat/Web UI 인터페이스를 통해 Agent에게 작업을 지시합니다.
   > **엔지니어:** "Source 서버 `10.0.1.50`을 Target VPC의 신규 EC2로 마이그레이션 실행하고, 컷오버 직후 무결성 검증을 진행해 줘."
2. **Agent 동작:**
   * Source 서버 및 Target AWS 환경 권한 확인.
   * AWS MGN 동기화 상태 점검 (Replication Lag < 1초 확인).
   * 사전 검증 결과 문제없음을 사용자에게 보고 후 승인 요청.

### Step 2: 마이그레이션 실행 (Cutover Execution)
1. **Agent 동작:**
   * AWS MGN API를 호출하여 Source 서버의 최종 데이터 동기화(Final Sync) 트리거.
   * Target EC2 인스턴스 Launch 수행.
   * 인스턴스가 `Running` 상태가 되면 Target IP/ID 수집.
   * Route 53 DNS 레코드를 신규 EC2 IP로 교체하여 트래픽 Cutover 진행.
2. **알림:**
   > **Agent:** "Cutover가 성공적으로 완료되었습니다. Target EC2(`i-0abc1234def56789`) 부팅 완료. 즉시 무결성 검증(Validation)을 시작하겠습니다."

### Step 3: 무결성 검증 수행 (Post-Migration Validation)
Agent가 SSM/SSH를 통해 Target EC2에 접속 후 3단계 정밀 검증을 자동 수행합니다.

1. **System Diff 검증:**
   * Source vs Target 간 `systemctl` 서비스 비교 ➔ Nginx, Node.js 서비스 동작 확인.
   * 마운트 디렉토리 (`/mnt/data`) 해시 및 디스크 용량 검증 ➔ 정상.
2. **Network & Security 검증:**
   * 필수 포트 (`80`, `443`, `8080`) Listening 상태 검증 ➔ 정상.
   * 외부 RDS 데이터베이스 통신 연결 테스트 ➔ 정상.
3. **Application Synthetic Test:**
   * `/api/v1/health` 및 주요 GET/POST API 엔드포인트 호출 ➔ HTTP 200 OK 수신.

### Step 4 (예외 처리 시나리오): 자동 롤백 및 RCA 보고 (Auto-Rollback & RCA)
* *상황 조건: 만약 Target EC2의 환경변수 누락으로 `/api/v1/health`가 HTTP 500 에러를 반환하는 경우*

1. **Agent 동작:**
   * 검증 실패(Critical Threshold 초과) 감지.
   * 즉시 Route 53 DNS를 기존 Source 서버 IP (`10.0.1.50`)로 원복(Rollback).
   * Target EC2의 `/var/log/syslog` 및 App 에러 로그 수집.
2. **RCA 보고서 제출:**
   > **Agent:** "무결성 검증 실패로 인해 트래픽을 기존 서버로 자동 롤백했습니다."
   > * **실패 원인:** Target EC2의 `/etc/environment` 내 `DB_HOST` 환경변수 누락으로 인한 DB Connection Refused.
   > * **권고 사항:** Launch Template에 환경변수 주입 스크립트를 추가 후 마이그레이션을 재시도하세요.

### Step 5: 작업 완료 및 보고서 발행 (Success Scenario)
모든 검증 단계가 통과한 경우:

1. **Agent:**
   * 검증 요약 보고서를 Slack / Web Console로 전송.
   > **Agent:** "모든 무결성 검증 테스트(System 100%, Network 100%, App 100%)를 통과했습니다. 기존 Source 서버를 중지할까요?"
2. **엔지니어:** "응, 기존 Source 서버 정지하고 마이그레이션 종료해."
3. **Agent:** Source 서버 Graceful Shutdown 및 최종 컷오버 리포트 발행 후 작업 마감.