# Event-driven Real-time Game Platform Architecture

> **실시간 판정은 메모리에서 끝나고, 기록과 복구는 비동기로 흡수되는 구조**

[![Architecture](https://img.shields.io/badge/Architecture-Event--Driven-blue)](docs/architecture-detail.md)
[![Status](https://img.shields.io/badge/Status-Design%20Complete-green)](docs/implementation-roadmap.md)
[![License](https://img.shields.io/badge/License-Portfolio-orange)](LICENSE)

---

## 📌 Executive Summary

**이 포트폴리오가 증명하는 것:**

```
✓ 실시간 시스템에서의 책임 분리 설계 능력
✓ Server-authoritative 구조에 대한 깊은 이해
✓ 이벤트 기반 아키텍처의 실무적 적용
✓ 장애, 복구, 운영까지 고려한 시스템 설계
✓ 개인이 아닌 조직에 남는 시스템을 만드는 관점
```

**대상 독자**: CTO, 테크 리드, 시니어 백엔드/서버 엔지니어

**핵심 메시지**: 
> "코드를 작성하는 능력이 아니라, 시스템을 설계하고 판단하는 능력을 보여줍니다."

---

## 🎯 왜 이 아키텍처인가?

### 많은 게임 서비스가 겪는 구조적 문제

```
🚨 사용자 증가 → 서버 복잡도 폭증 → 운영 불가능
🚨 실시간 처리와 기록 처리의 경계 불명확
🚨 장애 발생 시 영향 범위 예측 불가
🚨 특정 개발자에게 구조 이해가 집중됨
🚨 기능 추가 시 기존 로직 안정성 훼손
```

### 핵심 판단

> **문제의 핵심은 기술 부족이 아니라 구조 부재입니다.**

이 포트폴리오는 위 문제를 **구조적으로 해결**하는 과정을 보여줍니다.

---

## 🏗️ 3가지 핵심 설계 결정

### 1️⃣ 실시간 판정과 기록의 완전한 분리

```
[ 게임 서버 ]
  ↓ 메모리에서 즉시 판정 (< 50ms)
  ↓ Domain Event 발행 (Fire-and-Forget)
[ Kafka ]
  ↓ 비동기 처리
[ 플랫폼 서버 ]
  ↓ DB 저장, 통계, 운영
```

**판단 근거**:
- ✅ 게임플레이는 DB 지연의 영향을 받지 않음
- ✅ 장애 격리: Kafka/DB 다운 시에도 게임 진행
- ✅ 확장성: 이벤트 스트림으로 신규 서비스 추가 가능

**실무 시나리오**:
```
Kafka 다운 발생:
❌ 잘못된 설계: 게임 서버도 멈춤
✅ 이 설계: 게임은 계속, 이벤트는 메모리 버퍼링
```

---

### 2️⃣ Server-authoritative 구조

```
클라이언트: "W키를 눌렀어요" (의도만 전달)
    ↓
서버: 검증 → 승인 → 상태 변경 → 응답
    ↓
클라이언트: 서버 응답을 받아야만 화면 갱신
```

**판단 근거**:
- ✅ 치트 방지는 구조적으로 해결
- ✅ 클라이언트는 언제든 서버 기준으로 교정 가능
- ✅ 복잡해서가 아니라 안정성을 위해 선택

**트레이드오프**:
```
Client-authoritative:
- 장점: 빠른 반응성, 구현 단순
- 단점: 치트 가능, 동기화 복잡

Server-authoritative:
- 장점: 치트 원천 차단, 상태 일관성 보장
- 단점: 네트워크 지연 체감, 구현 복잡
```

**결론**: 장기 운영 안정성을 위해 Server-authoritative 선택

---

### 3️⃣ 의도적으로 선택하지 않은 것들

```
❌ 게임 서버 직접 DB 접근
   → GameLoop이 DB에 의존하게 됨
   
❌ 모든 처리를 동기로
   → 사용자 증가 시 선형적 성능 저하
   
❌ 초기부터 마이크로서비스
   → 운영 복잡도 대비 얻는 가치 부족
   
❌ UDP 프로토콜
   → 포트폴리오 목적상 TCP로 충분
```

**핵심 원칙**: 
> **"지금 필요하지 않으면, 지금 만들지 않는다"**

---

## 📊 시스템 아키텍처

### 전체 구성도

```mermaid
graph TB
    subgraph "Client Layer"
        UNITY[Unity Client<br/>Server-Authoritative]
    end
    
    subgraph "Game Server Layer (C#)"
        TCP[TCP Socket Server]
        GAMELOOP[GameLoop Ticker<br/>Fixed Update]
        MEMORY[In-Memory State<br/>Player/Monster/Items]
        COMMAND[Command Handler<br/>Validation]
        EVENT_PUB[Event Publisher<br/>Kafka Producer]
    end
    
    subgraph "Event Stream"
        KAFKA[Apache Kafka<br/>Domain Events]
    end
    
    subgraph "Platform Server Layer (TypeScript/Bun.js)"
        EVENT_SUB[Event Consumer<br/>Kafka Subscriber]
        HANDLER[Event Handlers<br/>Idempotency Check]
        REST[REST API<br/>Operations]
    end
    
    subgraph "Storage Layer"
        REDIS[(Redis<br/>Hot Snapshot<br/>10s Recovery)]
        MONGO[(MongoDB<br/>Cold Snapshot<br/>2-3min Recovery)]
        MYSQL[(MySQL<br/>OLTP Data)]
    end
    
    UNITY -->|Command Request| TCP
    TCP --> COMMAND
    COMMAND --> GAMELOOP
    GAMELOOP --> MEMORY
    MEMORY --> EVENT_PUB
    EVENT_PUB -->|Fire & Forget| KAFKA
    KAFKA --> EVENT_SUB
    EVENT_SUB --> HANDLER
    HANDLER --> REST
    
    GAMELOOP -.->|Periodic Snapshot| REDIS
    HANDLER -.->|Cold Snapshot| MONGO
    HANDLER --> MYSQL
    
    style UNITY fill:#90EE90,stroke:#228B22,stroke-width:2px
    style GAMELOOP fill:#FFB6C1,stroke:#DC143C,stroke-width:3px
    style KAFKA fill:#FFA07A,stroke:#FF4500,stroke-width:2px
    style EVENT_SUB fill:#87CEEB,stroke:#4169E1,stroke-width:2px
    style REDIS fill:#FFE4E1,stroke:#DC143C
    style MONGO fill:#E0FFE0,stroke:#228B22

```

### 핵심 패턴: Command vs Event

| 구분 | Command | Domain Event |
|------|---------|--------------|
| **의미** | "해달라" (요청) | "이미 일어났다" (사실) |
| **시점** | 미래 | 과거 |
| **실패** | 가능 | 불가능 (이미 발생) |
| **흐름** | Client → Server | Server → Platform |
| **용도** | 게임 로직 실행 | 기록 및 연동 |

```mermaid
sequenceDiagram
    participant C as Client
    participant GS as Game Server
    participant K as Kafka
    participant PS as Platform Server
    participant DB as Database
    
    Note over C,GS: Command Pattern (동기)
    C->>GS: MoveCommand(newPos)
    GS->>GS: Validate Authority
    alt Valid
        GS->>GS: Update Memory State
        GS->>C: MoveResponse(accepted)
    else Invalid
        GS->>C: MoveResponse(rejected)
    end
    
    Note over GS,PS: Event Pattern (비동기)
    GS->>K: PlayerMovedEvent
    Note over K: Fire-and-Forget<br/>게임은 계속 진행
    K->>PS: Event Delivery
    PS->>PS: Idempotency Check
    PS->>DB: Persist Movement
    
    Note over C,DB: 핵심: Kafka 응답을 기다리지 않음!
```

---

## 🛡️ 장애 대응 설계

### 장애 영향도 매트릭스

```mermaid
graph TB
    subgraph "Always Available"
        GAMEPLAY[Game Server<br/>메모리 상태 관리]
    end
    
    subgraph "Can Fail Without Impact"
        KAFKA_FAIL[Kafka Down]
        REDIS_FAIL[Redis Down]
        DB_FAIL[Database Down]
        PLATFORM_FAIL[Platform Server Down]
    end
    
    subgraph "Degraded Mode"
        BUFFER[Memory Event Buffer]
        CACHE[In-Memory Cache]
    end
    
    GAMEPLAY -->|정상 동작| KAFKA_FAIL
    KAFKA_FAIL -->|버퍼링| BUFFER
    BUFFER -.->|복구 시 재전송| KAFKA_FAIL
    
    GAMEPLAY -->|정상 동작| REDIS_FAIL
    REDIS_FAIL -->|일시 캐시 사용| CACHE
    
    GAMEPLAY -->|정상 동작| DB_FAIL
    GAMEPLAY -->|정상 동작| PLATFORM_FAIL
    
    style GAMEPLAY fill:#90EE90,stroke:#228B22,stroke-width:3px
    style KAFKA_FAIL fill:#FFB6C1,stroke:#DC143C
    style REDIS_FAIL fill:#FFB6C1,stroke:#DC143C
    style DB_FAIL fill:#FFB6C1,stroke:#DC143C
    style PLATFORM_FAIL fill:#FFB6C1,stroke:#DC143C
    style BUFFER fill:#FFF8DC,stroke:#DAA520
    style CACHE fill:#FFF8DC,stroke:#DAA520
```

**설계 철학**: 
> "게임플레이는 어떤 백엔드 장애에도 멈추지 않는다"

### 복구 전략

```mermaid
graph TD
    CRASH[Game Server Crash] --> TRY_HOT{Redis Available?}
    
    TRY_HOT -->|Yes| HOT_RECOVER[Hot Snapshot Recovery<br/>RTO: 10초]
    TRY_HOT -->|No| TRY_COLD{MongoDB Available?}
    
    TRY_COLD -->|Yes| COLD_RECOVER[Cold Snapshot Recovery<br/>RTO: 2-3분]
    TRY_COLD -->|No| EVENT_REPLAY[Event Replay<br/>RTO: 수분~수십분]
    
    HOT_RECOVER --> ONLINE[서비스 재개]
    COLD_RECOVER --> ONLINE
    EVENT_REPLAY --> ONLINE
    
    style CRASH fill:#DC143C,stroke:#8B0000,color:#fff
    style HOT_RECOVER fill:#90EE90,stroke:#228B22
    style COLD_RECOVER fill:#FFA07A,stroke:#FF4500
    style EVENT_REPLAY fill:#FFB6C1,stroke:#DC143C
    style ONLINE fill:#4169E1,stroke:#00008B,color:#fff


```

**증거 코드**:

```csharp
public class GameServerRecovery
{
    public async Task<RecoveryResult> Recover(string zoneId)
    {
        // 1. Hot Snapshot 시도
        var hotSnapshot = await _redis.LoadSnapshot(zoneId);
        if (hotSnapshot != null)
        {
            Logger.Info("Recovered from Redis in 10s");
            return RecoveryResult.FromHotSnapshot(hotSnapshot);
        }

        // 2. Cold Snapshot 시도
        var coldSnapshot = await _mongo.LoadSnapshot(zoneId);
        if (coldSnapshot != null)
        {
            Logger.Info("Recovered from MongoDB in 2~3min");
            return RecoveryResult.FromColdSnapshot(coldSnapshot);
        }

        // 3. Event Replay
        Logger.Warn("No snapshot found, replaying events");
        var events = await _kafka.ReplayEvents(zoneId);
        return RecoveryResult.FromEventReplay(events);
    }
}
```

---

## 🔄 핵심 흐름: Command → Event

### 플레이어 이동 시나리오

```csharp
// 1. 클라이언트 요청
client.SendMoveRequest(newPosition);

// 2. 서버 검증 & 판정
public void ProcessMove(MoveCommand cmd)
{
    var player = GetPlayer(cmd.PlayerId);
    
    // 검증 (서버 권한)
    if (!ValidateMove(cmd, player))
    {
        SendRejection(cmd.PlayerId, "Invalid move");
        return;
    }
    
    // 상태 변경 (메모리에서 즉시)
    var oldPos = player.Position;
    player.Position = cmd.NewPosition;
    
    // Domain Event 발행 (비동기, Fire-and-Forget)
    PublishEvent(new PlayerMovedEvent
    {
        EventId = Guid.NewGuid(),
        PlayerId = player.Id,
        FromPosition = oldPos,
        ToPosition = cmd.NewPosition,
        OccurredAt = DateTime.UtcNow
    });
    
    // 즉시 응답 (Kafka 응답 기다리지 않음!)
    SendResponse(cmd.PlayerId, player.Position);
}

// 3. 플랫폼 서버 처리 (별도 프로세스)
public async Task HandlePlayerMoved(PlayerMovedEvent evt)
{
    // Idempotency 검증
    if (await IsProcessed(evt.EventId))
        return;
    
    // DB 영속화
    await _db.SaveMovement(evt);
    
    // 처리 완료 기록
    await MarkProcessed(evt.EventId);
}
```

```mermaid

sequenceDiagram
    autonumber
    participant C as Unity Client
    participant GS as Game Server
    participant M as Memory State
    participant K as Kafka
    participant PS as Platform Server
    participant DB as Database
    
    Note over C: Player presses W key
    C->>GS: MoveCommand(playerId, newPosition)
    
    Note over GS: Server Authority
    GS->>GS: Validate Move<br/>(충돌, 속도, 치트)
    
    alt Valid Move
        GS->>M: Update player.Position
        Note over M: 상태 변경 완료<br/>(메모리에서 즉시)
        
        GS->>K: Publish PlayerMovedEvent<br/>(Fire-and-Forget)
        Note over GS,K: 비동기! Kafka 응답 안 기다림
        
        GS->>C: MoveResponse(success, newPosition)
        Note over C: 화면 업데이트
        
        K->>PS: Deliver Event
        PS->>PS: Idempotency Check<br/>(eventId 중복 확인)
        PS->>DB: Save Movement History
        
    else Invalid Move
        GS->>C: MoveResponse(rejected, reason)
        Note over C: 이동 취소, 원위치
    end
    
    Note over GS,DB: 중요: DB 저장 실패가 게임플레이를 막지 않음

```

**핵심 포인트**:
1. 게임 서버는 Kafka 응답을 기다리지 않음
2. 상태는 메모리에서 이미 확정됨
3. 기록 실패가 게임플레이를 막지 않음

---

## 📈 확장 시나리오

### Zone 기반 수평 확장

```mermaid
graph TB
    subgraph "100 CCU"
        Z1[Zone 1<br/>100 players]
    end
    
    subgraph "1,000 CCU"
        Z2_1[Zone 1<br/>100 players]
        Z2_2[Zone 2<br/>100 players]
        Z2_3[Zone 3<br/>100 players]
        Z2_N[... Zone 10]
    end
    
    subgraph "10,000 CCU"
        COORD[Zone Coordinator<br/>Load Balancer]
        Z3_1[Zone 1-10<br/>1,000 players]
        Z3_2[Zone 11-20<br/>1,000 players]
        Z3_N[... Zone 91-100]
        
        COORD --> Z3_1
        COORD --> Z3_2
        COORD --> Z3_N
    end
    
    style Z1 fill:#E8F4F8,stroke:#4A90E2
    style COORD fill:#FFB6C1,stroke:#DC143C,stroke-width:3px

```

### B2B 비즈니스 모델 확장

```mermaid
graph LR
    subgraph "Core Game Engine"
        CORE[Game Server Core<br/>변경 없음]
    end
    
    subgraph "Event Stream"
        KAFKA[Kafka Topics]
    end
    
    subgraph "Tenant A"
        PA[Platform A<br/>커스텀 로직]
        DBA[(Database A)]
    end
    
    subgraph "Tenant B"
        PB[Platform B<br/>커스텀 로직]
        DBB[(Database B)]
    end
    
    subgraph "Tenant C"
        PC[Platform C<br/>커스텀 로직]
        DBC[(Database C)]
    end
    
    CORE -->|Standard Events| KAFKA
    KAFKA --> PA
    KAFKA --> PB
    KAFKA --> PC
    PA --> DBA
    PB --> DBB
    PC --> DBC
    
    style CORE fill:#4169E1,stroke:#00008B,stroke-width:3px,color:#fff
    style KAFKA fill:#FFA07A,stroke:#FF4500

```

**핵심**: 게임 서버 코드 수정 없이 확장 가능

---

## 🛠️ 기술 스택

### 게임 서버 (C#)
- **언어**: C#
- **프로토콜**: TCP/IP
- **직렬화**: MessagePack
- **패턴**: Command Pattern, Event Sourcing
- **캐시**: Redis (Hot Snapshot)
- **이벤트**: Kafka Producer

### 플랫폼 서버 (TypeScript)
- **런타임**: bun.js
- **프레임워크**: ElysiaJS
- **ORM**: Drizzle
- **DB**: MySQL (정형), MongoDB (비정형)
- **이벤트**: Kafka Consumer

### 클라이언트 (Unity)
- **엔진**: Unity 2022.3 LTS
- **구조**: Server-authoritative
- **프로토콜**: TCP Socket

---

## 📚 상세 문서

| 문서 | 설명 | 대상 독자 |
|------|------|----------|
| [아키텍처 상세](docs/architecture-detail.md) | 전체 시스템 구조 및 설계 원칙 | 백엔드 엔지니어 |
| [설계 결정 과정](docs/design-decisions.md) | 왜 이렇게 설계했는가 | 테크 리드, CTO |
| [운영 가이드](docs/operational-guide.md) | 장애 대응 및 모니터링 | DevOps, SRE |
| [구현 로드맵](docs/implementation-roadmap.md) ⭐ | 단계별 구현 계획 | 개발자, PM |
| [기술 스택 가이드](docs/tech-stack-guide.md) | 언어별 구현 예시 | 개발자 |
| [다이어그램](docs/diagrams.md) | 시스템 시각화 자료 | 모든 이해관계자 |

---

## 🗺️ 구현 로드맵

```
Phase 0: 설계 확정               ✅ 완료
Phase 1: MVP 구현 (핵심 흐름)     🔄 진행 예정
Phase 2: 이벤트 신뢰성            📋 계획
Phase 3: Hot/Cold Snapshot       📋 계획
Phase 4: 운영 도구 구현           📋 계획
```

**예상 완료 기간**: 3~4주 (Phase 1 MVP까지는 1~2주)

### MVP 범위

**포함**:
- ✅ TCP 게임 서버 (C#)
- ✅ Command → Domain → Event 흐름
- ✅ Kafka Producer/Consumer
- ✅ 간단한 상태 변경 (이동)
- ✅ TypeScript 플랫폼 서버
- ✅ Unity 테스트 클라이언트

**의도적으로 제외**:
- ❌ 전투 시스템
- ❌ 복잡한 게임 콘텐츠
- ❌ 완전한 매치메이킹
- ❌ 운영 대시보드 (Phase 4에서 구현)

**왜 여기서 멈췄는가?**

> "더 만들 수 있다"가 아니라 **"언제 멈춰야 하는지 안다"**를 증명하기 위해

---

## 🎨 Phase 4: Admin Dashboard

### React 기반 운영 도구 구현

**관련 프로젝트**: [React Object State Manager](https://github.com/1985jwlee/portpolio_react)

Phase 4에서는 설계된 Admin Dashboard를 실제로 구현합니다.

#### 구현 예정 기능

```
1. 실시간 모니터링
   - Zone별 동접자 수 (CCU)
   - GameLoop Tick 지연 모니터링
   - 서버 Health Check 현황

2. 플레이어 상태 조회
   - 플레이어별 오브젝트 상태
   - Component 필드값 실시간 조회
   - 상태 변경 이력

3. Event Stream 시각화
   - Kafka Topic별 이벤트 흐름
   - Consumer Lag 모니터링
   - 이벤트 처리 속도

4. 장애 대응 인터페이스
   - Snapshot 복구 트리거
   - 서버 재시작 컨트롤
   - 긴급 공지 발송

5. Snapshot 관리
   - Hot/Cold Snapshot 조회
   - 수동 Snapshot 생성
   - 복구 테스트
```

#### 기술 스택

```
Frontend: React 19 + TypeScript
State: Zustand (전역 상태 관리)
UI: Tailwind CSS
Real-time: WebSocket (Server → Client)
API: REST (Client → Server)
```

#### React 프로토타입에서 검증된 것

- ✅ 동적 오브젝트 상태 관리
- ✅ Component 기반 필드 편집
- ✅ 상태 저장/복원 메커니즘
- ✅ Snapshot 관리 UI

이 프로토타입을 기반으로 실제 Admin Dashboard를 구현합니다.

---

## 💡 설계 철학

### 배운 교훈

**기술적 교훈**:
1. **복잡도는 비용이다**
   - "할 수 있다"와 "해야 한다"는 다름
   - 복잡한 구조는 반드시 그만한 가치를 제공해야 함

2. **장애는 언제나 발생한다**
   - 장애를 막는 것보다 격리하는 것이 현실적
   - "장애 시 어떻게 되는가"가 설계의 핵심

3. **확장은 선형적이어야 한다**
   - 사용자 2배 → 비용 2배가 이상적
   - 비선형 확장은 지속 불가능

**조직 관점 교훈**:
1. **문서화는 필수다**
   - 개인의 지식은 조직에 남지 않음
   - 구조를 설명할 수 없으면 좋은 구조가 아님

2. **운영 가능성이 구현보다 중요하다**
   - 만들 수 있어도 운영할 수 없으면 의미 없음
   - 운영팀이 이해할 수 있는 구조여야 함

3. **인수인계 가능한 시스템**
   - 특정 개발자에게 의존하는 구조는 위험
   - 시스템 자체가 설명할 수 있어야 함

---

## 🔗 관련 포트폴리오

이 설계 원칙은 다른 도메인에도 적용 가능합니다:

### 🎨 [Coin Data API — Platform Server in Practice](https://github.com/1985jwlee/portpolio_coindataapi)

**동일한 원칙의 비게임 도메인 적용 사례**

| 원칙 | 게임 서버 (본 프로젝트) | Coin API |
|------|----------------------|----------|
| **외부 격리** | DB 장애 시 게임 진행 | 거래소 API 장애 시 제한 제공 |
| **정규화 계층** | Event → DB Schema | External API → Internal Schema |
| **계약 안정성** | 운영 API 불변 | 클라이언트 API 불변 |
| **비동기 처리** | Kafka Event Stream | WebSocket Stream |

### 🎨 [React Object State Manager](https://github.com/1985jwlee/portpolio_react)

**Admin Dashboard 프로토타입**

| Main Portfolio | React Portfolio |
|----------------|-----------------|
| 서버 오브젝트 상태 관리 | UI 오브젝트 상태 관리 |
| Event Sourcing | State Management |
| Snapshot 복구 (서버) | 저장/불러오기 (클라이언트) |
| 운영 대시보드 설계 | 운영 도구 구현 |

> **핵심 메시지**: "설계 원칙은 도메인을 넘어 일반화 가능합니다"

---

## 📧 Contact

**Portfolio**: [GitHub Repository](https://github.com/1985jwlee)  
**Email**: [leejae.w.jl@icloud.com]  
**Blog**: [기술 블로그]

---

## 📝 License

이 문서는 설계 포트폴리오로, 학습 및 평가 목적으로 공개되었습니다.

---

## 🎓 최종 메시지

이 포트폴리오는 **코드를 작성하는 능력**이 아니라  
**시스템을 설계하고 판단하는 능력**을 증명합니다.

### 증명된 것:

✅ 실시간 시스템의 구조적 설계 능력  
✅ 장애를 격리하고 복구하는 전략  
✅ 확장 가능한 아키텍처 설계  
✅ 운영 가능성까지 고려한 시스템 설계  
✅ 조직에 남는 시스템을 만드는 사고방식  

### 검증 방법:

- 📖 [설계 결정 과정](docs/design-decisions.md): 모든 판단의 근거 명시
- 🔧 [운영 가이드](docs/operational-guide.md): 장애 시나리오별 대응 방안
- 📈 [확장 시나리오](docs/architecture-detail.md): 10배 성장 대응 전략
- 🚀 [구현 로드맵](docs/implementation-roadmap.md): 실제 구현 가능성 증명

---

**Last Updated**: 2025-01-15  

**Note**: 이 포트폴리오는 실제 게임 출시를 목적으로 하지 않으며,  
**시스템 설계 판단력과 아키텍처 사고**를 증명하기 위한 자료입니다.
