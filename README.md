
flowchart TD
    %% ---------- 前端 ----------
    FE[Front‑end (Home / Detail)]

    %% ---------- API Gateway ----------
    GW[API Gateway\n(Auth + Rate‑limit + Trace‑ID)]

    %% ---------- 业务服务 ----------
    FloatS[FloatService\nrecord exposure & anti‑spam]
    DialogS[DialogService\ncall LLM + intent routing]
    RecS[RecommendService\nsimilar search & price compare]
    LifeS[LifeService\nlocal travel & hotel]
    CartS[CartService\nadd to cart]

    %% ---------- 数据库/缓存 ----------
    Redis[(Redis Cache)]
    MySQL[(MySQL – product / user / logs)]
    ES[(Elasticsearch – vector index)]
    Mongo[(MongoDB – scenery / hotel)]
    Kafka[(Kafka – event stream)]
    ClickHouse[(ClickHouse – behavior logs)]

    %% ---------- 外部系统 ----------
    LLM[(LLM Container\n/api/v1/chat/completions)]
    MapAPI[(Map API\nGaode / Tencent)]

    %% ======================================================
    %% 1️⃣ 曝光（首页 / 商品详情）
    FE -->|page load| GW
    GW -->|POST /api/float/expose| FloatS
    FloatS -->|INSERT float_expose_log| MySQL
    FloatS -->|SET Redis key float:expose:UID| Redis
    FloatS -->|PUBLISH event_float| Kafka
    Kafka -->|consume>>| ClickHouse

    %% 2️⃣ 打开对话框（主动或自动）
    GW -->|POST /api/dialog (user text)| DialogS
    DialogS -->|GET session from Redis| Redis
    DialogS -->|POST /v1/chat/completions| LLM
    LLM -->|return JSON (reply, intent, meta, cards?)| DialogS
    DialogS -->|INSERT dialog_log| MySQL
    DialogS -->|PUBLISH event_dialog| Kafka
    DialogS -->|UPDATE session (last_intent, last_sku)| Redis

    %% 3️⃣ 根据 LLM Intent 调用业务
    DialogS -->|intent = shopping| RecS
    DialogS -->|intent = life| LifeS
    DialogS -->|intent = smalltalk| NoBiz[Only text reply]

    %% 4️⃣ 商品相似推荐（shopping）
    RecS -->|GET cache recommend:similar:SKU| Redis
    RecS -->|MISS → KNN search on ES| ES
    ES -->|return similar SKU list| RecS
    RecS -->|SELECT product details from MySQL| MySQL
    RecS -->|BUILD ProductCard list| DialogS
    RecS -->|SET cache (TTL 5 min)| Redis
    RecS -->|PUBLISH event_recommend(similar)| Kafka

    %% 5️⃣ 商品比价（compare action）
    RecS -->|POST /api/compare/price (skuA, skuB)| RecS
    RecS -->|SELECT two products from MySQL| MySQL
    RecS -->|CALCULATE diff & cheaperSku| DialogS
    RecS -->|SET cache (TTL 3 min)| Redis
    RecS -->|PUBLISH event_recommend(price)| Kafka

    %% 6️⃣ 本地生活攻略（life intent）
    LifeS -->|GET cache life:route| Redis
    LifeS -->|MISS → CALL Map API| MapAPI
    LifeS -->|GET scenery from MongoDB| Mongo
    LifeS -->|GET hotels from MongoDB| Mongo
    LifeS -->|BUILD LifeRouteResult| DialogS
    LifeS -->|SET cache (TTL 10 min)| Redis
    LifeS -->|PUBLISH event_life_route| Kafka

    %% 7️⃣ 加入购物车（任何页面的卡片按钮）
    GW -->|POST /api/cart/add (sku, qty)| CartS
    CartS -->|CHECK stock (optimistic lock) in MySQL| MySQL
    CartS -->|INSERT cart_item| MySQL
    CartS -->|PUBLISH event_cart_add| Kafka

    %% ==================== 事件流统一入口 ====================
    Kafka -->|real‑time consumer| ClickHouse
    ClickHouse -->|BI / offline feature calc| BI[Dashboard / Model Training]
