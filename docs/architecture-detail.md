# 아키텍처 상세 문서

[← 메인으로 돌아가기](../README.md)

---

## 📋 목차

1. [전체 시스템 개요](#전체-시스템-개요)
2. [클라이언트-서버 경계](#클라이언트-서버-경계)
3. [게임 서버 구조](#게임-서버-구조)
4. [이벤트 스트림 설계](#이벤트-스트림-설계)
5. [플랫폼 서버 구조](#플랫폼-서버-구조)
6. [데이터 계층 전략](#데이터-계층-전략)

---

## 전체 시스템 개요

### 핵심 목표

```
✓ 실시간 판정의 안정성 확보
✓ 비동기 처리로 인한 장애 전파 최소화
✓ 운영과 확장이 가능한 구조
✓ 캐주얼 게임에 적합하지만 MMO까지 확장 가능한 설계
```

### 전체 구성 요소

```
[ Unity 게임 클라이언트 ]
        ↓ TCP/IP
[ C# 실시간 게임 서버 ]
        ↓ Domain Events
[ Kafka 이벤트 스트림 ]
        ↓
[ TypeScript 플랫폼 서버 ]
        ↓
[ Redis / MongoDB / MySQL ]
        ↓
[ React 운영 대시보드 ]
```

### 중요한 개념적 분리

| 구분 | 책임 |
|------|------|
| **실시간 처리** | 게임 서버 메모리에서 즉시 판정 |
| **비동기 처리** | Kafka를 통한 이벤트 기록 및 확장 |
| **판정 책임** | 게임 서버 단독 수행 |
| **기록 책임** | 플랫폼 서버 담당 |
| **게임 서버** | 실시간 로직, 상태 관리 |
| **플랫폼 서버** | 이벤트 처리, DB 연동, API 제공 |

---

## 클라이언트-서버 경계

### 설계 철학: Server-authoritative (서버 권한)

```
클라이언트의 역할:
- "의도"만 전달한다 (MoveRequest, AttackRequest)
- 상태 변경 권한은 없다
- 서버 응답을 기다린다
- 서버가 승인한 상태만 렌더링

서버의 역할:
- 상태 변경 권한은 서버에만 존재한다
- 클라이언트 상태는 언제든 서버 기준으로 교정 가능하다
- 모든 판정을 수행한다
- 치트 방지는 구조적으로 해결됨
```

### 왜 이렇게 설계했는가?

클라이언트에 권한을 주는 순간, 다음 문제가 발생합니다:

- 메모리 해킹으로 상태 조작 가능
- 클라이언트 간 동기화 불일치
- 판정 결과에 대한 신뢰성 상실

**서버 권한 모델은 복잡해서가 아니라, 안정성을 위해 선택한 구조입니다.**

### 프로토콜 흐름

```
1. 클라이언트: MoveRequest 전송
2. 서버: 검증 → 승인/거부 판정
3. 서버: 상태 변경 (메모리)
4. 서버: MoveResponse 전송
5. 클라이언트: 서버 응답 수신 후 화면 갱신
```

---

## 게임 서버 구조

### 기술 스택

- **언어**: C#
- **프로토콜**: TCP/IP
- **직렬화**: MessagePack
- **캐시**: Redis
- **스냅샷**: MongoDB
- **이벤트**: Kafka

### 패킷 처리 파이프라인

```
1. Deserialization (비신뢰 영역)
   ↓
2. Authentication / Session 검증
   ↓
3. Rate Limiting (플레이어별, IP별)
   ↓
4. Command Validation
   ↓
5. Command Queue 적재
   ↓
6. GameLoop Tick에서 Domain 처리
   ↓
7. Domain Event 생성
   ↓
8. Kafka Publishing (Fire-and-Forget)
```

### Command Validation 예시

```csharp
public class MoveCommandValidator
{
    public ValidationResult Validate(MoveCommand cmd, Player player)
    {
        // 1. 범위 검증
        if (!IsWithinMapBounds(cmd.Position))
            return ValidationResult.Fail("Out of bounds");
        
        // 2. 거리 검증 (텔레포트 방지)
        var distance = Vector3.Distance(player.Position, cmd.Position);
        if (distance > player.MaxMoveDistance)
            return ValidationResult.Fail("Move distance too large");
        
        // 3. 상태 검증
        if (player.IsDead || player.IsStunned)
            return ValidationResult.Fail("Cannot move in current state");
        
        // 4. 쿨다운 검증
        if (player.LastMoveTime + player.MoveCooldown > CurrentTime)
            return ValidationResult.Fail("Move on cooldown");
        
        return ValidationResult.Success();
    }
}
```

### GameLoop 구조

**핵심 결정**: 단일 GameLoop + Command Queue 구조

**단일 스레드를 선택한 이유**:
- 동시성 버그 원천 차단
- 상태 일관성 보장
- 디버깅 및 추적 용이
- 성능은 Zone 분산으로 해결

```csharp
public class GameTick
{
    private const int TargetTickRate = 20; // 50ms
    
    public async Task RunAsync()
    {
        while (true)
        {
            var startTime = DateTime.UtcNow;
            
            // 1. Command 처리
            ProcessCommands();
            
            // 2. World Update
            _world.Update();
            
            // 3. Event Publishing
            PublishEvents();
            
            // 4. 다음 Tick 대기
            await WaitForNextTick(startTime);
        }
    }
}
```

---

## 이벤트 스트림 설계

### 기술 스택: Kafka

#### Command vs Event

| 구분 | Command | Domain Event |
|------|---------|--------------|
| **의미** | "해달라" (요청) | "이미 일어났다" (사실) |
| **시점** | 미래 | 과거 |
| **실패** | 가능 | 불가능 (이미 발생함) |
| **상태 변경** | 요청함 | 하지 않음 |
| **용도** | 게임 로직 실행 | 기록 및 연동 |

#### Event 설계 예시

```
✅ 좋은 Event 이름:
- PlayerMovedEvent
- PlayerDiedEvent
- ItemAcquiredEvent
- MatchCompletedEvent

❌ 나쁜 Event 이름:
- MovePlayerCommand
- KillPlayer
- GiveItemCommand
```

#### Kafka Topic 구조

```
game.events.player          # 플레이어 상태 변경
game.events.combat          # 전투 로그
game.events.economy         # 재화 변동
game.events.match           # 매치 결과
game.snapshot.zone          # Zone 스냅샷
platform.commands           # 플랫폼→게임 지시
```

**파티션 전략**:
- Key: PlayerId 또는 ZoneId
- 순서 보장이 필요한 이벤트는 동일 파티션 할당

**Consumer Group**:
```
Analytics Service    → 통계 및 분석
Persistence Service  → DB 저장
Notification Service → 푸시 알림
Admin Dashboard      → 실시간 모니터링
```

### Fire-and-Forget 패턴

> **게임 서버는 Kafka의 응답을 기다리지 않습니다.**

```csharp
// ❌ 잘못된 예시 (동기 처리)
public void ProcessMove(MoveCommand cmd)
{
    player.Move(cmd.Position);
    
    var evt = new PlayerMovedEvent { ... };
    await kafka.Publish(evt);  // ❌ 대기하면 안 됨!
    
    SendResponse(player.Position);
}

// ✅ 올바른 예시 (비동기 처리)
public void ProcessMove(MoveCommand cmd)
{
    player.Move(cmd.Position);
    
    var evt = new PlayerMovedEvent { ... };
    kafka.PublishAsync(evt);  // ✅ Fire-and-Forget
    
    SendResponse(player.Position);  // 즉시 응답
}
```

**왜 기다리지 않는가?**
- GameLoop Tick이 DB/Kafka 지연에 영향받으면 안 됨
- 판정은 메모리에서 이미 확정됨
- 기록 실패가 게임플레이를 막아서는 안 됨

---

## 플랫폼 서버 구조

### 기술 스택

- **런타임**: bun.js
- **프레임워크**: ElysiaJS
- **이벤트**: Kafka
- **DB**: MySQL
- **ORM**: Drizzle

### 구조

```
[ Kafka Consumer ]
    ↓ Event Handling
[ Business Logic ]
    ↓
[ Database (MySQL) ]
    ↓
[ REST API / WebSocket ]
```

### 역할 분리

```
플랫폼 서버가 하는 것:
✓ Kafka 이벤트 수신 및 처리
✓ DB 영속화 (비동기)
✓ 운영 API 제공
✓ 통계 및 분석
✓ 관리자 기능

플랫폼 서버가 하지 않는 것:
✗ 게임 로직 실행
✗ 실시간 판정
✗ 게임 서버 상태 관리
```

### Idempotency 처리

```typescript
// 중복 이벤트 처리 방지
async function handleRewardEvent(event: RewardGrantedEvent) {
  const key = `event:${event.eventId}`;
  
  // 이미 처리된 이벤트인지 확인
  const processed = await redis.get(key);
  if (processed) {
    logger.info(`Event already processed: ${event.eventId}`);
    return;
  }
  
  // 처리
  await grantReward(event.playerId, event.amount);
  
  // 처리 완료 기록 (TTL: 1시간)
  await redis.setex(key, 3600, 'processed');
}
```

---

## 데이터 계층 전략

### 이중 스냅샷 구조

| 저장소 | 역할 | 주기 | TTL | 복구 시간 |
|--------|------|------|-----|-----------|
| **Redis** | Hot Snapshot | 5~10초 | 2~5분 | 수초 |
| **MongoDB** | Cold Snapshot | 1~5분 | - | 수분 |
| **MySQL** | 영속 데이터 | 이벤트 기반 | - | - |

### Redis (Hot Snapshot)

```
키 구조:
snapshot:zone:{zoneId}
snapshot:player:{playerId}
snapshot:session:{sessionId}

용도:
- 게임 서버 크래시 시 빠른 복구
- 세션 관리
- Rate Limiting 카운터
```

### MongoDB (Cold Snapshot)

```json
// Document 예시
{
  "_id": "zone_1_20240115_143000",
  "zoneId": "zone_1",
  "timestamp": 1705304600,
  "players": [...],
  "entities": [...],
  "checksum": "abc123..."
}
```

**용도**:
- 서버 점검 시 복구
- Redis 장애 시 복구
- 히스토리 분석

### MySQL (영속 데이터)

```
테이블 설계:
- accounts (계정 정보)
- inventory (인벤토리)
- transactions (재화 거래 내역)
- match_history (매치 기록)
```

**트랜잭션 사용 시점**:
- 재화 차감/지급 (ACID 보장 필요)
- 결제 처리
- 중요 운영 데이터

### 복구 우선순위

```
1순위: Redis Snapshot (수초 내 복구)
    ↓ 실패 시
2순위: MongoDB Snapshot (수분 내 복구)
    ↓ 실패 시
3순위: 초기 상태 + Kafka Event Replay
```

### 핵심 원칙

> **게임 서버는 DB 성공/실패에 의존하지 않습니다.**

이는 성능뿐 아니라 **장애 격리**를 위한 결정입니다.

---

[← 메인으로 돌아가기](../README.md)