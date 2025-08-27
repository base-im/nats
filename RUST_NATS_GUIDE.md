# Rust 访问 NATS 集群指南

## 连接信息

### NATS 服务端点
- **主节点**: `nats://localhost:4222` (对外暴露)
- **集群内部节点**: 
  - `nats://nats-1:4222`
  - `nats://nats-2:4222`
  - `nats://nats-3:4222`

### 认证信息
```rust
// 只读用户
username: "im_readonly"
password: "CHANGE_ME_STRONG_RO"
permissions: 只能订阅，不能发布（除了 _INBOX.>）

// 读写用户
username: "im_service"  
password: "CHANGE_ME_STRONG_RW"
permissions: 可以发布和订阅所有主题
```

## Cargo.toml 依赖配置

```toml
[dependencies]
# 异步运行时
tokio = { version = "1", features = ["full"] }

# NATS 客户端
async-nats = "0.33"  # 推荐：官方异步客户端

# 或使用同步客户端
# nats = "0.24"

# 序列化（可选）
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# 日志（可选）
tracing = "0.1"
tracing-subscriber = "0.3"
```

## 连接配置示例

### 1. 基础连接（带认证）

```rust
use async_nats;

#[tokio::main]
async fn main() -> Result<(), async_nats::Error> {
    // 连接到 NATS（带用户密码认证）
    let client = async_nats::ConnectOptions::new()
        .user_and_password("im_service".into(), "CHANGE_ME_STRONG_RW".into())
        .connect("nats://localhost:4222")
        .await?;
    
    println!("Connected to NATS!");
    Ok(())
}
```

### 2. 高可用连接（多服务器）

```rust
use async_nats;

#[tokio::main]
async fn main() -> Result<(), async_nats::Error> {
    // 连接到多个服务器，自动故障转移
    let client = async_nats::ConnectOptions::new()
        .user_and_password("im_service".into(), "CHANGE_ME_STRONG_RW".into())
        .connect(vec![
            "nats://localhost:4222",
            // 如果在 Docker 网络内部，可以使用：
            // "nats://nats-1:4222",
            // "nats://nats-2:4222",
            // "nats://nats-3:4222",
        ])
        .await?;
    
    println!("Connected to NATS cluster!");
    Ok(())
}
```

### 3. 完整连接配置

```rust
use async_nats;
use std::time::Duration;

#[tokio::main]
async fn main() -> Result<(), async_nats::Error> {
    let client = async_nats::ConnectOptions::new()
        // 认证
        .user_and_password("im_service".into(), "CHANGE_ME_STRONG_RW".into())
        
        // 客户端名称
        .name("rust-client")
        
        // 重连配置
        .max_reconnects(10)
        .reconnect_delay_callback(|attempts| {
            std::cmp::min(attempts * 100, 2000) // 最大2秒重连延迟
        })
        
        // 超时配置
        .connection_timeout(Duration::from_secs(10))
        .request_timeout(Some(Duration::from_secs(5)))
        
        // 连接多个服务器
        .connect(vec!["nats://localhost:4222"])
        .await?;
    
    println!("Connected with advanced options!");
    Ok(())
}
```

## 使用示例

### 1. 发布/订阅模式

```rust
use async_nats;
use futures::StreamExt;

#[tokio::main]
async fn main() -> Result<(), async_nats::Error> {
    let client = async_nats::ConnectOptions::new()
        .user_and_password("im_service".into(), "CHANGE_ME_STRONG_RW".into())
        .connect("nats://localhost:4222")
        .await?;

    // 订阅主题
    let mut subscriber = client.subscribe("messages.>").await?;
    
    // 异步处理消息
    tokio::spawn(async move {
        while let Some(msg) = subscriber.next().await {
            println!("Received: {}", String::from_utf8_lossy(&msg.payload));
            
            // 如果需要响应
            if let Some(reply) = msg.reply {
                let _ = msg.respond("ACK".into()).await;
            }
        }
    });

    // 发布消息
    client.publish("messages.test", "Hello NATS!".into()).await?;
    
    // 保持连接
    tokio::time::sleep(Duration::from_secs(60)).await;
    Ok(())
}
```

### 2. 请求/响应模式

```rust
use async_nats;
use std::time::Duration;

#[tokio::main]
async fn main() -> Result<(), async_nats::Error> {
    let client = async_nats::ConnectOptions::new()
        .user_and_password("im_service".into(), "CHANGE_ME_STRONG_RW".into())
        .connect("nats://localhost:4222")
        .await?;

    // 启动响应服务
    let service = client.clone();
    tokio::spawn(async move {
        let mut sub = service.subscribe("service.echo").await.unwrap();
        while let Some(msg) = sub.next().await {
            if let Some(reply) = msg.reply {
                let response = format!("Echo: {}", String::from_utf8_lossy(&msg.payload));
                service.publish(reply, response.into()).await.unwrap();
            }
        }
    });

    // 发送请求并等待响应
    let response = client
        .request("service.echo", "Hello Service".into())
        .await?;
    
    println!("Response: {}", String::from_utf8_lossy(&response.payload));
    Ok(())
}
```

### 3. JetStream 使用

