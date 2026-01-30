# Rust Port Implementation Roadmap

**Purpose:** Detailed step-by-step implementation guide for the recommended hybrid approach

---

## Phase 0: Preparation & Planning (Weeks 1-4)

### Week 1: Performance Profiling

**Objective:** Identify actual bottlenecks with data

**Tasks:**
1. Set up performance monitoring
   - Add instrumentation to TypeScript codebase
   - Track: CPU usage, memory usage, response times, throughput
   - Tools: clinic.js, autocannon, 0x profiler

2. Run load tests
   ```bash
   # Example load test
   autocannon -c 100 -d 60 http://localhost:18789/api/message
   ```

3. Profile specific operations
   - Gateway request handling
   - Message routing
   - Media processing
   - Database queries
   - WebSocket handling

4. Document findings
   - Top 10 CPU hotspots
   - Top 10 memory allocations
   - Slowest operations
   - Resource utilization patterns

**Deliverable:** Performance baseline report with metrics

### Week 2: Rust Environment Setup

**Objective:** Prepare development environment and team

**Tasks:**
1. Install Rust toolchain
   ```bash
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   rustup component add clippy rustfmt rust-analyzer
   ```

2. Set up project structure
   ```
   openclaw-rs/
   ├── Cargo.toml
   ├── src/
   │   ├── main.rs
   │   ├── gateway/
   │   ├── router/
   │   └── lib.rs
   ├── tests/
   └── benches/
   ```

3. Choose key libraries
   - Web: `axum = "0.7"`
   - Async: `tokio = { version = "1", features = ["full"] }`
   - WebSocket: `axum-tungstenite = "0.1"`
   - HTTP client: `reqwest = "0.12"`
   - Serialization: `serde = { version = "1", features = ["derive"] }`
   - Database: `sqlx = { version = "0.7", features = ["sqlite"] }`
   - Logging: `tracing = "0.1"`

4. Team training
   - Rust book reading group
   - Hands-on exercises
   - Code review process

**Deliverable:** Working Rust environment, trained team

### Week 3: Architecture Design

**Objective:** Design Rust-TypeScript integration

**Tasks:**
1. Define interfaces
   - HTTP/WebSocket API contracts
   - Message formats (JSON)
   - Error handling strategy
   - Logging integration

2. Plan FFI boundaries
   - What stays in TypeScript?
   - What moves to Rust?
   - Communication protocol

3. Design deployment strategy
   - Single binary with embedded TypeScript runtime?
   - Separate processes communicating via HTTP/IPC?
   - Hybrid with FFI/N-API?

4. Create migration plan
   - Component prioritization
   - Rollback strategy
   - Testing approach
   - Deployment phases

**Deliverable:** Architecture design document

### Week 4: Prototype Development

**Objective:** Build minimal working prototype

**Tasks:**
1. Create "Hello World" Rust gateway
   ```rust
   // src/main.rs
   use axum::{routing::get, Router};
   
   #[tokio::main]
   async fn main() {
       let app = Router::new()
           .route("/health", get(|| async { "OK" }));
       
       let listener = tokio::net::TcpListener::bind("0.0.0.0:18789")
           .await
           .unwrap();
       
       axum::serve(listener, app).await.unwrap();
   }
   ```

2. Add WebSocket support
3. Implement basic message routing
4. Test TypeScript client connectivity
5. Measure baseline performance

**Deliverable:** Working prototype, performance comparison

---

## Phase 1: Gateway Server Core (Months 2-4)

### Month 2: HTTP/WebSocket Server

**Objective:** Port core gateway server to Rust

**Milestones:**
- Week 1: HTTP endpoints
- Week 2: WebSocket handling
- Week 3: Request routing
- Week 4: Integration testing

**Implementation:**

