# Event-driven Real-time Game Platform Architecture

> **실시간 판정은 메모리에서 끝나고, 기록과 복구는 비동기로 흡수되는 구조**

[![Architecture](https://img.shields.io/badge/Architecture-Event--Driven-blue)](docs/architecture-detail.md)
[![Status](https://img.shields.io/badge/Status-Design%20Complete-green)](docs/implementation-roadmap.md)
[![License](https://img.shields.io/badge/License-Portfolio-orange)](LICENSE)

---

## 📌 이 포트폴리오가 증명하는 것

이 문서는 **코드 구현 능력**이 아니라 **시스템 설계 판단력**을 보여주기 위한 포트폴리오입니다.

```
✓ 실시간 시스템에서의 책임 분리 능력
✓ Server-authoritative 구조에 대한 이해
✓ 이벤트 기반 아키텍처 설계 사고
✓ 장애, 복구, 운영까지 포함한 구조적 판단
✓ 개인이 아닌 조직에 남는 시스템을 설계하는 관점
```

**대상 독자**: CTO, 테크 리드, 시니어 백엔드/서버 엔지니어

---

## 🎯 핵심 설계 결정 3가지

### 1️⃣ 실시간 판정과 기록의 완전한 분리

```
[ 게임 서버 ]
  ↓ 메모리에서 즉시 판정 (50ms)
  ↓ Domain Event 발행 (Fire-and-Forget)
[ Kafka ]
  ↓ 비동기 처리
[ 플랫폼 서버 ]
  ↓ DB 저장, 통계, 운영
```

**판단 근거**:
- 게임플레이는 DB 지연의 영향을 받지 않음
- 장애 격리: Kafka/DB 다운 시에도 게임 진행
- 확장성: 이벤트 스트림으로 신규 서비스 추가 가능

### 2️⃣ Server-authoritative 구조

```
클라이언트: "W키를 눌렀어요" (의도만 전달)
    ↓
서버: "검증 → 승인 → 상태 변경 → 응답"
    ↓
클라이언트: 서버 응답을 받아야만 화면 갱신
```

**판단 근거**:
- 치트 방지는 구조적으로 해결
- 클라이언트는 언제든 서버 기준으로 교정 가능
- 복잡해서가 아니라 안정성을 위해 선택

### 3️⃣ 의도적으로 선택하지 않은 것들

```
❌ 게임 서버 직접 DB 접근 → GameLoop이 DB에 의존하게 됨
❌ 모든 처리를 동기로 → 사용자 증가 시 선형적 성능 저하
❌ 초기부터 MSA → 운영 복잡도 대비 얻는 가치 부족
❌ UDP 프로토콜 → 포트폴리오 목적상 TCP로 충분
```

**핵심 원칙**: "지금 필요하지 않으면, 지금 만들지 않는다"

---

## 🏗️ 시스템 아키텍처

### 전체 구성도

```
┌─────────────────┐
│ Unity Client    │
│ (Server Auth)   │
└────────┬────────┘
         │ TCP/IP
         ↓
┌─────────────────┐
│ Game Server     │
│ (C#)            │
│                 │
│ • TCP Socket    │
│ • GameLoop Tick │
│ • Memory State  │
│ • Domain Events │
└────────┬────────┘
         │ Kafka
         ↓
┌─────────────────┐
│ Event Stream    │
│ (Kafka)         │
└────────┬────────┘
         │
         ↓
┌─────────────────┐
│ Platform Server │
│ (TypeScript)    │
│                 │
│ • Event Handler │
│ • Persistence   │
│ • REST API      │
└────────┬────────┘
         │
         ↓
┌─────────────────┐
│ Storage Layer   │
│                 │
│ • Redis (Hot)   │
│ • MongoDB (Cold)│
│ • MySQL (OLTP)  │
└─────────────────┘
```

### 책임 분리

| 구분 | 게임 서버 | 플랫폼 서버 |
|------|-----------|-------------|
| **역할** | 실시간 판정 | 기록 및 운영 |
| **응답 시간** | < 50ms | 제한 없음 |
| **DB 의존** | ❌ 없음 | ✅ 있음 |
| **장애 영향** | 게임 중단 | 게임 무영향 |

---

## 🛡️ 장애 대응 설계

### 장애 영향도 매트릭스

| 장애 대상 | 게임플레이 | 기록 | 운영 API | 복구 시간 |
|-----------|------------|------|----------|-----------|
| 게임 서버 | 🔴 중단 | 🟡 일시 중단 | 🟢 정상 | 수초 (Redis) |
| Redis | 🟡 순간 지연 | 🟢 정상 | 🟢 정상 | 수초 |
| Kafka | 🟢 정상 | 🟡 일시 중단 | 🟢 정상 | 즉시 |
| MySQL | 🟢 정상 | 🟡 일시 중단 | 🔴 일부 실패 | 즉시 |
| 플랫폼 서버 | 🟢 정상 | 🟡 일시 중단 | 🔴 중단 | 수초 |

**설계 철학**: 게임플레이는 어떤 백엔드 장애에도 멈추지 않습니다.

### 복구 전략

```
게임 서버 크래시 시:
1순위: Redis Hot Snapshot (수초)
    ↓ 실패 시
2순위: MongoDB Cold Snapshot (수분)
    ↓ 실패 시
3순위: Event Replay (수분~수십분)
```

---

## 📊 확장 시나리오

### Zone 기반 수평 확장

```
100 CCU:      [ Zone 1 ]
              
1,000 CCU:    [ Zone 1 ] [ Zone 2 ]
              
10,000 CCU:   [ Zone Coordinator ]
                    ↓
              [ Zone 1-10 ]
```

### 비즈니스 모델 확장

```
현재 (B2C):
[ Game Server ] → [ 자사 플랫폼 ]

확장 (B2B):
[ Core Game Server ]
    ↓ Event Stream
    ├── [ Tenant A Platform ]
    ├── [ Tenant B Platform ]
    └── [ Tenant C Platform ]
```

**핵심**: 게임 서버 코드 수정 없이 확장 가능

---

## 🛠️ 기술 스택

### 게임 서버
- **언어**: C#
- **프로토콜**: TCP/IP
- **직렬화**: MessagePack
- **캐시**: Redis
- **이벤트**: Kafka

### 플랫폼 서버
- **런타임**: bun.js
- **프레임워크**: ElysiaJS
- **ORM**: Drizzle
- **DB**: MySQL, MongoDB
- **이벤트**: Kafka

### 클라이언트
- **엔진**: Unity
- **구조**: Server-authoritative

---

## 📚 상세 문서

| 문서 | 내용 |
|------|------|
| [아키텍처 상세](docs/architecture-detail.md) | 전체 시스템 구조 및 설계 원칙 |
| [설계 결정 과정](docs/design-decisions.md) | 왜 이렇게 설계했는가 |
| [운영 가이드](docs/operational-guide.md) | 장애 대응 및 모니터링 |
| [구현 로드맵](docs/implementation-roadmap.md) | 단계별 구현 계획 ⭐ |
| [기술 스택 가이드](docs/tech-stack-guide.md) | 언어별 구현 예시 |

---

## 🗺️ 구현 로드맵

```
Phase 0: 설계 확정               ✅ 완료
Phase 1: MVP 구현 (핵심 흐름)     🔄 진행 예정
Phase 2: 이벤트 신뢰성            📋 계획
Phase 3: Hot/Cold Snapshot       📋 계획
Phase 4: 포트폴리오 정리          📋 계획
```

**예상 완료 기간**: 3~4주 (Phase 1 MVP까지는 1~2주)

**MVP 범위**:
- TCP 게임 서버 (C#)
- Command → Domain → Event 흐름
- Kafka Producer/Consumer
- 간단한 상태 변경 (이동)
- TypeScript 플랫폼 서버
- Unity 테스트 클라이언트

---

## 🎓 이 포트폴리오를 통해 배운 것

### 기술적 학습
- 실시간 시스템에서의 동기/비동기 경계 설정
- 이벤트 기반 아키텍처의 장단점과 트레이드오프
- 장애 격리를 위한 구조적 설계

### 설계 철학
- "지금 필요하지 않으면 만들지 않는다"
- "복잡도는 비용이다"
- "운영 가능성이 구현보다 중요하다"

### 조직 관점
- 개인의 지식이 아닌 시스템으로 남기기
- 인수인계 가능한 구조 설계
- 문서화의 중요성

---

## 📧 Contact

**Portfolio**: [GitHub Repository]  
**Email**: [이메일]  
**Blog**: [기술 블로그]

---

## 📝 License

이 문서는 설계 포트폴리오로, 학습 및 평가 목적으로 공개되었습니다.

---

**Last Updated**: 2024.01.14

**Note**: 이 포트폴리오는 실제 게임 출시를 목적으로 하지 않으며, 시스템 설계 능력을 증명하기 위한 자료입니다.