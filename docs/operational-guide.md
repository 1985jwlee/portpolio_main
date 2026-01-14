# 운영 가이드

[← 메인으로 돌아가기](../README.md)

---

## 📋 목차

1. [운영 철학](#운영-철학)
2. [장애 시나리오별 대응](#장애-시나리오별-대응)
3. [무중단 배포](#무중단-배포)
4. [모니터링](#모니터링)
5. [재해 복구 훈련](#재해-복구-훈련)

---

## 운영 철학

### 핵심 원칙

> **"장애가 나지 않는 것"보다 "장애가 나도 예측 가능하게 동작하는 것"**

이 설계는 다음을 가정합니다:
- 장애는 언제나 발생한다
- 모든 시스템은 언젠가 다운된다
- 예측 가능한 장애는 통제 가능하다

### 설계 목표

```
✓ 게임플레이는 백엔드 장애에 영향받지 않음
✓ 장애는 국소화되고 전파되지 않음
✓ 복구는 자동화되거나 단순한 절차
✓ 운영자가 상황을 즉시 파악 가능
```

---

## 장애 시나리오별 대응

### 장애 영향도 매트릭스

| 장애 대상 | 게임플레이 | 기록 | 운영 API | 복구 난이도 |
|-----------|------------|------|----------|-------------|
| 게임 서버 | 🔴 중단 | 🟡 일시 중단 | 🟢 정상 | 낮음 |
| Redis | 🟡 순간 지연 | 🟢 정상 | 🟢 정상 | 낮음 |
| MongoDB | 🟢 정상 | 🟢 정상 | 🟢 정상 | 낮음 |
| Kafka | 🟢 정상 | 🟡 일시 중단 | 🟢 정상 | 중간 |
| MySQL | 🟢 정상 | 🟡 일시 중단 | 🔴 일부 실패 | 중간 |
| 플랫폼 서버 | 🟢 정상 | 🟡 일시 중단 | 🔴 중단 | 낮음 |

---

### 시나리오 1: 게임 서버 크래시

#### 증상
```
- 특정 Zone 플레이어 전원 연결 끊김
- 게임 서버 프로세스 다운
- Health Check 실패
```

#### 자동 복구 프로세스

```
[ 장애 발생 ]
    ↓
[ Health Check 실패 감지 ]
    ↓
[ 새 게임 서버 인스턴스 시작 ]
    ↓
[ Redis Hot Snapshot 로드 ]
    ↓ (수초)
[ 플레이어 세션 복원 ]
    ↓
[ 게임 재개 ]
```

#### 복구 시간
- **RTO (Recovery Time Objective)**: 10초
- **RPO (Recovery Point Objective)**: 5~10초 (마지막 스냅샷 이후)

#### 데이터 손실
- 최대 5~10초 (마지막 Hot Snapshot 이후)
- Kafka 이벤트는 유실 없음 (나중에 재처리 가능)

#### 수동 대응
```bash
# 1. 로그 확인
tail -f /var/log/game-server/crash.log

# 2. 코어 덤프 수집
gdb game-server core.dump

# 3. 알람 확인
- Slack/PagerDuty 알림 확인
- 영향받은 플레이어 수 파악

# 4. 재발 방지
- 크래시 원인 분석
- 핫픽스 배포 계획
```

---

### 시나리오 2: Redis 장애

#### 증상
```
- 게임 서버 스냅샷 저장 실패
- Session 관리 에러
- Rate Limiting 비활성화
```

#### 자동 복구 프로세스

```
[ Redis 다운 감지 ]
    ↓
[ MongoDB Cold Snapshot으로 전환 ]
    ↓
[ 게임 서버: 일시적 성능 저하 ]
    ↓ (수분)
[ Redis 복구 ]
    ↓
[ Hot Snapshot 재개 ]
```

#### 영향
- **게임플레이**: 일시적 지연 (수백ms)
- **기록**: 영향 없음 (Kafka로 처리)
- **세션**: 일시적 불안정

#### 수동 대응
```bash
# 1. Redis 상태 확인
redis-cli ping

# 2. Failover 확인
redis-cli sentinel masters

# 3. 백업 복구 (필요 시)
redis-cli --rdb /var/redis/dump.rdb

# 4. 재시작
systemctl restart redis
```

---

### 시나리오 3: Kafka 장애

#### 증상
```
- 이벤트 발행 실패 로그 증가
- Consumer Lag 급증
- 플랫폼 서버 이벤트 처리 중단
```

#### 자동 복구 프로세스

```
[ Kafka 다운 ]
    ↓
[ 게임 서버: 영향 없음 ]
    ↓
[ 이벤트는 메모리 버퍼에 임시 저장 ]
    ↓
[ Kafka 복구 ]
    ↓
[ 버퍼링된 이벤트 일괄 전송 ]
```

#### 영향
- **게임플레이**: 영향 없음
- **기록**: 일시 중단 (복구 후 재개)
- **운영 API**: 영향 없음

#### 버퍼 전략
```csharp
public class EventBuffer
{
    private Queue _buffer = new();
    private const int MaxBufferSize = 10000;
    
    public void BufferEvent(DomainEvent evt)
    {
        if (_buffer.Count >= MaxBufferSize)
        {
            // 오래된 이벤트 제거 (FIFO)
            _buffer.Dequeue();
            Logger.Warn("Event buffer full, dropping old events");
        }
        
        _buffer.Enqueue(evt);
    }
    
    public async Task FlushBuffer()
    {
        while (_buffer.Count > 0)
        {
            var evt = _buffer.Dequeue();
            await kafka.PublishAsync(evt);
        }
    }
}
```

#### 수동 대응
```bash
# 1. Kafka 클러스터 상태 확인
kafka-broker-api-versions.sh --bootstrap-server localhost:9092

# 2. Topic 상태 확인
kafka-topics.sh --describe --topic game.events.player

# 3. Consumer Group 확인
kafka-consumer-groups.sh --describe --group platform-server-group

# 4. 재시작
systemctl restart kafka
```

---

### 시나리오 4: MySQL 장애

#### 증상
```
- 플랫폼 서버 DB 쿼리 실패
- 운영 API 에러 증가
- 이벤트 영속화 실패
```

#### 영향
- **게임플레이**: 영향 없음
- **기록**: 일시 중단 (Kafka에 이벤트 누적)
- **운영 API**: 일부 실패

#### 자동 복구 프로세스

```
[ MySQL 다운 ]
    ↓
[ 플랫폼 서버: 읽기 요청 실패 ]
    ↓
[ 이벤트는 Kafka에 누적 ]
    ↓
[ MySQL 복구 ]
    ↓
[ 누적된 이벤트 자동 처리 ]
```

#### 수동 대응
```bash
# 1. MySQL 상태 확인
systemctl status mysql

# 2. Replication 확인
mysql> SHOW SLAVE STATUS\G

# 3. 백업 복구 (필요 시)
mysql < backup.sql

# 4. 재시작
systemctl restart mysql
```

---

### 시나리오 5: 플랫폼 서버 장애

#### 증상
```
- 운영 API 응답 없음
- Kafka Consumer 중단
- 관리자 대시보드 접근 불가
```

#### 영향
- **게임플레이**: 영향 없음
- **기록**: 일시 중단
- **운영 API**: 완전 중단

#### 자동 복구 프로세스

```
[ 플랫폼 서버 다운 ]
    ↓
[ 게임 서버: 정상 동작 ]
    ↓
[ Kafka 이벤트 누적 ]
    ↓
[ 플랫폼 서버 재시작 ]
    ↓
[ Consumer Offset부터 재개 ]
    ↓
[ 누적 이벤트 순차 처리 ]
```

---

## 무중단 배포

### Rolling Update 프로세스

```
1. 새 버전 게임 서버 기동
   ↓
2. Health Check 통과 확인
   ↓
3. Load Balancer에 신규 서버 등록
   ↓
4. 신규 접속자만 새 서버 할당
   ↓
5. 기존 서버는 플레이어 0명까지 대기
   ↓
6. 스냅샷 저장
   ↓
7. Graceful Shutdown
```

### 배포 체크리스트

#### 배포 전
```
☐ 버전 호환성 검증
☐ 프로토콜 변경사항 확인
☐ 부하 테스트 통과 확인
☐ Rollback 계획 수립
☐ 배포 시간대 확인 (저사용 시간)
☐ 운영팀 대기 확인
```

#### 배포 중
```
☐ 새 버전 서버 Health Check
☐ 신규 접속 모니터링
☐ 에러 로그 실시간 확인
☐ 메모리/CPU 사용률 모니터링
☐ GameLoop Tick 지연 확인
☐ 플레이어 이탈률 확인
```

#### 배포 후
```
☐ 구버전 서버 완전 종료 확인
☐ 스냅샷 저장 성공 확인
☐ 24시간 모니터링
☐ 장애 대응 준비 태세 유지
☐ Rollback 대기
```

### Rollback 절차

```bash
# 1. 신규 버전 서버 제거
kubectl scale deployment game-server-v2 --replicas=0

# 2. 구버전 서버 복원
kubectl scale deployment game-server-v1 --replicas=3

# 3. 확인
kubectl get pods -l app=game-server

# 4. 로드밸런서 업데이트
kubectl rollout undo deployment/game-server
```

---

## 모니터링

### 시스템 메트릭

#### 게임 서버
```
✓ Zone별 동접자 수 (CCU)
✓ CPU/Memory 사용률
✓ GameLoop Tick 지연 (Target: < 50ms)
✓ Command Queue 길이
✓ 초당 패킷 수 (In/Out)
✓ 평균 RTT (Round-Trip Time)
✓ 패킷 손실률
```

#### 플랫폼 서버
```
✓ Kafka Consumer Lag
✓ 이벤트 처리 속도
✓ DB 커넥션 풀 사용률
✓ API 응답 시간
✓ 에러율
```

### 경보 조건

#### 🚨 Critical (즉시 대응)
```
- 게임 서버 다운
- Tick > 100ms 지속 30초
- CCU 급락 (30% 이상)
- Error Rate > 5%
- DB 커넥션 풀 고갈
```

#### ⚠️ Warning (확인 필요)
```
- Command Queue > 1000
- Snapshot 실패 연속 3회
- Kafka Consumer Lag > 10000
- Tick > 70ms 지속 1분
- CPU > 80% 지속 5분
```

#### ℹ️ Info (관찰)
```
- 신규 버전 배포
- 정기 점검 시작/종료
- 스냅샷 저장 완료
- CCU 임계값 도달 (확장 검토)
```

### 대시보드

#### 실시간 모니터링
```
┌─────────────────────────────────────┐
│ Game Server Health                  │
├─────────────────────────────────────┤
│ Zone 1: 243 CCU | Tick: 45ms | ✅   │
│ Zone 2: 189 CCU | Tick: 52ms | ✅   │
│ Zone 3: 301 CCU | Tick: 48ms | ✅   │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Kafka Status                        │
├─────────────────────────────────────┤
│ Topic: game.events.player           │
│ Messages/sec: 1,234                 │
│ Consumer Lag: 234                   │
└─────────────────────────────────────┘
```

---

## 재해 복구 훈련

### DR Drill 계획

**주기**: 분기별 1회

**목표**:
- RTO/RPO 실제 측정
- 복구 절차 검증
- 운영팀 훈련

### 시나리오

#### 시나리오 1: Redis 강제 중단
```bash
# 1. Redis 중단
systemctl stop redis

# 2. 복구 시간 측정 시작
start_timer

# 3. MongoDB Snapshot 로드 확인
# 4. 게임 서버 정상 동작 확인
# 5. 복구 시간 기록

# 예상 RTO: 2~3분
```

#### 시나리오 2: 전체 Zone 다운
```bash
# 1. 전체 게임 서버 중단
kubectl delete deployment game-server

# 2. 스냅샷 복구
# 3. 새 서버 기동
# 4. 플레이어 재접속 확인

# 예상 RTO: 5~10분
```

#### 시나리오 3: 데이터센터 장애
```bash
# 1. 전체 인프라 중단 (시뮬레이션)
# 2. DR 사이트로 전환
# 3. 백업에서 복구
# 4. 서비스 재개

# 예상 RTO: 30분~1시간
```

### DR Drill 체크리스트

```
배포 전:
☐ 백업 최신 상태 확인
☐ Runbook 검토
☐ 운영팀 교육
☐ 알람 테스트

실행 중:
☐ 시간 기록
☐ 문제점 기록
☐ 의사결정 기록
☐ 커뮤니케이션 기록

실행 후:
☐ RTO/RPO 분석
☐ 개선사항 도출
☐ Runbook 업데이트
☐ 다음 Drill 계획
```

---

## 운영 Runbook

### 일일 점검
```
08:00 - 시스템 Health Check
12:00 - 로그 확인
18:00 - 백업 확인
22:00 - 성능 메트릭 확인
```

### 주간 점검
```
월요일 - 백업 테스트
수요일 - 보안 패치 검토
금요일 - 성능 리포트
```

### 월간 점검
```
첫째 주 - DR Drill (분기별)
둘째 주 - 용량 계획 검토
셋째 주 - 보안 감사
넷째 주 - 성능 최적화
```

---

[← 메인으로 돌아가기](../README.md)