1. **HTTP Endpoints** (Week 1)
   ```rust
   // src/gateway/http.rs
   use axum::{
       extract::State,
       http::StatusCode,
       response::IntoResponse,
       routing::{get, post},
       Json, Router,
   };
   use serde::{Deserialize, Serialize};
   use std::sync::Arc;
   
   #[derive(Clone)]
   pub struct GatewayState {
       router: Arc<MessageRouter>,
   }
   
   #[derive(Deserialize)]
   pub struct MessageRequest {
       channel: String,
       target: String,
       content: String,
   }
   
   #[derive(Serialize)]
   pub struct MessageResponse {
       success: bool,
       message_id: Option<String>,
       error: Option<String>,
   }
   
   async fn send_message(
       State(state): State<GatewayState>,
       Json(req): Json<MessageRequest>,
   ) -> impl IntoResponse {
       match state.router.route_message(req).await {
           Ok(id) => Json(MessageResponse {
               success: true,
               message_id: Some(id),
               error: None,
           }),
           Err(e) => (
               StatusCode::INTERNAL_SERVER_ERROR,
               Json(MessageResponse {
                   success: false,
                   message_id: None,
                   error: Some(e.to_string()),
               }),
           ),
       }
   }
   
   pub fn create_router(state: GatewayState) -> Router {
       Router::new()
           .route("/health", get(health_check))
           .route("/api/message", post(send_message))
           .with_state(state)
   }
   ```

2. **WebSocket Handler** (Week 2)
   ```rust
   // src/gateway/websocket.rs
   use axum::{
       extract::{ws::WebSocket, State, WebSocketUpgrade},
       response::IntoResponse,
   };
   use futures::{sink::SinkExt, stream::StreamExt};
   
   pub async fn websocket_handler(
       ws: WebSocketUpgrade,
       State(state): State<GatewayState>,
   ) -> impl IntoResponse {
       ws.on_upgrade(|socket| handle_socket(socket, state))
   }
   
   async fn handle_socket(socket: WebSocket, state: GatewayState) {
       let (mut sender, mut receiver) = socket.split();
       
       while let Some(msg) = receiver.next().await {
           let msg = match msg {
               Ok(msg) => msg,
               Err(e) => {
                   tracing::error!("WebSocket error: {}", e);
                   break;
               }
           };
           
           if let axum::extract::ws::Message::Text(text) = msg {
               let response = handle_message(&state, text).await;
               if sender.send(axum::extract::ws::Message::Text(response))
                   .await
                   .is_err()
               {
                   break;
               }
           }
       }
   }
   ```

3. **Message Router** (Week 3)
   ```rust
   // src/router/mod.rs
   use std::collections::HashMap;
   use std::sync::Arc;
   use tokio::sync::RwLock;
   use anyhow::Result;
   
   pub struct MessageRouter {
       routes: Arc<RwLock<HashMap<String, Route>>>,
       typescript_bridge: TypeScriptBridge,
   }
   
   impl MessageRouter {
       pub async fn route_message(&self, req: MessageRequest) -> Result<String> {
           // Check if we handle this channel in Rust
           if self.is_rust_channel(&req.channel) {
               self.handle_rust_channel(req).await
           } else {
               // Delegate to TypeScript
               self.typescript_bridge.send_message(req).await
           }
       }
   }
   ```

4. **TypeScript Bridge** (Week 3)
   ```rust
   // src/bridge/typescript.rs
   use reqwest::Client;
   use anyhow::Result;
   
   pub struct TypeScriptBridge {
       client: Client,
       ts_gateway_url: String,
   }
   
   impl TypeScriptBridge {
       pub async fn send_message(&self, req: MessageRequest) -> Result<String> {
           let response = self.client
               .post(&format!("{}/api/legacy/message", self.ts_gateway_url))
               .json(&req)
               .send()
               .await?;
           
           let result: MessageResponse = response.json().await?;
           Ok(result.message_id.unwrap_or_default())
       }
   }
   ```

5. **Integration Tests** (Week 4)
   ```rust
   // tests/gateway_integration.rs
   #[tokio::test]
   async fn test_health_check() {
       let response = reqwest::get("http://localhost:18789/health")
           .await
           .unwrap();
       assert_eq!(response.status(), 200);
   }
   
   #[tokio::test]
   async fn test_send_message() {
       let client = reqwest::Client::new();
       let req = MessageRequest {
           channel: "telegram".to_string(),
           target: "@testuser".to_string(),
           content: "Hello!".to_string(),
       };
       
       let response = client
           .post("http://localhost:18789/api/message")
           .json(&req)
           .send()
           .await
           .unwrap();
       
       assert_eq!(response.status(), 200);
   }
   ```

