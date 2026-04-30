# API Gateway Interview - TODO

40 分鐘面試題:在 `gateway/` 目錄實作一個 API Gateway,轉發請求到三個後端服務。

## 後端服務(已提供,不可改)

| Service | Port | Path Prefix |
|---------|------|-------------|
| user-service | 8081 | `/api/users/**` |
| order-service | 8082 | `/api/orders/**` |
| product-service | 8083 | `/api/products/**` |

啟動:`docker compose up --build`,每個服務都有 `/` 跟 `/health`。

---

## 必做 (Minimum Requirement)

- [ ] Gateway 監聽 `:8080`
- [ ] Path-based routing 到對應後端
  - `/api/users/**` → `localhost:8081`
  - `/api/orders/**` → `localhost:8082`
  - `/api/products/**` → `localhost:8083`
- [ ] 保留原始 path、method、header、body
- [ ] Gateway 加進 `docker-compose.yml`(或至少能本地跑起來)
- [ ] 用 curl 驗證三條路線都通

## Bonus(依 CP 值排序,40 分鐘內挑著做)

### 高 CP 值
- [x] env-driven 路由(`.env` + `godotenv`,k8s ConfigMap 友善)
- [x] `/health` endpoint
- [x] 結構化 log + Request ID(X-Request-Id,方便 trace)
- [x] Timeout 與錯誤處理(後端掛掉回 502,不是 panic)
- [x] Graceful shutdown(收到 SIGTERM 10s drain)

### 中等
- [ ] Rate limiting(token bucket,per-IP 或 global)
- [ ] Retry(只對冪等方法 GET/HEAD)
- [ ] CORS

### 進階(時間夠才做,否則口頭聊)
- [ ] Circuit breaker
- [ ] Auth middleware(API key / JWT)
- [ ] Metrics endpoint(Prometheus `/metrics`)
- [ ] Load balancing(目前每服務只有 1 instance,意義不大)

---

## 限制 / 注意

- **不可修改** `services/` 底下任何檔案
- 所有程式碼寫在 `gateway/` 目錄
- 最後以 PR 形式繳交
- 面試官會看螢幕,重點在「跟 AI 怎麼協作」,不只結果

## 已決定

- **技術棧**:Go,只用 stdlib(`net/http` + `net/http/httputil.ReverseProxy`)+ `godotenv`(本地 dev 載 `.env`)
- **跑法**:本地 `go run ./cmd`,連 host 上的 `localhost:8081/8082/8083`(三個 docker service 的 host port)
- **架構**:layered pattern,`/cmd` + `/internal`,參考 `go-layered-server`
- **設定**:env-driven(k8s ConfigMap/Secret 友善),本地 dev 用 `.env`(gitignored),schema 看 `.env.example`

---

## 設計架構

### 目錄結構

```
gateway/
├── cmd/
│   └── main.go                    # 入口:wire 依賴 + 啟 server
├── go.mod
└── internal/
    ├── config/
    │   └── config.go              # PORT + 路由表(MVP hardcode,bonus 改讀檔)
    ├── repository/                # ← microservice 業務隔離(infrastructure)
    │   ├── interfaces.go          # Backend interface
    │   └── http/
    │       ├── backend.go         # 共用 *httputil.ReverseProxy 實作
    │       ├── user_service.go    # NewUserServiceClient
    │       ├── order_service.go   # NewOrderServiceClient
    │       └── product_service.go # NewProductServiceClient
    ├── service/                   # ← gateway 業務層
    │   ├── type.go
    │   └── routing_service.go     # path-prefix → Backend lookup, Forward
    ├── handler/
    │   ├── type.go
    │   ├── health.go              # /health
    │   └── proxy.go               # catchall → service.Routing.Forward
    ├── middleware/
    │   ├── type.go
    │   └── logger.go              # access log + X-Request-Id
    ├── router/
    │   └── router.go              # 組 mux,掛 middleware + handler
    └── factory/
        ├── repository.go          # build 三個 backend
        ├── service.go
        ├── handler.go
        └── middleware.go
```

### 呼叫鏈

```
HTTP request
  → router (mux)
  → middleware.Logger
  → handler.Proxy (catchall)
  → service.Routing.Forward(w, r)                       ← 業務:path → backend
  → repository/http.UserServiceClient.ServeHTTP(w, r)   ← infra:reverse proxy
  → backend service
```

### 設計重點(可講的 trade-off)

1. **每個 microservice 一個檔案,但共用 `HTTPBackend` impl**
   - 業務隔離:per-service 客製(timeout / retry / auth header)只動自己的檔
   - 不重複代碼:底層 reverse proxy wrapper 共用
2. **service 層在 pass-through 看起來薄,但是 bonus feature 的歸宿**(rate limit / auth / transform)
3. **`Backend` interface 而非 concrete type**:可換 `MockBackend`(test)、`GRPCBackend`(未來)
4. **不用 framework**:gateway 是 pure HTTP plumbing,stdlib 就夠;少依賴、啟動快
