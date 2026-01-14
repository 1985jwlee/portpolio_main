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
7. [Phase 4: 문서 정리](#phase-4-문서-정리)

---

## 전체 목표

### 핵심 목표

> **"실시간 판정은 메모리에서 끝나고, 기록과 복구는 비동기로 흡수되는 구조를 실제로 증명한다."**

### 이 포트폴리오는

```
✗ 대규모 MMO 완성 구현을 목표로 하지 않습니다
✗ 화려한 클라이언트나 콘텐츠 볼륨을 추구하지 않습니다
✗ 모든 기능을 다 만드는 것을 지양합니다

✓ 운영 가능한 실시간 시스템의 핵심 구조만을
✓ 최소 구현으로 증명하는 것이 목적입니다
```

---

## 로드맵 개요

```
Phase 0. 설계 확정 (문서)              ✅ 완료
Phase 1. MVP 구현 (핵심 흐름)          🔄 진행 예정
Phase 2. 이벤트 신뢰성                  📋 계획
Phase 3. Hot/Cold Snapshot             📋 계획
Phase 4. 포트폴리오 정리                📋 계획
```

### 전체 타임라인

| Phase | 내용 | 예상 소요 | 상태 |
|-------|------|----------|------|
| Phase 0 | 설계 확정 | 1~2일 | ✅ 완료 |
| Phase 1 | MVP 구현 | 1~2주 | 🔄 진행 예정 |
| Phase 2 | 이벤트 신뢰성 | 3~5일 | 📋 계획 |
| Phase 3 | Snapshot | 4~7일 | 📋 계획 |
| Phase 4 | 문서 정리 | 2~3일 | 📋 계획 |

**총 예상 기간**: 약 3~4주

---

## Phase 0: 설계 확정

### ✅ 완료

**목적**: "이 시스템은 이렇게 만들기로 결정했다"는 기준을 고정

**산출물**:
- ✅ 최종 아키텍처 설계 문서
- ✅ 설계 원칙 명문화
- ✅ 대안 비교와 선택 이유
- ✅ 운영 관점 설계

**확정된 설계 원칙**:
```
✓ Server-authoritative 구조
✓ Packet → Command → Domain → Event 흐름
✓ 실시간 처리와 비동기 기록의 명확한 분리
✓ 게임 서버는 DB 성공/실패를 기다리지 않는다
```

**이 단계는 이후 모든 구현 판단의 기준선이 됩니다.**

---

## Phase 1: MVP 구현

### 🔄 진행 예정

**목적**: 실시간 게임 서버와 이벤트 기반 플랫폼 서버가 실제로 분리되어 동작함을 증명

**예상 소요**: 1~2주

---

### 1-1. 게임 서버 (C# TCP/IP)

#### 구현 범위 (최소)

```
✓ TCP/IP 소켓 서버
✓ Session 관리
✓ Packet → Command 변환
✓ 단일 GameLoop Tick
✓ In-memory Player State
✓ Domain Event 생성
✓ Kafka Producer 연동
```

#### 반드시 포함할 최소 기능

**1. 플레이어 접속**
```csharp
public void OnPlayerConnect(Session session)
{
    var player = new Player(session.PlayerId);
    _activePlayers.Add(player);
    
    PublishEvent(new PlayerConnectedEvent
    {
        PlayerId = player.Id,
        ConnectedAt = DateTime.UtcNow
    });
}
```

**2. 간단한 상태 변경 (위치 변경)**
```csharp
public void ProcessMoveCommand(MoveCommand cmd)
{
    var player = GetPlayer(cmd.PlayerId);
    
    // Validation
    if (!ValidateMove(cmd))
    {
        SendRejection(cmd.PlayerId, "Invalid move");
        return;
    }
    
    // State Change (메모리에서 즉시)
    var oldPosition = player.Position;
    player.Position = cmd.NewPosition;
    
    // Domain Event 발행 (비동기)
    PublishEvent(new PlayerMovedEvent
    {
        PlayerId = player.Id,
        FromPosition = oldPosition,
        ToPosition = cmd.NewPosition,
        OccurredAt = DateTime.UtcNow
    });
    
    // 즉시 응답 (Kafka 응답 대기 X)
    SendResponse(cmd.PlayerId, player.Position);
}
```

#### 의도적으로 구현하지 않는 것

```
✗ 전투 시스템
✗ MMO 콘텐츠
✗ 복잡한 동기화 로직
✗ 클라이언트 예측
✗ 인벤토리 시스템
```

#### 체크리스트

```
게임 서버:
☐ TCP 서버 기동
☐ 클라이언트 연결 수락
☐ 패킷 수신 및 역직렬화
☐ Command 생성
☐ GameLoop Tick 동작
☐ Domain Event 발행
☐ Kafka 연동
☐ 로그 출력
```

---

### 1-2. 플랫폼 서버 (TypeScript / bun.js)

#### 구현 범위

```typescript
// Kafka Consumer
class EventConsumer {
  async start() {
    await this.kafka.subscribe('game.events.player');
    
    await this.kafka.consume(async (event) => {
      switch (event.type) {
        case 'PlayerConnectedEvent':
          await this.handlePlayerConnected(event);
          break;
        case 'PlayerMovedEvent':
          await this.handlePlayerMoved(event);
          break;
      }
    });
  }
  
  private async handlePlayerMoved(event: PlayerMovedEvent) {
    // 단순한 DB 저장 (MySQL)
    await this.db.insert('player_movements', {
      player_id: event.playerId,
      from_position: event.fromPosition,
      to_position: event.toPosition,
      occurred_at: event.occurredAt
    });
    
    console.log(`[Event] Player moved: ${event.playerId}`);
  }
}
```

#### 핵심 포인트

```
✓ 플랫폼 서버는 오직 Kafka 이벤트만 소비
✓ 실시간 로직에 절대 개입하지 않음
✓ 게임 서버와 직접 통신하지 않음
```

#### 체크리스트

```
플랫폼 서버:
☐ Kafka Consumer 연결
☐ 이벤트 수신 확인
☐ 이벤트 파싱
☐ DB 저장
☐ 로그 출력
☐ 간단한 API 엔드포인트 (선택)
```

---

### 1-3. Unity 클라이언트 (증명용 테스트 클라이언트)

#### 구현 범위

```csharp
public class PlayerController : MonoBehaviour
{
    private NetworkClient _client;
    
    void Update()
    {
        // 입력 감지
        if (Input.GetKeyDown(KeyCode.W))
        {
            var newPos = transform.position + Vector3.forward;
            
            // 로컬에서 즉시 변경하지 않음!
            // 서버에 요청만 전송
            _client.SendMoveRequest(newPos);
            
            Debug.Log($"[Client] Move requested: {newPos}");
        }
    }
    
    // 서버로부터 승인된 위치만 반영
    public void OnServerMoveConfirmed(Vector3 serverPosition)
    {
        transform.position = serverPosition;
        Debug.Log($"[Client] Move confirmed: {serverPosition}");
    }
}
```

#### 시각적 기준

```
✓ 큐브 하나면 충분
✓ 뚝뚝 이동해도 문제 없음
✓ 부드러운 보간(Lerp)은 선택 사항
✓ Debug UI로 상태 표시
```

#### 체크리스트

```
Unity 클라이언트:
☐ TCP 연결
☐ 패킷 전송
☐ 패킷 수신
☐ WASD 입력
☐ 서버 응답 대기
☐ 위치 갱신
☐ Debug 로그 출력
```

---

### MVP 단계에서 증명되는 것

```
✓ 실시간 판정과 기록의 분리 구조
✓ 서로 다른 언어(C#, TypeScript) 간 이벤트 연계
✓ 게임 서버는 가볍고, 플랫폼 서버가 무거운 책임
✓ Server-authoritative 구조
```

**이 단계까지가 MVP이며, 여기까지만으로도 포트폴리오로 충분한 설득력을 가집니다.**

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

---

## Phase 2: 이벤트 신뢰성

### 📋 계획

**목적**: Kafka 기반 구조에서 반드시 질문받게 되는 "이벤트 신뢰성"에 대한 답을 제시

**예상 소요**: 3~5일

---

### 2-1. Domain Event 확장

#### 이벤트 메타데이터 추가

```csharp
public abstract class DomainEvent
{
    public string EventId { get; set; }        // UUID
    public string AggregateId { get; set; }    // playerId 또는 entityId
    public DateTime OccurredAt { get; set; }   // 발생 시각
    public int Version { get; set; }           // 이벤트 버전
}

public class PlayerMovedEvent : DomainEvent
{
    public string PlayerId { get; set; }
    public Vector3 FromPosition { get; set; }
    public Vector3 ToPosition { get; set; }
}
```

---

### 2-2. 플랫폼 서버 처리 전략

#### Idempotency 구현

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
      // 이벤트 처리
      await this.processEvent(event);
      
      // 처리 완료 기록 (TTL: 1시간)
      await this.redis.setex(key, 3600, 'processed');
    } catch (error) {
      // 실패 시 DLQ로 전송
      await this.sendToDLQ(event, error);
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

#### 핵심 원칙

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

### 📋 계획

**목적**: 게임 서버 장애 시에도 서비스가 복구 가능함을 구조적으로 증명

**예상 소요**: 4~7일

---

### 3-1. Hot Snapshot (Redis)

```csharp
public class RedisSnapshotService
{
    private readonly IRedisClient _redis;
    
    // 주기적으로 Player State 저장 (5~10초)
    public async Task SaveHotSnapshot(Player player)
    {
        var key = $"snapshot:player:{player.Id}";
        
        // MessagePack으로 binary 직렬화
        var snapshot = MessagePackSerializer.Serialize(player);
        
        // TTL: 5분
        await _redis.SetAsync(key, snapshot, TimeSpan.FromMinutes(5));
    }
    
    // 복구
    public async Task LoadHotSnapshot(string playerId)
    {
        var key = $"snapshot:player:{playerId}";
        var snapshot = await _redis.GetAsync(key);
        
        if (snapshot == null)
            return null;
        
        return MessagePackSerializer.Deserialize(snapshot);
    }
}
```

**특징**:
```
✓ 정확성보다 속도를 우선
✓ TTL로 자동 정리
✓ 수초 내 복구 가능
```

---

### 3-2. Cold Snapshot (MongoDB)

```csharp
public class MongoSnapshotService
{
    // 서버 종료 또는 장애 시
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
    
    // 복구
    public async Task LoadColdSnapshot(string zoneId)
    {
        var snapshot = await _mongo
            .Find(s => s.ZoneId == zoneId)
            .SortByDescending(s => s.Timestamp)
            .FirstOrDefaultAsync();
        
        return snapshot?.Deserialize();
    }
}
```

**특징**:
```
✓ 장기 보관
✓ 정합성 검증 (Checksum)
✓ 점검 및 분석 용도
```

---

### 3-3. 복구 시나리오

```
[ 게임 서버 크래시 ]
    ↓
[ 복구 프로세스 시작 ]
    ↓
1. Redis Snapshot 존재?
   Yes → 즉시 복구 (수초)
   No → 2단계
    ↓
2. MongoDB Snapshot 존재?
   Yes → 복구 (수분)
   No → 3단계
    ↓
3. 초기 상태로 시작
   (Kafka Event Replay로 보정 가능)
```

---

### Phase 3 완료 조건

```
☐ Redis Snapshot 저장
☐ Redis Snapshot 로드
☐ MongoDB Snapshot 저장
☐ MongoDB Snapshot 로드
☐ 복구 우선순위 로직
☐ 테스트: 게임 서버 재시작 후 플레이어 상태 복원
```

---

## Phase 4: 문서 정리

### 📋 계획

**목적**: 포트폴리오로서의 완결성 확보

**예상 소요**: 2~3일

---

### 4-1. 반드시 명시해야 할 것

```markdown
## 구현 범위

### ✅ 구현 완료
- Game Server (C#): TCP, Session, GameLoop, Domain Event
- Platform Server (TypeScript): Kafka Consumer, DB Persistence
- Unity Client: Server-authoritative 증명용 테스트 클라이언트
- Event Idempotency & DLQ
- Hot/Cold Snapshot

### 📋 의도적으로 구현하지 않은 것
- 전투 시스템
- 복잡한 게임 콘텐츠
- 클라이언트 예측
- 완전한 매치메이킹
- 운영 대시보드 (설계만)

### 🎯 왜 여기서 멈췄는가?

**이 포트폴리오는 대규모 MMO 구현을 목표로 하지 않았습니다.**

실시간 판정, 이벤트 기반 확장, 장애 흡수 구조를 증명하는 데 
필요한 최소 구현까지만 진행했습니다.

추가 기능은 "더 만들 수 있다"가 아니라 
"언제 멈춰야 하는지 안다"를 증명하기 위해 의도적으로 제외했습니다.
```

---

### 4-2. 실행 가이드 작성

```markdown
# 로컬 실행 가이드

## 필요 환경
- .NET 8.0
- bun.js
- Unity 2022.3 LTS
- Docker (Kafka, Redis, MongoDB, MySQL)

## 1. 인프라 시작
```bash
docker-compose up -d
```

## 2. 게임 서버 시작
```bash
cd game-server
dotnet run
```

## 3. 플랫폼 서버 시작
```bash
cd platform-server
bun install
bun run dev
```

## 4. Unity 클라이언트 실행
- Unity에서 GameScene 실행
- WASD로 이동 테스트
```

---

### 4-3. 데모 영상 녹화

**시나리오**:
```
1. 게임 서버 시작 (터미널)
2. 플랫폼 서버 시작 (터미널)
3. Unity 클라이언트 실행
4. WASD로 이동
5. 서버 로그에서 이벤트 확인
6. DB에서 기록 확인
```

**길이**: 1~2분

**형식**: MP4, 1080p

---

### Phase 4 완료 조건

```
☐ README 최종 검토
☐ 실행 가이드 작성
☐ 데모 영상 녹화
☐ GitHub 정리
☐ 코드 주석 추가
☐ 라이선스 파일
```

---

## 가장 중요한 결론

> **이 로드맵의 목적은 "완성"이 아닙니다.**

> **"이 사람은 언제 구현을 멈춰야 하는지도 아는 엔지니어다"**

> **이 인상을 남기는 것이 핵심입니다.**

---

[← 메인으로 돌아가기](../README.md)