**Success Metrics:**
- [ ] All HTTP endpoints functional
- [ ] WebSocket connections stable
- [ ] 99.9% compatibility with TypeScript clients
- [ ] Response time < 10ms (p99)
- [ ] Memory usage < 50MB under load

### Month 3: Message Routing & State

**Objective:** Implement routing logic and state management

**Tasks:**
1. Session management
2. Route table management
3. Connection pooling
4. Health checks
5. Metrics collection

**Implementation:**

```rust
// src/router/session.rs
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::RwLock;
use chrono::{DateTime, Utc};

#[derive(Clone)]
pub struct Session {
    pub id: String,
    pub channel: String,
    pub target: String,
    pub last_active: DateTime<Utc>,
    pub metadata: HashMap<String, String>,
}

pub struct SessionManager {
    sessions: Arc<RwLock<HashMap<String, Session>>>,
}

impl SessionManager {
    pub fn new() -> Self {
        Self {
            sessions: Arc::new(RwLock::new(HashMap::new())),
        }
    }
    
    pub async fn get(&self, id: &str) -> Option<Session> {
        self.sessions.read().await.get(id).cloned()
    }
    
    pub async fn save(&self, session: Session) {
        self.sessions.write().await.insert(session.id.clone(), session);
    }
    
    pub async fn cleanup_stale(&self, max_age_seconds: i64) {
        let now = Utc::now();
        let mut sessions = self.sessions.write().await;
        sessions.retain(|_, s| {
            (now - s.last_active).num_seconds() < max_age_seconds
        });
    }
}
```

**Success Metrics:**
- [ ] Session persistence working
- [ ] Route lookups < 1ms
- [ ] Automatic cleanup working
- [ ] No memory leaks after 24h

### Month 4: Production Hardening

**Objective:** Make production-ready

**Tasks:**
1. Error handling
2. Logging & tracing
3. Configuration management
4. Graceful shutdown
5. Monitoring & metrics

**Implementation:**

```rust
// src/observability/mod.rs
use tracing::{info, warn, error, instrument};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

pub fn init_tracing() {
    tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::new(
            std::env::var("RUST_LOG").unwrap_or_else(|_| "info".into()),
        ))
        .with(tracing_subscriber::fmt::layer())
        .init();
}

#[instrument(skip(state))]
pub async fn handle_request(state: &GatewayState, req: MessageRequest) -> Result<String> {
    info!("Handling message to {}", req.target);
    
    match state.router.route_message(req).await {
        Ok(id) => {
            info!("Message sent successfully: {}", id);
            Ok(id)
        }
        Err(e) => {
            error!("Failed to send message: {}", e);
            Err(e)
        }
    }
}
```

**Success Metrics:**
- [ ] Structured logging working
- [ ] Metrics exported (Prometheus)
- [ ] Graceful shutdown < 5s
- [ ] Config hot-reload working

---

## Phase 2: Media Processing (Month 5)

**Objective:** Port image/video processing to Rust

**Tasks:**
1. Image resizing/optimization
2. Format conversion
3. Metadata extraction
4. FFI bridge to TypeScript

**Implementation:**

```rust
// src/media/image.rs
use image::{ImageFormat, DynamicImage, imageops::FilterType};
use anyhow::Result;

pub struct ImageProcessor;

impl ImageProcessor {
    pub async fn process(&self, input: &[u8], opts: ProcessingOptions) -> Result<Vec<u8>> {
        // Run CPU-intensive work in thread pool
        tokio::task::spawn_blocking(move || {
            let img = image::load_from_memory(input)?;
            
            let img = if img.width() > opts.max_width || img.height() > opts.max_height {
                img.resize(opts.max_width, opts.max_height, FilterType::Lanczos3)
            } else {
                img
            };
            
            let mut output = Vec::new();
            img.write_to(&mut std::io::Cursor::new(&mut output), opts.format)?;
            
            Ok(output)
        }).await?
    }
}
```

