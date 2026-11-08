# EC2 마이그레이션 실행 및 무결성 검증 Agent 구현 방식 명세서

본 문서는 온프레미스/기존 클라우드 인프라의 AWS EC2 이전 시 마이그레이션 실행(Execution) 및 후속 무결성 검증(Validation)을 자동화하기 위한 AI Agent 시스템의 구체적인 구현 방식을 정의합니다.

---

## 1. 시스템 핵심 아키텍처 및 동작 메커니즘

AI Agent는 **Stateful Workflow Engine(예: LangGraph)** 기반으로 동작하며, 대형 언어 모델(LLM)이 각 단계의 판단, 로그 분석, Diff 해석을 담당하고, 실제 시스템 제어는 결정론적(Deterministic) Python/AWS SDK 커넥터 및 SSH 실행 엔진이 담당하는 구조입니다.

```
+-----------------------------------------------------------------------+
|                         Agent Controller (LLM)                        |
|   - Task Planning  - Intent Parsing  - Log Analysis & RCA  - Decision |
+-----------------------------------+-----------------------------------+
                                    |
            +-----------------------+-----------------------+
            |                                               |
            v                                               v
+-----------------------+                       +-----------------------+
|  Execution Orchestrator|                      | Integrity Validator   |
|  - AWS MGN API / Boto3|                       | - System/Port Diff    |
|  - DataSync / Rsync   |                       | - App Health Check    |
|  - Route53 / ALB Switch|                      | - Log Collector       |
+-----------------------+                       +-----------------------+
```

---

## 2. 모듈별 세부 구현 방식

### Module 1: 마이그레이션 실행 엔진 (Migration Execution Module)

1. **AWS MGN / API 연동 인터페이스**
   * AWS SDK(`boto3`)의 `mgn` 클라이언트를 제어하여 복제 상태 모니터링 및 Cutover 트리거.
   * 주요 API: `start_cutover()`, `get_replication_configuration()`, `finalize_cutover()`.
2. **데이터 동기화 및 Cutover 실행**
   * Cutover 시작 전 최종 Incremental Sync(Rsync/DataSync) 자동 실행.
   * Target 인스턴스의 Launch Template 적용 및 ENI(Elastic Network Interface) / Security Group 바인딩.
3. **네트워크 & 트래픽 Switching**
   * AWS Route 53 A/CNAME 레코드 갱신 또는 ALB Target Group의 IP/인스턴스 등록 교체.

### Module 2: 무결성 검증 엔진 (Post-Migration Integrity Validation Module)

검증 파이프라인은 Target 인스턴스가 완전히 부팅된 직후 SSH/SSM(AWS Systems Manager)을 통해 파라벌렐(Parallel)로 실행됩니다.

1. **System & OS Level Diff Engine**
   * **프로세스 및 서비스 상태:** Source와 Target에서 `systemctl list-units --state=running` 실행 후 비교.
   * **마운트 포인트 및 디스크 용량:** `df -hT` 및 `/etc/fstab` 엔트리 검증.
   * **환경 변수 및 커널 파라미터:** `sysctl -a`, `/etc/environment` 대조.
2. **Network & Security Level Inspector**
   * **포트 바인딩:** `netstat -tuln` 또는 `ss -tuln` 결과 비교 (Source에서 Listening 중이던 포트가 Target에서도 활성화되었는지 확인).
   * **인바운드/아웃바인딩 통신 검증:** Target 인스턴스에서 외부 연동 DB/API 서비스로의 소켓 연결 테스트(`nc -zv`).
3. **Application Level Synthetic Health Check**
   * REST API 엔드포인트에 대한 HTTP GET/POST 요청 수행.
   * 응답 코드(200 OK), Payload 형태, 응답 지연 시간(Latency) 비교.

### Module 3: 롤백 및 자동 원인 분석 엔진 (Auto-Rollback & RCA Engine)

1. **임계치 기반 자동 롤백 (Threshold Trigger)**
   * 검증 항목 중 **Critical(서비스 포트 미기동, 주요 API 실패, DB 마운트 실패 등)** 항목 감지 시 즉시 롤백 시퀀스 발동.
   * Route 53 / ALB를 이전 Source 인스턴스로 즉시 원복.
2. **RCA (Root Cause Analysis) 보고서 생성**
   * Agent가 Target 인스턴스에서 `/var/log/messages`, `journalctl -xe`, 애플리케이션 로그를 파싱.
   * LLM을 사용하여 파싱된 에러 스택 트레이스를 분석하고, 인간 언어로 요약된 원인 및 조치 가이드 생성.

---

## 3. 데이터 구조 및 상태 관리 명세 (State Schema)

LangGraph 기반의 Agent State 예시 명세:

```python
from typing import TypedDict, List, Dict, Optional

class MigrationState(TypedDict):
    source_ip: str
    target_instance_id: Optional[str]
    target_ip: Optional[str]
    migration_status: str  # PREPARED, CUTOVER_IN_PROGRESS, VALIDATING, COMPLETED, ROLLED_BACK
    system_diff_results: Dict[str, any]
    network_check_results: Dict[str, any]
    health_check_results: Dict[str, any]
    critical_failures: List[str]
    rca_report: Optional[str]
```