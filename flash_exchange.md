```mermaid
flowchart TD
    A[POST /orders/flash-exchange<br/>X-Request-Id] --> B{实名认证?}
    B -->|否| B1[拒绝]
    B -->|是| C[校验交易密码]
    C --> D[RequestId → orderNo]
    D --> E[FlashExchangeService.flashExchangeOrder]

    E --> F[算闪兑手续费<br/>resolveOutgoingFee]
    F --> G[净额 = amount - fee]

    G --> H{Redis 取 rateId 报价<br/>约 10 秒有效?}
    H -->|不存在/过期| H1[RATE_EXPIRE]
    H -->|存在| I{前端 exchangeRate<br/>与 Redis 一致?<br/>容差 0.0001}
    I -->|否| I1[RATE_CHANGE]
    I -->|是| J{源币是 NB?}
    J -->|是| J1[汇率取倒数]
    J -->|否| K
    J1 --> K[按净额重算 targetAmount]

    K --> L[插入 flash_exchange_order<br/>status=PENDING]
    L --> M[BizFinanceService.swap]
    M --> N[账户中心资金流]
    N --> N1[fee: USER_AVAILABLE → PLATFORM_SWAP_FEE]
    N --> N2[净额: USER_AVAILABLE → PLATFORM_TREASURY_IN]
    N --> N3[目标币: PLATFORM_TREASURY_OUT → USER_AVAILABLE]

    N -->|成功| O[订单 → SUCCEED]
    O --> P[推送近期交易缓存]
    P --> Q[通知闪兑成功]
    Q --> R[返回 orderNo]

    subgraph reconcile [FlashExchangeReconcileJob]
        S[创建超 5 分钟仍 PENDING] --> T{账户中心 swapExists?}
        T -->|否| U[订单 → CANCELLED]
        T -->|是| V[补写 SUCCEED + 刷新缓存]
    end

    L -.-> S
