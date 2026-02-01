# 구현 로드맵

[← 메인으로 돌아가기](../README.md)

---

## 📋 목차

1. [전체 목표](#전체-목표)
2. [로드맵 개요](#로드맵-개요)
3. [Phase 0: 설계 확정](#phase-0-설계-확정)
4. [Phase 1: MVP 구현](#phase-1-mvp-구현)
5. [Phase 2: 이벤트 신뢰성](#phase-2-이벤트-신뢰성)
6. [Phase 3: Snapshot](#phase-3-snapshot)
7. [Phase 4: Admin Dashboard](#phase-4-admin-dashboard)

---

## 전체 목표

> **"실시간 판정은 메모리에서 끝나고, 기록과 복구는 비동기로 흡수되는 구조를 실제로 증명한다."**

이 포트폴리오는 대규모 MMO 완성을 목표로 하지 않습니다. **운영 가능한 실시간 시스템의 핵심 구조만을 최소 구현으로 증명**하는 것이 목적입니다.

---

## 로드맵 개요

| Phase | 내용 | 예상 소요 | 상태 |
|-------|------|-----------|------|
| Phase 0 | 설계 확정 | 1~2일 | ✅ 완료 |
| Phase 1 | MVP 구현 | 1~2주 | 🔄 진행 예정 |
| Phase 2 | 이벤트 신뢰성 | 3~5일 | 📋 계획 |
| Phase 3 | Snapshot | 4~7일 | 📋 계획 |
| Phase 4 | Admin Dashboard | 3~5일 | 📋 계획 |

**총 예상 기간**: 약 3~4주

### Phase별 완료 조건 요약

| Phase | 핵심 완료 조건 |
|-------|---------------|
| **Phase 1** | 클라이언트 → 서버 → Kafka → 플랫폼 → DB 전체 흐름 동작 |
| **Phase 2** | 동일 이벤트 2회 전송 시 1회만 처리, DLQ로 실패 이벤트 분리 |
| **Phase 3** | 게임 서버 재시작 후 플레이어 상태 복원 (Redis → MongoDB 순차) |
| **Phase 4** | 실시간 모니터링 + Snapshot 관리 UI 동작 |

---

## Phase 0: 설계 확정

### ✅ 완료

**목적:** "이 시스템은 이렇게 만들기로 결정했다"는 기준을 고정

**확정된 설계 원칙:**
```
✓ Server-authoritative 구조
✓ Packet → Command → Domain → Event 흐름
✓ 실시간 처리와 비동기 기록의 명확한 분리
✓ 게임 서버는 DB 성공/실패를 기다리지 않는다
```

> 이 단계는 이후 모든 구현 판단의 기준선이 됩니다.

---

## Phase 1: MVP 구현

### 🔄 진행 예정 | 예상 소요: 1~2주

**목적:** 실시간 게임 서버와 이벤트 기반 플랫폼 서버가 실제로 분리되어 동작함을 증명

> 구체적 구현 코드는 [기술 스택 가이드](docs/tech-stack-guide.md)를 참조합니다.

---

### 1-1. 게임 서버 (C# TCP/IP)

**구현 범위:**
```
✓ TCP/IP 소켓 서버        ✓ Session 관리
✓ Packet → Command 변환   ✓ 단일 GameLoop Tick
✓ In-memory Player State  ✓ Domain Event 생성
✓ Kafka Producer 연동
```

**의도적으로 구현하지 않는 것:**
```
✗ 전투 시스템           ✗ MMO 콘텐츠
✗ 복잡한 동기화 로직     ✗ 클라이언트 예측
✗ 인벤토리 시스템
```

**체크리스트:**
```
☐ TCP 서버 기동                ☐ 클라이언트 연결 수락
☐ 패킷 수신 및 역직렬화         ☐ Command 생성
☐ GameLoop Tick 동작           ☐ Domain Event 발행
☐ Kafka 연동                  ☐ 로그 출력
```

---

### 1-2. 플랫폼 서버 (TypeScript / bun.js)

**구현 범위:**
```
✓ Kafka Consumer 연결     ✓ 이벤트 수신 및 파싱
✓ DB 저장 (MySQL)         ✓ 간단한 API 엔드포인트
```

**핵심 포인트:**
```
✓ 플랫폼 서버는 오직 Kafka 이벤트만 소비
✓ 실시간 로직에 절대 개입하지 않음
✓ 게임 서버와 직접 통신하지 않음
```

**체크리스트:**
```
☐ Kafka Consumer 연결      ☐ 이벤트 수신 확인
☐ 이벤트 파싱              ☐ DB 저장
☐ 로그 출력                ☐ API 엔드포인트 (선택)
```

---

### 1-3. Unity 클라이언트 (증명용 테스트 클라이언트)

**구현 범위:** TCP 연결, WASD 입력 → 서버 요청 전송, 서버 응답 수신 후 위치 갱신

**시각적 기준:**
```
✓ 큐브 하나면 충분
✓ 뚝뚝 이동해도 문제 없음
✓ 부드러운 보간(Lerp)은 선택 사항
```

**체크리스트:**
```
☐ TCP 연결                 ☐ WASD 입력
☐ 패킷 전송/수신            ☐ 서버 응답 대기
☐ 위치 갱신                ☐ Debug 로그 출력
```

---

### Phase 1 완료 조건

```
☐ 게임 서버 실행
☐ Unity 클라이언트 연결
☐ WASD로 이동 요청
☐ 서버에서 검증 후 승인
☐ Kafka로 이벤트 발행
☐ 플랫폼 서버에서 이벤트 수신
☐ DB에 이동 기록 저장
☐ 1분 영상 녹화
```

**이 단계까지가 MVP이며, 여기까지만으로도 포트폴리오로 충분한 설득력을 가집니다.**

---

## Phase 2: 이벤트 신뢰성

### 📋 계획 | 예상 소요: 3~5일

**목적:** Kafka 기반 구조에서 반드시 질문받게 되는 "이벤트 신뢰성"에 대한 답을 제시

---

### 2-1. Domain Event 메타데이터 추가

```csharp
public abstract class DomainEvent
{
    public string EventId { get; set; }        // UUID — Idempotency 키
    public string AggregateId { get; set; }    // playerId 또는 entityId
    public DateTime OccurredAt { get; set; }   // 발생 시각
    public int Version { get; set; }           // 이벤트 버전
}
```

---

### 2-2. Idempotency 구현

```typescript
class IdempotentEventHandler {
  async handle(event: DomainEvent) {
    const key = `event:${event.eventId}`;

    // 이미 처리된 이벤트인지 확인
    const processed = await this.redis.get(key);
    if (processed) {
      logger.info(`Event already processed: ${event.eventId}`);
      return;
    }

    try {
      await this.processEvent(event);
      await this.redis.setex(key, 3600, 'processed'); // TTL: 1시간
    } catch (error) {
      await this.sendToDLQ(event, error); // 실패 시 DLQ로 전송
    }
  }
}
```

---

### 2-3. DLQ (Dead Letter Queue)

```
정상 흐름:
[ Game Server ] → [ Kafka ] → [ Platform Server ] → [ DB ]

실패 흐름:
[ Game Server ] → [ Kafka ] → [ Platform Server ]
                                     ↓ (실패)
                                [ DLQ Topic ]
                                     ↓ (수동 처리)
                                [ Admin Review ]
```

**핵심 원칙:**
```
✓ DB 저장 실패 시 즉시 재시도하지 않음
✓ 게임 서버는 실패 여부를 알지 못함
✓ 실패 처리는 전적으로 운영 영역의 책임
```

---

### Phase 2 완료 조건

```
☐ Event ID 추가
☐ Idempotency 검증 로직
☐ Redis 중복 체크
☐ DLQ Topic 생성
☐ 실패 이벤트 DLQ 전송
☐ 테스트: 동일 이벤트 2번 전송 시 1번만 처리
```

---

## Phase 3: Snapshot

### 📋 계획 | 예상 소요: 4~7일

**목적:** 게임 서버 장애 시에도 서비스가 복구 가능함을 구조적으로 증명

---

### 3-1. Hot Snapshot (Redis)

```csharp
public class RedisSnapshotService
{
    // 주기적으로 Player State 저장 (5~10초)
    public async Task SaveHotSnapshot(Player player)
    {
        var key = $"snapshot:player:{player.Id}";
        var snapshot = MessagePackSerializer.Serialize(player);
        await _redis.SetAsync(key, snapshot, TimeSpan.FromMinutes(5)); // TTL: 5분
    }

    // 복구
    public async Task<Player> LoadHotSnapshot(string playerId)
    {
        var key = $"snapshot:player:{playerId}";
        var snapshot = await _redis.GetAsync<byte[]>(key);
        if (snapshot == null) return null;
        return MessagePackSerializer.Deserialize<Player>(snapshot);
    }
}
```

특징: 정확성보다 속도를 우선, TTL로 자동 정리, 수초 내 복구 가능

---

### 3-2. Cold Snapshot (MongoDB)

```csharp
public class MongoSnapshotService
{
    public async Task SaveColdSnapshot(Zone zone)
    {
        var snapshot = new ZoneSnapshot
        {
            ZoneId = zone.Id,
            Timestamp = DateTime.UtcNow,
            Players = zone.Players.Select(p => p.Serialize()).ToList(),
            Checksum = CalculateChecksum(zone)
        };
        await _mongo.InsertOneAsync(snapshot);
    }

    public async Task<ZoneSnapshot> LoadColdSnapshot(string zoneId)
    {
        return await _mongo
            .Find(s => s.ZoneId == zoneId)
            .SortByDescending(s => s.Timestamp)
            .FirstOrDefaultAsync();
    }
}
```

특징: 장기 보관, Checksum으로 정합성 검증, 점검/분석 용도

---

### 3-3. 복구 우선순위

```
[ 게임 서버 크래시 ]
    ↓
1. Redis Snapshot 존재? → Yes → 즉시 복구 (수초)
    ↓ No
2. MongoDB Snapshot 존재? → Yes → 복구 (수분)
    ↓ No
3. 초기 상태로 시작 (Kafka Event Replay로 보정 가능)
```

---

### Phase 3 완료 조건

```
☐ Redis Snapshot 저장/로드
☐ MongoDB Snapshot 저장/로드
☐ 복구 우선순위 로직
☐ 테스트: 게임 서버 재시작 후 플레이어 상태 복원
```

---

## Phase 4: Admin Dashboard

### 📋 계획 | 예상 소요: 3~5일

**목적:** 운영 도구를 실제로 구현하여 시스템의 운영 가능성을 증명

**관련 프로젝트**: [React Object State Manager](https://github.com/1985jwlee/portpolio_react)

---

### 구현 예정 기능

```
1. 실시간 모니터링          2. 플레이어 상태 조회
   - Zone별 CCU               - 오브젝트 상태
   - Tick 지연 모니터링         - Component 필드값 실시간 조회
   - Health Check 현황         - 상태 변경 이력

3. Event Stream 시각화      4. 장애 대응 인터페이스
   - Topic별 이벤트 흐름        - Snapshot 복구 트리거
   - Consumer Lag 모니터링      - 서버 재시작 컨트롤
   - 이벤트 처리 속도           - 긴급 공지 발송

5. Snapshot 관리
   - Hot/Cold Snapshot 조회
   - 수동 Snapshot 생성
   - 복구 테스트
```

### 기술 스택

```
Frontend: React 19 + TypeScript
State: Zustand (전역 상태 관리)
UI: Tailwind CSS
Real-time: WebSocket (Server → Client)
API: REST (Client → Server)
```

### 플랫폼 서버 WebSocket API

```typescript
import { Elysia } from 'elysia';
import { ws } from '@elysiajs/websocket';

const app = new Elysia()
  .use(ws())
  .ws('/ws/monitor', {
    open(ws) {
      const interval = setInterval(() => {
        ws.send({
          type: 'metrics',
          data: {
            ccu: getCurrentCCU(),
            tickDelay: getAverageTickDelay(),
            eventRate: getEventRate()
          }
        });
      }, 1000);
      ws.data.interval = interval;
    },
    close(ws) {
      clearInterval(ws.data.interval);
    }
  });
```

---

### Phase 4 완료 조건

```
☐ WebSocket API 구현
☐ React Dashboard 구현
☐ 실시간 모니터링 기능
☐ 플레이어 상태 조회
☐ Snapshot 관리 UI
☐ 데모 영상 녹화
```

---

## 가장 중요한 결론

> **이 로드맵의 목적은 "완성"이 아닙니다.**
>
> **"이 사람은 언제 구현을 멈추어야 하는지도 아는 엔지니어다"**
>
> 이 인상을 남기는 것이 핵심입니다.

---

[← 메인으로 돌아가기](../README.md)