**FFI Bridge (N-API):**

```rust
// src/ffi/mod.rs
use napi::bindgen_prelude::*;
use napi_derive::napi;

#[napi]
pub struct ImageProcessorFFI {
    processor: ImageProcessor,
}

#[napi]
impl ImageProcessorFFI {
    #[napi(constructor)]
    pub fn new() -> Self {
        Self {
            processor: ImageProcessor::new(),
        }
    }
    
    #[napi]
    pub async fn process(&self, input: Buffer, opts: ProcessingOptions) -> Result<Buffer> {
        let output = self.processor.process(&input, opts).await?;
        Ok(Buffer::from(output))
    }
}
```

**TypeScript Usage:**

```typescript
import { ImageProcessorFFI } from './native';

const processor = new ImageProcessorFFI();
const output = await processor.process(inputBuffer, {
  maxWidth: 1920,
  maxHeight: 1080,
  quality: 85,
  format: 'jpeg',
});
```

**Success Metrics:**
- [ ] 3x faster than Sharp
- [ ] Same output quality
- [ ] FFI overhead < 1ms
- [ ] No memory leaks

---

## Phase 3: Vector Search (Month 6)

**Objective:** Implement high-performance memory search

**Implementation:**

```rust
// src/memory/search.rs
use tantivy::*;
use anyhow::Result;

pub struct VectorSearch {
    index: Index,
}

impl VectorSearch {
    pub async fn search(&self, query: &str, limit: usize) -> Result<Vec<SearchResult>> {
        let searcher = self.index.reader()?.searcher();
        let query_parser = QueryParser::for_index(&self.index, vec![]);
        let query = query_parser.parse_query(query)?;
        
        let top_docs = searcher.search(&query, &TopDocs::with_limit(limit))?;
        
        let results = top_docs
            .into_iter()
            .map(|(score, doc_address)| {
                let doc = searcher.doc(doc_address)?;
                Ok(SearchResult {
                    score,
                    content: extract_content(&doc),
                })
            })
            .collect::<Result<Vec<_>>>()?;
        
        Ok(results)
    }
}
```

**Success Metrics:**
- [ ] 10x faster than SQLite FTS
- [ ] Handles 1M+ documents
- [ ] Query time < 10ms (p95)

---

## Phase 4: Deployment (Month 7)

**Objective:** Deploy to production alongside TypeScript

**Strategy: Blue-Green Deployment**

1. **Week 1: Build & Package**
   ```bash
   # Build release binary
   cargo build --release
   
   # Create Dockerfile
   FROM debian:bookworm-slim
   COPY target/release/openclaw-gateway /usr/local/bin/
   EXPOSE 18789
   CMD ["openclaw-gateway"]
   ```

2. **Week 2: Staged Rollout**
   - 5% traffic to Rust gateway
   - Monitor metrics
   - Compare with TypeScript baseline
   
3. **Week 3: Ramp Up**
   - 25% traffic
   - 50% traffic
   - 100% traffic
   
4. **Week 4: Validate**
   - Performance metrics
   - Error rates
   - User feedback

**Rollback Plan:**
- Keep TypeScript gateway running
- Feature flag to switch back
- Maximum rollback time: 5 minutes

---

## Phase 5: Monitoring & Optimization (Month 8-9)

**Objective:** Measure, optimize, iterate

**Tasks:**
1. Set up dashboards (Grafana)
2. Alert on regressions
3. Profile performance
4. Optimize hot paths
5. Document learnings

**Metrics to Track:**
- Request latency (p50, p95, p99)
- Throughput (req/s)
- Memory usage (RSS, heap)
- CPU utilization
- Error rate
- WebSocket connections

**Optimization Targets:**
- Reduce p99 latency by 50%
- Reduce memory by 60%
- Increase throughput by 3x

---

## Success Criteria

