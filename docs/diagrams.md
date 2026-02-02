# 시스템 아키텍처 다이어그램

[← 메인으로 돌아가기](../README.md)

---

## 📋 다이어그램 사용 가이드

### README.md에 포함할 다이어그램
1. **전체 시스템 아키텍처** - 첫인상용
2. **장애 영향도 맵** - 안정성 증명용

### 면접/프레젠테이션용
1. **Command/Event 처리 흐름** - 설계 설명용
2. **장애 복구 플로우** - 운영 관점 설명용

### 기술 문서용
1. **데이터 흐름** - 아키텍처 상세 설명용
2. **배포 아키텍처** - 확장성 설명용

---

## 📋 목차

1. [전체 시스템 아키텍처](#전체-시스템-아키텍처)
2. [Command/Event 처리 흐름](#commandevent-처리-흐름)
3. [장애 복구 플로우](#장애-복구-플로우)
4. [데이터 흐름](#데이터-흐름)
5. [배포 아키텍처](#배포-아키텍처)

---

## 전체 시스템 아키텍처

### 컴포넌트 구성도

```mermaid
graph TB
    subgraph "Client Layer"
        UC[Unity Client<br/>Server-authoritative]
    end
    
    subgraph "Game Server Layer"
        GS[C# Game Server<br/>TCP/IP]
        GM[GameLoop Tick<br/>50ms]
        MS[Memory State<br/>Player/World]
    end
    
    subgraph "Event Stream Layer"
        KF[Kafka<br/>Event Stream]
        T1[game.events.player]
        T2[game.events.combat]
        T3[game.snapshot]
    end
    
    subgraph "Platform Server Layer"
        PS[TypeScript Platform<br/>bun.js]
        EH[Event Handler<br/>Consumer]
        API[REST API<br/>ElysiaJS]
    end
    
    subgraph "Storage Layer"
        RD[(Redis<br/>Hot Snapshot<br/>TTL: 5min)]
        MG[(MongoDB<br/>Cold Snapshot<br/>Permanent)]
        MY[(MySQL<br/>Persistent Data<br/>ACID)]
    end
    
    UC -->|TCP Request| GS
    GS -->|Response| UC
    GS --> GM
    GM --> MS
    GS -->|Fire-and-Forget| KF
    KF --> T1
    KF --> T2
    KF --> T3
    T1 --> EH
    T2 --> EH
    T3 --> EH
    EH --> PS
    PS --> API
    GS -.->|Save| RD
    GS -.->|Save| MG
    PS -->|Persist| MY
```

---

## Command/Event 처리 흐름

### 패킷부터 DB까지의 완전한 여정

```mermaid
sequenceDiagram
    participant C as Unity Client
    participant GS as Game Server
    participant M as Memory (World)
    participant K as Kafka
    participant PS as Platform Server
    participant DB as Database
    
    Note over C,DB: 1. 클라이언트 요청
    C->>GS: MoveRequest(newPosition)
    
    Note over GS: 2. 검증 & 판정
    GS->>GS: Validate Move
    GS->>GS: Check Distance
    GS->>GS: Check Cooldown
    
    Note over GS,M: 3. 상태 변경 (메모리)
    GS->>M: player.Position = newPosition
    M-->>GS: State Updated
    
    Note over GS,C: 4. 즉시 응답
    GS->>C: MoveResponse(confirmedPosition)
    
    Note over GS,K: 5. Domain Event 발행 (비동기)
    GS->>K: PlayerMovedEvent
    Note over K: Fire-and-Forget<br/>게임 서버는 대기하지 않음
    
    Note over K,PS: 6. Event 소비
    K->>PS: PlayerMovedEvent
    PS->>PS: Idempotency Check
    
    Note over PS,DB: 7. DB 영속화
    PS->>DB: INSERT player_movements
    DB-->>PS: Success
    
    Note over C,DB: ✅ 전체 흐름 완료
```

### 핵심 타이밍

```mermaid
gantt
    title 처리 타이밍 분석
    dateFormat X
    axisFormat %L ms
    
    section Client
    요청 전송    :0, 5
    응답 대기    :5, 50
    화면 갱신    :50, 60
    
    section Game Server
    패킷 수신    :5, 10
    검증        :10, 15
    상태 변경    :15, 20
    응답 전송    :20, 50
    Event 발행  :20, 25
    
    section Kafka
    Event 저장  :25, 100
    
    section Platform
    Event 소비  :100, 150
    DB 저장     :150, 250
```

---

## 장애 복구 플로우

### 게임 서버 크래시 복구 시나리오

```mermaid
flowchart TD
    Start([게임 서버 크래시]) --> Detect[Health Check 실패 감지]
    Detect --> NewServer[새 서버 인스턴스 시작]
    
    NewServer --> CheckRedis{Redis Snapshot<br/>존재?}
    
    CheckRedis -->|Yes| LoadRedis[Redis 스냅샷 로드]
    LoadRedis --> RestoreRedis[플레이어 상태 복원]
    RestoreRedis --> RecoveredFast([✅ 복구 완료<br/>RTO: 10초])
    
    CheckRedis -->|No| CheckMongo{MongoDB Snapshot<br/>존재?}
    
    CheckMongo -->|Yes| LoadMongo[MongoDB 스냅샷 로드]
    LoadMongo --> RestoreMongo[플레이어 상태 복원]
    RestoreMongo --> RecoveredSlow([⚠️ 복구 완료<br/>RTO: 2-3분])
    
    CheckMongo -->|No| InitState[초기 상태로 시작]
    InitState --> EventReplay[Kafka Event Replay]
    EventReplay --> RecoveredManual([🔧 수동 복구<br/>RTO: 5-10분])
```

### 장애 영향도 맵

```mermaid
graph LR
    subgraph "장애 발생"
        GS[게임 서버<br/>DOWN]
        RD[Redis<br/>DOWN]
        KF[Kafka<br/>DOWN]
        MY[MySQL<br/>DOWN]
        PS[플랫폼 서버<br/>DOWN]
    end
    
    subgraph "게임플레이 영향"
        GP1[🔴 중단]
        GP2[🟡 순간 지연]
        GP3[🟢 정상]
        GP4[🟢 정상]
        GP5[🟢 정상]
    end
    
    subgraph "기록 영향"
        RC1[🟡 일시 중단]
        RC2[🟢 정상]
        RC3[🟡 일시 중단]
        RC4[🟡 일시 중단]
        RC5[🟡 일시 중단]
    end
    
    subgraph "복구 시간"
        RT1[수초<br/>Redis]
        RT2[수초<br/>Failover]
        RT3[즉시<br/>Buffer]
        RT4[즉시<br/>Queue]
        RT5[수초<br/>Restart]
    end
    
    GS --> GP1 --> RC1 --> RT1
    RD --> GP2 --> RC2 --> RT2
    KF --> GP3 --> RC3 --> RT3
    MY --> GP4 --> RC4 --> RT4
    PS --> GP5 --> RC5 --> RT5
```

---

## 데이터 흐름

### 실시간 데이터 vs 영속 데이터

```mermaid
flowchart TB
    subgraph "실시간 경로 (Hot Path)"
        Client[Client Request]
        GameServer[Game Server<br/>Memory]
        Response[Immediate Response<br/>< 50ms]
        
        Client --> GameServer
        GameServer --> Response
        Response --> Client
    end
    
    subgraph "비동기 경로 (Cold Path)"
        Event[Domain Event]
        Kafka[Kafka Stream]
        Platform[Platform Server]
        
        GameServer -.->|Fire-and-Forget| Event
        Event --> Kafka
        Kafka --> Platform
    end
    
    subgraph "저장소 계층"
        Redis[(Redis<br/>Hot Snapshot<br/>5-10초 주기)]
        Mongo[(MongoDB<br/>Cold Snapshot<br/>1-5분 주기)]
        MySQL[(MySQL<br/>Persistent<br/>Event 기반)]
        
        GameServer -.->|Async| Redis
        GameServer -.->|Async| Mongo
        Platform -->|Sync| MySQL
    end
```

### 스냅샷 저장 전략

```mermaid
graph TB
    subgraph "게임 서버"
        Player[Player State<br/>in Memory]
    end
    
    subgraph "Hot Snapshot (Redis)"
        R1[5초마다 저장]
        R2[TTL: 5분]
        R3[즉시 복구용]
    end
    
    subgraph "Cold Snapshot (MongoDB)"
        M1[1분마다 저장]
        M2[영구 보관]
        M3[분석 & 복구용]
        M4[Checksum 검증]
    end
    
    subgraph "Event Store (Kafka)"
        K1[모든 변경사항]
        K2[Event Replay 가능]
        K3[Audit Trail]
    end
    
    Player -->|Every 5s| R1
    R1 --> R2
    R2 --> R3
    
    Player -->|Every 1m| M1
    M1 --> M2
    M2 --> M3
    M3 --> M4
    
    Player -->|Every Change| K1
    K1 --> K2
    K2 --> K3
```

---

## 배포 아키텍처

### Zone 기반 수평 확장

```mermaid
graph TB
    subgraph "Load Balancer"
        LB[Load Balancer<br/>Session Routing]
    end
    
    subgraph "Game Server Cluster"
        Z1[Zone 1<br/>100-300 CCU]
        Z2[Zone 2<br/>100-300 CCU]
        Z3[Zone 3<br/>100-300 CCU]
        ZN[Zone N<br/>...]
    end
    
    subgraph "Shared Services"
        Redis[(Redis Cluster)]
        Kafka[Kafka Cluster<br/>3 Brokers]
        Mongo[(MongoDB<br/>Replica Set)]
    end
    
    subgraph "Platform Services"
        PS1[Platform Server 1]
        PS2[Platform Server 2]
        PSN[Platform Server N]
    end
    
    LB --> Z1
    LB --> Z2
    LB --> Z3
    LB --> ZN
    
    Z1 -.-> Redis
    Z2 -.-> Redis
    Z3 -.-> Redis
    ZN -.-> Redis
    
    Z1 --> Kafka
    Z2 --> Kafka
    Z3 --> Kafka
    ZN --> Kafka
    
    Z1 -.-> Mongo
    Z2 -.-> Mongo
    Z3 -.-> Mongo
    ZN -.-> Mongo
    
    Kafka --> PS1
    Kafka --> PS2
    Kafka --> PSN
```

### 무중단 배포 (Rolling Update)

```mermaid
sequenceDiagram
    participant LB as Load Balancer
    participant V1 as Game Server v1
    participant V2 as Game Server v2 (New)
    participant Redis as Redis Snapshot
    
    Note over LB,Redis: 1. 새 버전 서버 시작
    V2->>V2: Start & Health Check
    V2->>LB: Ready
    
    Note over LB,V2: 2. 신규 유저만 V2로 라우팅
    LB->>V2: New Connections
    LB->>V1: Existing Connections (유지)
    
    Note over V1: 3. V1 서버 Graceful Shutdown
    V1->>V1: Stop accepting new connections
    V1->>V1: Wait for players to leave
    
    Note over V1,Redis: 4. 플레이어 0명 시 종료
    V1->>Redis: Save final snapshot
    V1->>V1: Shutdown
    
    Note over LB,V2: 5. V2 서버만 운영
    LB->>V2: All Connections
    
    Note over LB,V2: ✅ 무중단 배포 완료
```

---

## 확장 시나리오

### CCU 증가에 따른 확장

```mermaid
graph TB
    subgraph "100 CCU"
        Z100[Zone 1<br/>100 CCU]
    end
    
    subgraph "1,000 CCU"
        Z1K1[Zone 1<br/>300 CCU]
        Z1K2[Zone 2<br/>300 CCU]
        Z1K3[Zone 3<br/>300 CCU]
        Z1K4[Zone 4<br/>100 CCU]
    end
    
    subgraph "10,000 CCU"
        Coord[Zone Coordinator]
        Z10K1[Zone 1-5<br/>1,500 CCU]
        Z10K2[Zone 6-10<br/>1,500 CCU]
        Z10K3[Zone 11-20<br/>3,000 CCU]
        Z10K4[Zone 21-30<br/>3,000 CCU]
        Z10K5[Zone 31-35<br/>1,000 CCU]
    end
    
    Z100 -.->|Scale Out| Z1K1
    Z1K1 -.->|Scale Out| Coord
    
    Coord --> Z10K1
    Coord --> Z10K2
    Coord --> Z10K3
    Coord --> Z10K4
    Coord --> Z10K5
```

### B2B 비즈니스 모델 확장

```mermaid
graph TB
    subgraph "Core Platform"
        Core[Game Server Core<br/>이벤트 발행자]
        Stream[Event Stream<br/>Kafka]
    end
    
    subgraph "Tenant A"
        A1[Custom Platform]
        A2[Custom DB]
        A3[Custom API]
    end
    
    subgraph "Tenant B"
        B1[Custom Platform]
        B2[Custom DB]
        B3[Custom API]
    end
    
    subgraph "Tenant C"
        C1[Custom Platform]
        C2[Custom DB]
        C3[Custom API]
    end
    
    Core --> Stream
    Stream --> A1
    Stream --> B1
    Stream --> C1
    
    A1 --> A2
    A1 --> A3
    
    B1 --> B2
    B1 --> B3
    
    C1 --> C2
    C1 --> C3
```

---

## 상태 머신 (플레이어 생명주기)

```mermaid
stateDiagram-v2
    [*] --> Disconnected
    
    Disconnected --> Connecting : Connect Request
    Connecting --> Connected : Auth Success
    Connecting --> Disconnected : Auth Failed
    
    Connected --> InGame : Enter Game
    InGame --> Connected : Exit Game
    
    InGame --> Moving : Move Command
    Moving --> InGame : Move Complete
    
    InGame --> Fighting : Attack Command
    Fighting --> InGame : Combat End
    Fighting --> Dead : HP = 0
    
    Dead --> InGame : Respawn
    
    Connected --> Disconnected : Disconnect
    InGame --> Disconnected : Network Error
    Fighting --> Disconnected : Network Error
    
    Disconnected --> [*]
```

---

[← 메인으로 돌아가기](../README.md)