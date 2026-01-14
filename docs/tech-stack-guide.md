# 기술 스택 구현 가이드

[← 메인으로 돌아가기](../README.md)

---

## 📋 목차

1. [C# 게임 서버](#c-게임-서버)
2. [TypeScript 플랫폼 서버](#typescript-플랫폼-서버)
3. [Unity 클라이언트](#unity-클라이언트)
4. [인프라 설정](#인프라-설정)

---

## C# 게임 서버

### 프로젝트 구조

```
GameServer/
├── Program.cs                      # 진입점
├── Network/
│   ├── TcpServer.cs               # TCP 서버
│   ├── Session.cs                 # 클라이언트 세션
│   ├── PacketHandler.cs           # 패킷 처리
│   └── Serializer.cs              # MessagePack 직렬화
├── GameLoop/
│   ├── GameTick.cs                # 메인 게임 루프
│   └── CommandQueue.cs            # Command 큐
├── Domain/
│   ├── Player.cs                  # 플레이어 도메인
│   ├── World.cs                   # 게임 월드
│   └── Commands/
│       ├── MoveCommand.cs
│       └── ICommand.cs
├── Events/
│   ├── DomainEvent.cs             # 이벤트 베이스
│   ├── PlayerMovedEvent.cs
│   └── EventPublisher.cs          # Kafka 발행
└── Infrastructure/
    ├── RedisClient.cs             # Redis 연동
    └── KafkaProducer.cs           # Kafka 연동
```

### 핵심 구현

#### 1. TCP 서버

```csharp
public class TcpServer
{
    private readonly TcpListener _listener;
    private readonly Dictionary _sessions = new();
    
    public TcpServer(string host, int port)
    {
        _listener = new TcpListener(IPAddress.Parse(host), port);
    }
    
    public async Task StartAsync()
    {
        _listener.Start();
        Console.WriteLine($"[Server] Listening on {_listener.LocalEndpoint}");
        
        while (true)
        {
            var client = await _listener.AcceptTcpClientAsync();
            _ = HandleClientAsync(client); // Fire-and-forget
        }
    }
    
    private async Task HandleClientAsync(TcpClient client)
    {
        var session = new Session(client);
        _sessions[session.Id] = session;
        
        try
        {
            await session.StartReceiveAsync(OnPacketReceived);
        }
        finally
        {
            _sessions.Remove(session.Id);
            session.Dispose();
        }
    }
    
    private void OnPacketReceived(Session session, byte[] data)
    {
        // Packet → Command 변환
        var packet = MessagePackSerializer.Deserialize(data);
        var command = PacketHandler.ToCommand(packet, session);
        
        // Command Queue에 적재
        GameLoop.Instance.EnqueueCommand(command);
    }
}
```

#### 2. 게임 루프

```csharp
public class GameTick
{
    private readonly ConcurrentQueue _commandQueue = new();
    private readonly World _world = new();
    private readonly EventPublisher _eventPublisher;
    private const int TargetTickRate = 20; // 50ms
    
    public void EnqueueCommand(ICommand command)
    {
        _commandQueue.Enqueue(command);
    }
    
    public async Task RunAsync()
    {
        var interval = TimeSpan.FromMilliseconds(1000.0 / TargetTickRate);
        
        while (true)
        {
            var startTime = DateTime.UtcNow;
            
            // 1. Command 처리
            ProcessCommands();
            
            // 2. World Update
            _world.Update();
            
            // 3. Tick 시간 측정
            var elapsed = DateTime.UtcNow - startTime;
            if (elapsed > interval)
            {
                Console.WriteLine($"[Warning] Tick overrun: {elapsed.TotalMilliseconds}ms");
            }
            
            // 4. 다음 Tick까지 대기
            var delay = interval - elapsed;
            if (delay > TimeSpan.Zero)
            {
                await Task.Delay(delay);
            }
        }
    }
    
    private void ProcessCommands()
    {
        var processCount = Math.Min(_commandQueue.Count, 1000);
        
        for (int i = 0; i < processCount; i++)
        {
            if (!_commandQueue.TryDequeue(out var command))
                break;
            
            try
            {
                var events = command.Execute(_world);
                
                foreach (var evt in events)
                {
                    _eventPublisher.PublishAsync(evt);
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine($"[Error] Command execution failed: {ex.Message}");
            }
        }
    }
}
```

#### 3. Command 구현

```csharp
public class MoveCommand : ICommand
{
    public string PlayerId { get; set; }
    public Vector3 NewPosition { get; set; }
    
    public List Execute(World world)
    {
        var events = new List();
        var player = world.GetPlayer(PlayerId);
        
        if (player == null)
        {
            Console.WriteLine($"[Warning] Player not found: {PlayerId}");
            return events;
        }
        
        // Validation
        if (!ValidateMove(player, NewPosition))
        {
            Console.WriteLine($"[Reject] Invalid move: {PlayerId}");
            return events;
        }
        
        // State Change (메모리에서 즉시)
        var oldPosition = player.Position;
        player.Position = NewPosition;
        player.LastMoveTime = DateTime.UtcNow;
        
        // Domain Event 생성
        events.Add(new PlayerMovedEvent
        {
            EventId = Guid.NewGuid().ToString(),
            AggregateId = PlayerId,
            OccurredAt = DateTime.UtcNow,
            PlayerId = PlayerId,
            FromPosition = oldPosition,
            ToPosition = NewPosition
        });
        
        return events;
    }
    
    private bool ValidateMove(Player player, Vector3 newPosition)
    {
        var distance = Vector3.Distance(player.Position, newPosition);
        if (distance > player.MaxMoveDistance)
            return false;
        
        if (DateTime.UtcNow - player.LastMoveTime < player.MoveCooldown)
            return false;
        
        return true;
    }
}
```

#### 4. Kafka Producer

```csharp
public class KafkaProducer
{
    private readonly IProducer _producer;
    
    public KafkaProducer(string bootstrapServers)
    {
        var config = new ProducerConfig
        {
            BootstrapServers = bootstrapServers,
            Acks = Acks.Leader,
            EnableIdempotence = true
        };
        
        _producer = new ProducerBuilder(config).Build();
    }
    
    public async Task PublishAsync(DomainEvent evt)
    {
        try
        {
            var data = MessagePackSerializer.Serialize(evt);
            
            var message = new Message
            {
                Key = evt.AggregateId,
                Value = data
            };
            
            _ = _producer.ProduceAsync("game.events.player", message);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"[Warning] Kafka publish failed: {ex.Message}");
        }
    }
}
```

### 필수 패키지

```xml

  
  
  

```

---

## TypeScript 플랫폼 서버

### 프로젝트 구조

```
platform-server/
├── src/
│   ├── index.ts                   # 진입점
│   ├── kafka/
│   │   ├── consumer.ts            # Kafka Consumer
│   │   ├── eventHandler.ts        # 이벤트 처리
│   │   └── types.ts               # 이벤트 타입
│   ├── persistence/
│   │   ├── database.ts            # DB 연결
│   │   ├── repositories/
│   │   │   └── playerRepository.ts
│   │   └── schema.ts
│   ├── cache/
│   │   └── redis.ts               # Redis 클라이언트
│   └── api/
│       ├── server.ts              # Elysia 서버
│       └── routes/
│           └── players.ts
├── package.json
└── tsconfig.json
```

### 핵심 구현

#### 1. Kafka Consumer

```typescript
import { Kafka, Consumer } from 'kafkajs';
import { EventHandler } from './eventHandler';

export class KafkaConsumer {
  private kafka: Kafka;
  private consumer: Consumer;
  private eventHandler: EventHandler;
  
  constructor(brokers: string[]) {
    this.kafka = new Kafka({
      clientId: 'platform-server',
      brokers: brokers
    });
    
    this.consumer = this.kafka.consumer({
      groupId: 'platform-server-group'
    });
    
    this.eventHandler = new EventHandler();
  }
  
  async start() {
    await this.consumer.connect();
    console.log('[Kafka] Consumer connected');
    
    await this.consumer.subscribe({
      topics: ['game.events.player'],
      fromBeginning: false
    });
    
    await this.consumer.run({
      eachMessage: async ({ topic, partition, message }) => {
        if (!message.value) return;
        
        const event = JSON.parse(message.value.toString());
        await this.eventHandler.handle(event);
      }
    });
  }
}
```

#### 2. Event Handler

```typescript
import { Redis } from 'ioredis';
import { PlayerRepository } from '../persistence/repositories/playerRepository';

export class EventHandler {
  private redis: Redis;
  private playerRepo: PlayerRepository;
  
  constructor() {
    this.redis = new Redis({
      host: process.env.REDIS_HOST || 'localhost',
      port: 6379
    });
    
    this.playerRepo = new PlayerRepository();
  }
  
  async handle(event: DomainEvent) {
    // Idempotency 검증
    if (await this.isProcessed(event.eventId)) {
      console.log(`[Event] Already processed: ${event.eventId}`);
      return;
    }
    
    try {
      switch (event.type) {
        case 'PlayerMovedEvent':
          await this.handlePlayerMoved(event);
          break;
      }
      
      await this.markProcessed(event.eventId);
      
    } catch (error) {
      console.error(`[Event] Failed: ${event.eventId}`, error);
      await this.sendToDLQ(event, error);
    }
  }
  
  private async handlePlayerMoved(event: PlayerMovedEvent) {
    await this.playerRepo.saveMovement({
      playerId: event.playerId,
      fromPosition: event.fromPosition,
      toPosition: event.toPosition,
      occurredAt: event.occurredAt
    });
    
    console.log(`[Event] Player moved: ${event.playerId}`);
  }
  
  private async isProcessed(eventId: string): Promise {
    const exists = await this.redis.exists(`event:${eventId}`);
    return exists === 1;
  }
  
  private async markProcessed(eventId: string): Promise {
    await this.redis.setex(`event:${eventId}`, 3600, 'processed');
  }
}
```

#### 3. Database (Drizzle ORM)

```typescript
import { drizzle } from 'drizzle-orm/mysql2';
import mysql from 'mysql2/promise';
import * as schema from './schema';

export const connection = await mysql.createConnection({
  host: process.env.DB_HOST || 'localhost',
  user: process.env.DB_USER || 'root',
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME || 'game_platform'
});

export const db = drizzle(connection, { schema, mode: 'default' });
```

#### 4. Schema 정의

```typescript
import { mysqlTable, varchar, timestamp, json } from 'drizzle-orm/mysql-core';

export const playerMovements = mysqlTable('player_movements', {
  id: varchar('id', { length: 36 }).primaryKey(),
  playerId: varchar('player_id', { length: 36 }).notNull(),
  fromPosition: json('from_position').notNull(),
  toPosition: json('to_position').notNull(),
  occurredAt: timestamp('occurred_at').notNull(),
  createdAt: timestamp('created_at').defaultNow()
});
```

#### 5. API Server (Elysia)

```typescript
import { Elysia } from 'elysia';
import { PlayerRepository } from './persistence/repositories/playerRepository';

const playerRepo = new PlayerRepository();

const app = new Elysia()
  .get('/api/players/:playerId/movements', async ({ params }) => {
    const movements = await playerRepo.getPlayerMovements(params.playerId);
    return { data: movements };
  })
  .get('/health', () => ({ status: 'ok' }))
  .listen(3000);

console.log(`[API] Server running on port 3000`);
```

### 필수 패키지

```json
{
  "dependencies": {
    "elysia": "^0.8.0",
    "kafkajs": "^2.2.4",
    "drizzle-orm": "^0.29.0",
    "mysql2": "^3.6.5",
    "ioredis": "^5.3.2",
    "uuid": "^9.0.1"
  },
  "devDependencies": {
    "bun-types": "latest",
    "drizzle-kit": "^0.20.0"
  }
}
```

---

## Unity 클라이언트

### 프로젝트 구조

```
Assets/
├── Scripts/
│   ├── Network/
│   │   ├── NetworkClient.cs
│   │   └── PacketHandler.cs
│   ├── Player/
│   │   └── PlayerController.cs
│   └── Bootstrap/
│       └── GameBootstrap.cs
└── Scenes/
    └── GameScene.unity
```

### 핵심 구현

#### 1. Network Client

```csharp
using System;
using System.Net.Sockets;
using UnityEngine;
using MessagePack;

public class NetworkClient : MonoBehaviour
{
    private TcpClient _client;
    private NetworkStream _stream;
    private bool _isConnected;
    
    public event Action OnMoveConfirmed;
    
    public async void Connect(string host, int port)
    {
        try
        {
            _client = new TcpClient();
            await _client.ConnectAsync(host, port);
            _stream = _client.GetStream();
            _isConnected = true;
            
            Debug.Log($"[Network] Connected to {host}:{port}");
            _ = ReceiveLoop();
        }
        catch (Exception ex)
        {
            Debug.LogError($"[Network] Connection failed: {ex.Message}");
        }
    }
    
    public void SendMoveRequest(Vector3 newPosition)
    {
        if (!_isConnected) return;
        
        var packet = new MoveRequestPacket
        {
            Type = "MoveRequest",
            NewPosition = newPosition
        };
        
        var data = MessagePackSerializer.Serialize(packet);
        
        try
        {
            _stream.Write(data, 0, data.Length);
            Debug.Log($"[Network] Sent MoveRequest: {newPosition}");
        }
        catch (Exception ex)
        {
            Debug.LogError($"[Network] Send failed: {ex.Message}");
        }
    }
    
    private async System.Threading.Tasks.Task ReceiveLoop()
    {
        var buffer = new byte[4096];
        
        while (_isConnected)
        {
            try
            {
                var bytesRead = await _stream.ReadAsync(buffer, 0, buffer.Length);
                if (bytesRead == 0) break;
                
                ProcessPacket(buffer, bytesRead);
            }
            catch (Exception ex)
            {
                Debug.LogError($"[Network] Receive error: {ex.Message}");
                break;
            }
        }
    }
    
    private void ProcessPacket(byte[] data, int length)
    {
        var packet = MessagePackSerializer.Deserialize(data);
        
        if (packet.Type == "MoveResponse")
        {
            var response = packet as MoveResponsePacket;
            OnMoveConfirmed?.Invoke(response.Position);
        }
    }
}
```

#### 2. Player Controller

```csharp
using UnityEngine;

public class PlayerController : MonoBehaviour
{
    private NetworkClient _networkClient;
    private Vector3 _serverPosition;
    
    void Start()
    {
        _networkClient = FindObjectOfType();
        _networkClient.OnMoveConfirmed += OnServerMoveConfirmed;
        _serverPosition = transform.position;
    }
    
    void Update()
    {
        HandleInput();
    }
    
    private void HandleInput()
    {
        Vector3 moveDirection = Vector3.zero;
        
        if (Input.GetKeyDown(KeyCode.W))
            moveDirection = Vector3.forward;
        else if (Input.GetKeyDown(KeyCode.S))
            moveDirection = Vector3.back;
        else if (Input.GetKeyDown(KeyCode.A))
            moveDirection = Vector3.left;
        else if (Input.GetKeyDown(KeyCode.D))
            moveDirection = Vector3.right;
        
        if (moveDirection != Vector3.zero)
        {
            var newPosition = transform.position + moveDirection;
            _networkClient.SendMoveRequest(newPosition);
        }
    }
    
    private void OnServerMoveConfirmed(Vector3 serverPosition)
    {
        transform.position = serverPosition;
        _serverPosition = serverPosition;
        Debug.Log($"[Client] Move confirmed: {serverPosition}");
    }
}
```

---

## 인프라 설정

### Docker Compose

```yaml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  mongodb:
    image: mongo:7
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: password

  mysql:
    image: mysql:8
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: game_platform
```

### 실행 순서

```bash
# 1. 인프라 시작
docker-compose up -d

# 2. 게임 서버
cd game-server
dotnet run

# 3. 플랫폼 서버
cd platform-server
bun install
bun run dev

# 4. Unity 클라이언트
# Unity에서 GameScene 실행
```

---

[← 메인으로 돌아가기](../README.md)