### Phase 1 (Gateway) Success Metrics
- ✅ All existing API endpoints functional
- ✅ WebSocket stability (no disconnects)
- ✅ 40-60% memory reduction
- ✅ 2x throughput improvement
- ✅ Response time < 10ms (p99)
- ✅ Zero data loss
- ✅ TypeScript clients unmodified

### Phase 2 (Media) Success Metrics
- ✅ 3x faster image processing
- ✅ Same output quality
- ✅ FFI overhead < 1ms
- ✅ All formats supported

### Phase 3 (Search) Success Metrics
- ✅ 10x faster search
- ✅ Same relevance quality
- ✅ Handles 1M+ docs
- ✅ Query time < 10ms (p95)

### Overall Project Success
- ✅ Production stable for 30 days
- ✅ No increase in error rate
- ✅ Performance targets met
- ✅ Team comfortable with Rust
- ✅ Documentation complete
- ✅ Monitoring dashboards live

---

## Risk Mitigation

### Technical Risks

**Risk:** Performance not as expected  
**Mitigation:** Benchmark early, have fallback plan

**Risk:** TypeScript integration breaks  
**Mitigation:** Comprehensive integration tests, feature flags

**Risk:** Memory leaks in Rust  
**Mitigation:** Valgrind, ASAN, continuous monitoring

**Risk:** WebSocket instability  
**Mitigation:** Load testing, chaos engineering

### Team Risks

**Risk:** Rust learning curve too steep  
**Mitigation:** Pair programming, code reviews, training

**Risk:** Velocity drops  
**Mitigation:** Limit Rust scope, keep TypeScript option

### Business Risks

**Risk:** Delayed feature delivery  
**Mitigation:** Parallel teams, prioritize features

**Risk:** User-facing regressions  
**Mitigation:** Staged rollout, comprehensive testing

---

## Budget

### Engineering Time
- Phase 0 (Prep): 160 hours
- Phase 1 (Gateway): 480 hours
- Phase 2 (Media): 160 hours
- Phase 3 (Search): 160 hours
- Phase 4 (Deploy): 160 hours
- Phase 5 (Optimize): 160 hours
- **Total: 1,280 hours (7.5 months at full-time)**

### Team
- 2 Senior Rust Engineers @ $150k/year
- 1 DevOps Engineer @ $130k/year (20% time)

### Infrastructure
- Staging environment: $200/month
- Production monitoring: $100/month
- CI/CD: $50/month

### Training
- Rust training: $5,000
- Books & resources: $1,000

**Total Budget: ~$250k-$300k**

---

## Decision Checkpoints

### Checkpoint 1: After Prototype (Week 4)
**Question:** Are performance gains real?  
**Go:** Gains > 50%, continue  
**No-Go:** Gains < 30%, optimize TypeScript instead

### Checkpoint 2: After Gateway (Month 4)
**Question:** Is production stable?  
**Go:** Stable for 7 days, continue  
**No-Go:** Frequent issues, pause and fix

### Checkpoint 3: After Deployment (Month 7)
**Question:** Meeting all success criteria?  
**Go:** Expand Rust usage  
**No-Go:** Maintain hybrid, don't expand

---

## Conclusion

This roadmap provides a **pragmatic, incremental approach** to porting OpenClaw's performance-critical components to Rust while minimizing risk.

**Key Principles:**
1. **Start small** - Gateway first
2. **Measure everything** - Data-driven decisions
3. **Keep fallbacks** - TypeScript always available
4. **Validate early** - Frequent checkpoints
5. **Team first** - Training and support

**Expected Outcome:**
- 40-60% memory reduction
- 2-5x performance improvement
- Production-stable hybrid system
- Team experienced in Rust
- Clear path for future migrations

---

**See Also:**
- [Feasibility Assessment](./RUST_FEASIBILITY_ASSESSMENT.md)
- [Executive Summary](./RUST_PORT_EXECUTIVE_SUMMARY.md)
- [Technical Comparison](./RUST_TYPESCRIPT_COMPARISON.md)

**Last Updated:** 2026-01-30
