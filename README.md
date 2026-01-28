# Event-driven Real-time Game Platform Architecture

> **실시간 판정은 메모리에서 끝나고, 기록과 복구는 비동기로 흡수되는 구조**

[![Architecture](https://img.shields.io/badge/Architecture-Event--Driven-blue)](docs/architecture-detail.md)
[![Status](https://img.shields.io/badge/Status-Design%20Complete-green)](docs/implementation-roadmap.md)
[![License](https://img.shields.io/badge/License-Portfolio-orange)](LICENSE)

-----

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

> “코드를 작성하는 능력이 아니라, 시스템을 설계하고 판단하는 능력을 보여줍니다.”

-----

## 🎯 왜 이 아키텍처인가?

### 많은 게임 서비스가 겪는 구조적 문제

```mermaid
graph TD
    A[사용자 증가] --> B[서버 복잡도 폭증]
    B --> C[운영 불가능]
    A --> D[실시간/기록 경계 불명확]
    D --> E[장애 영향 범위 예측 불가]
    E --> F[특정 개발자에게 지식 집중]
    F --> G[기능 추가 시 안정성 훼손]
    
    style A fill:#ff6b6b
    style C fill:#ff6b6b
    style G fill:#ff6b6b
```

### 핵심 판단

> **문제의 핵심은 기술 부족이 아니라 구조 부재입니다.**

이 포트폴리오는 위 문제를 **구조적으로 해결**하는 과정을 보여줍니다.

-----

## 🏗️ 3가지 핵심 설계 결정

### 1️⃣ 실시간 판정과 기록의 완전한 분리

```mermaid
sequenceDiagram
    participant C as Client
    participant GS as Game Server
    participant M as Memory
    participant K as Kafka
    participant PS as Platform Server
    participant DB as Database
    
    C->>GS: MoveCommand
    GS->>GS: Validate
    GS->>M: Update State
    Note over M: 메모리에서 즉시 확정
    GS->>C: Response (< 50ms)
    GS--)K: Domain Event (Fire-and-Forget)
    Note over K: 비동기 처리
    K->>PS: Event Delivery
    PS->>DB: Persist
```

**판단 근거**:

```mermaid
graph LR
    A[게임플레이] -->|독립| B[DB 지연 영향 없음]
    A -->|독립| C[장애 격리]
    D[이벤트 스트림] --> E[신규 서비스 추가 용이]
    
    style A fill:#51cf66
    style D fill:#51cf66
```

**실무 시나리오**:

```
Kafka 다운 발생:
❌ 잘못된 설계: 게임 서버도 멈춤
✅ 이 설계: 게임은 계속, 이벤트는 메모리 버퍼링
```

-----

### 2️⃣ Server-authoritative 구조

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    participant M as Memory State
    
    Note over C: W키 입력
    C->>S: "이동하고 싶어요" (의도만 전달)
    S->>S: 검증 (거리, 속도, 충돌)
    alt Valid
        S->>M: Update Position
        S->>C: "승인, 새 위치는 X"
        Note over C: 서버 응답 후 화면 갱신
    else Invalid
        S->>C: "거부"
        Note over C: 원위치 유지
    end
```

**판단 근거**:

```mermaid
graph TB
    subgraph "Server Authority"
        SA[치트 원천 차단]
        SB[상태 일관성 보장]
        SC[클라이언트 교정 가능]
    end
    
    subgraph "Trade-off"
        TA[구현 복잡도 증가]
        TB[네트워크 지연 체감]
    end
    
    SA --> Decision[장기 운영 안정성 선택]
    SB --> Decision
    SC --> Decision
    TA -.감수.-> Decision
    TB -.감수.-> Decision
    
    style Decision fill:#51cf66,stroke:#2f9e44,stroke-width:3px
```

**결론**: 복잡해서가 아니라 안정성을 위해 선택

-----

### 3️⃣ 의도적으로 선택하지 않은 것들

```mermaid
mindmap
  root((의도적 비선택))
    게임 서버 직접 DB 접근
      GameLoop DB 의존
      장애 전파
    모든 처리 동기화
      선형 성능 저하
      확장 비용 급증
    초기 MSA
      Over-engineering
      운영 복잡도 과다
    UDP 프로토콜
      포트폴리오 목적상 불필요
      아키텍처 증명이 목표
```

**핵심 원칙**:

> **“지금 필요하지 않으면, 지금 만들지 않는다”**

-----

## 📊 시스템 아키텍처

### 전체 구성도

```mermaid
graph TB
    subgraph "Client Layer"
        UNITY[Unity Client<br/>Server-Authoritative]
    end
    
    subgraph "Game Server Layer (C#)"
        TCP[TCP Socket Server]
        GAMELOOP[GameLoop Ticker<br/>Fixed Update 50ms]
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

```mermaid
graph LR
    subgraph "Command (의도)"
        C1[Client → Server]
        C2["'해달라' 요청"]
        C3[실패 가능]
        C4[게임 로직 실행]
    end
    
    subgraph "Event (사실)"
        E1[Server → Platform]
        E2["'이미 일어났다'"]
        E3[실패 불가능]
        E4[기록 및 연동]
    end
    
    C1 --> C2 --> C3 --> C4
    E1 --> E2 --> E3 --> E4
    
    style C2 fill:#fff4e1
    style E2 fill:#e1ffe1
```

|구분    |Command        |Domain Event     |
|------|---------------|-----------------|
|**의미**|“해달라” (요청)     |“이미 일어났다” (사실)   |
|**시점**|미래             |과거               |
|**실패**|가능             |불가능 (이미 발생)      |
|**흐름**|Client → Server|Server → Platform|
|**용도**|게임 로직 실행       |기록 및 연동          |

-----

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

> “게임플레이는 어떤 백엔드 장애에도 멈추지 않는다”

### 복구 전략

```mermaid
flowchart TD
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

-----

## 🔄 핵심 흐름: Command → Event

### 플레이어 이동 시나리오

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
1. 상태는 메모리에서 이미 확정됨
1. 기록 실패가 게임플레이를 막지 않음

-----

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

-----

## 🛠️ 기술 스택

### 게임 서버 (C#)

- **언어**: C# .NET 8.0
- **프로토콜**: TCP/IP
- **직렬화**: MessagePack
- **패턴**: Command Pattern, Event Sourcing
- **캐시**: Redis (Hot Snapshot)
- **이벤트**: Kafka Producer

### 플랫폼 서버 (TypeScript)

- **런타임**: Bun.js
- **프레임워크**: ElysiaJS
- **ORM**: Drizzle
- **DB**: MySQL (정형), MongoDB (비정형)
- **이벤트**: Kafka Consumer

### 클라이언트 (Unity)

- **엔진**: Unity 2022.3 LTS
- **구조**: Server-authoritative
- **프로토콜**: TCP Socket

-----

## 📚 상세 문서

|문서                                        |설명               |대상 독자      |
|------------------------------------------|-----------------|-----------|
|[아키텍처 상세](docs/architecture-detail.md)    |전체 시스템 구조 및 설계 원칙|백엔드 엔지니어   |
|[설계 결정 과정](docs/design-decisions.md)      |왜 이렇게 설계했는가      |테크 리드, CTO |
|[운영 가이드](docs/operational-guide.md)       |장애 대응 및 모니터링     |DevOps, SRE|
|[구현 로드맵](docs/implementation-roadmap.md) ⭐|단계별 구현 계획        |개발자, PM    |

-----

## 🗺️ 구현 로드맵

```mermaid
gantt
    title Implementation Roadmap
    dateFormat YYYY-MM-DD
    section Phase 0
    설계 확정 (문서)              :done, p0, 2025-01-15, 2d
    section Phase 1
    MVP 구현 (핵심 흐름)          :active, p1, 2025-01-17, 14d
    section Phase 2
    이벤트 신뢰성                  :p2, after p1, 5d
    section Phase 3
    Hot/Cold Snapshot             :p3, after p2, 7d
    section Phase 4
    포트폴리오 정리                :p4, after p3, 3d
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

> “더 만들 수 있다”가 아니라 **“언제 멈춰야 하는지 안다”**를 증명하기 위해

-----

## 💡 설계 철학

### 배운 교훈

**기술적 교훈**:

1. **복잡도는 비용이다**

- “할 수 있다”와 “해야 한다”는 다름
- 복잡한 구조는 반드시 그만한 가치를 제공해야 함

1. **장애는 언제나 발생한다**

- 장애를 막는 것보다 격리하는 것이 현실적
- “장애 시 어떻게 되는가”가 설계의 핵심

1. **확장은 선형적이어야 한다**

- 사용자 2배 → 비용 2배가 이상적
- 비선형 확장은 지속 불가능

**조직 관점 교훈**:

1. **문서화는 필수다**

- 개인의 지식은 조직에 남지 않음
- 구조를 설명할 수 없으면 좋은 구조가 아님

1. **운영 가능성이 구현보다 중요하다**

- 만들 수 있어도 운영할 수 없으면 의미 없음
- 운영팀이 이해할 수 있는 구조여야 함

1. **인수인계 가능한 시스템**

- 특정 개발자에게 의존하는 구조는 위험
- 시스템 자체가 설명할 수 있어야 함

-----

## 🔗 관련 포트폴리오

### 🎨 [Coin Data API — Platform Server in Practice](https://github.com/1985jwlee/portpolio_coindataapi)

**동일한 원칙의 비게임 도메인 적용 사례**

```mermaid
graph TB
    subgraph "Game Domain"
        G1[실시간 판정]
        G2[이벤트 기록]
        G3[장애 격리]
    end
    
    subgraph "Financial Domain"
        F1[실시간 데이터 수집]
        F2[정규화 계층]
        F3[외부 API 격리]
    end
    
    P[공통 설계 원칙]
    
    P -.-> G1
    P -.-> G2
    P -.-> G3
    P -.-> F1
    P -.-> F2
    P -.-> F3
    
    style P fill:#4A90E2,stroke:#2E5C8A,stroke-width:3px,color:#fff
```

|원칙        |게임 서버 (본 프로젝트)    |Coin API                      |
|----------|------------------|------------------------------|
|**외부 격리** |DB 장애 시 게임 진행     |거래소 API 장애 시 제한 제공            |
|**정규화 계층**|Event → DB Schema |External API → Internal Schema|
|**계약 안정성**|운영 API 불변         |클라이언트 API 불변                  |
|**비동기 처리**|Kafka Event Stream|WebSocket Stream              |


> **핵심 메시지**: “설계 원칙은 도메인을 넘어 일반화 가능합니다”

-----

## 📧 Contact

**GitHub**: [@1985jwlee](https://github.com/1985jwlee)  
**Email**: leejae.w.jl@icloud.com

-----

## 📝 License

이 문서는 설계 포트폴리오로, 학습 및 평가 목적으로 공개되었습니다.

-----

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
- 📈 [아키텍처 상세](docs/architecture-detail.md): 10배 성장 대응 전략
- 🚀 [구현 로드맵](docs/implementation-roadmap.md): 실제 구현 가능성 증명

-----

**Last Updated**: 2025-01-28

**Note**: 이 포트폴리오는 실제 게임 출시를 목적으로 하지 않으며,  
**시스템 설계 판단력과 아키텍처 사고**를 증명하기 위한 자료입니다.​​​​​​​​​​​​​​​​