```rust
use async_nats;
use async_nats::jetstream;

#[tokio::main]
async fn main() -> Result<(), async_nats::Error> {
    let client = async_nats::ConnectOptions::new()
        .user_and_password("im_service".into(), "CHANGE_ME_STRONG_RW".into())
        .connect("nats://localhost:4222")
        .await?;

    // 创建 JetStream 上下文
    let jetstream = jetstream::new(client);

    // 创建流
    let stream = jetstream
        .create_stream(jetstream::stream::Config {
            name: "EVENTS".to_string(),
            subjects: vec!["events.>".to_string()],
            max_messages: 10_000,
            max_bytes: 1024 * 1024 * 100, // 100MB
            ..Default::default()
        })
        .await?;

    // 发布消息到流
    let ack = jetstream
        .publish("events.user.login", "user123 logged in".into())
        .await?
        .await?;
    
    println!("Message published, sequence: {}", ack.sequence);

    // 创建消费者
    let consumer = stream
        .create_consumer(jetstream::consumer::pull::Config {
            name: Some("processor".to_string()),
            durable_name: Some("processor".to_string()),
            ..Default::default()
        })
        .await?;

    // 拉取消息
    let mut messages = consumer.fetch()
        .max_messages(10)
        .messages()
        .await?;

    while let Some(Ok(message)) = messages.next().await {
        println!("JetStream message: {:?}", message);
        message.ack().await?;
    }

    Ok(())
}
```

### 4. 使用 Serde 序列化

```rust
use async_nats;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Event {
    id: String,
    user_id: String,
    action: String,
    timestamp: i64,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = async_nats::ConnectOptions::new()
        .user_and_password("im_service".into(), "CHANGE_ME_STRONG_RW".into())
        .connect("nats://localhost:4222")
        .await?;

    // 创建事件
    let event = Event {
        id: "evt_123".to_string(),
        user_id: "user_456".to_string(),
        action: "login".to_string(),
        timestamp: chrono::Utc::now().timestamp(),
    };

    // 序列化并发送
    let json = serde_json::to_vec(&event)?;
    client.publish("events.user", json.into()).await?;

    // 订阅并反序列化
    let mut sub = client.subscribe("events.>").await?;
    while let Some(msg) = sub.next().await {
        if let Ok(event) = serde_json::from_slice::<Event>(&msg.payload) {
            println!("Received event: {:?}", event);
        }
    }

    Ok(())
}
```

## 错误处理

```rust
use async_nats;

#[tokio::main]
async fn main() {
    // 处理连接错误
    match connect_to_nats().await {
        Ok(client) => {
            println!("Connected successfully");
            // 使用 client
        }
        Err(e) => {
            eprintln!("Connection failed: {}", e);
            // 实现重连逻辑或退出
        }
    }
}

async fn connect_to_nats() -> Result<async_nats::Client, async_nats::Error> {
    async_nats::ConnectOptions::new()
        .user_and_password("im_service".into(), "CHANGE_ME_STRONG_RW".into())
        .connect("nats://localhost:4222")
        .await
}
```

## 监控和调试

### 1. 启用日志

```rust
use tracing_subscriber;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 初始化日志
    tracing_subscriber::fmt()
        .with_env_filter("async_nats=debug,nats_client=debug")
        .init();

    // ... NATS 连接代码
    Ok(())
}
```

### 2. 连接状态监听

```rust
use async_nats;
use futures::StreamExt;

#[tokio::main]
async fn main() -> Result<(), async_nats::Error> {
    let client = async_nats::ConnectOptions::new()
        .user_and_password("im_service".into(), "CHANGE_ME_STRONG_RW".into())
        .event_callback(|event| async move {
            println!("Connection event: {:?}", event);
        })
        .connect("nats://localhost:4222")
        .await?;

    // 监控连接统计
    let stats = client.stats();
    println!("Messages sent: {}", stats.out_msgs);
    println!("Messages received: {}", stats.in_msgs);
    println!("Bytes sent: {}", stats.out_bytes);
    println!("Bytes received: {}", stats.in_bytes);

    Ok(())
}
```

## 性能优化建议

1. **使用连接池**: 对于高并发场景，考虑使用多个连接
2. **批量发送**: 使用 `publish_with_reply` 批量发送消息
3. **消息缓冲**: 设置适当的缓冲区大小
4. **异步处理**: 充分利用 Tokio 的异步特性
5. **压缩**: 对于大消息，考虑使用压缩

## 故障排查

### 常见错误和解决方案

| 错误 | 原因 | 解决方案 |
|-----|------|---------|
| `Connection refused` | NATS 服务器未启动 | 检查 Docker 容器状态 |
| `Authorization Violation` | 用户名/密码错误 | 验证认证信息 |
| `Maximum Payload Exceeded` | 消息太大 | 减小消息大小或调整服务器配置 |
| `Slow Consumer` | 消费速度太慢 | 优化消息处理逻辑或增加消费者 |

### 测试连接

```bash
# 使用 NATS CLI 测试
nats -s nats://im_service:CHANGE_ME_STRONG_RW@localhost:4222 pub test "Hello"
nats -s nats://im_service:CHANGE_ME_STRONG_RW@localhost:4222 sub ">"

# 查看服务器信息
curl http://localhost:8222/varz
curl http://localhost:8222/connz
curl http://localhost:8222/routez
```

## 生产环境建议

1. **使用 TLS**: 生产环境应启用 TLS 加密
2. **更改默认密码**: 使用强密码并定期更换
3. **限制权限**: 为不同服务创建专用账户
4. **监控指标**: 通过 Prometheus/Grafana 监控
5. **日志审计**: 记录关键操作日志
6. **容错设计**: 实现断线重连和消息重试机制