```mermaid
flowchart TD
    subgraph entry [卖家入口]
        A1[POST /orders/sell/quick-sell<br/>极速卖出] --> C
        A2[POST /orders/sell/pending<br/>挂单出售] --> C
        C{实名 + 交易密码<br/>X-Request-Id → orderNo}
    end

    C --> D[processSellTokenOrder]
    D --> E[校验收款方式 PaymentInfo]
    E --> F[按法币金额算汇率 → 代币数量]
    F --> G[算卖出手续费]
    G --> H[落 token_order<br/>初始 PENDING]
    H --> I[otcSellFreeze<br/>USER_AVAILABLE → USER_OTC<br/>fee → PLATFORM_OTC_FEE]
    I --> J[状态改为 OPEN]

    J --> K{类型?}
    K -->|SELL_TOKEN_QUICK| L1[极速卖单挂出<br/>5 分钟有效]
    K -->|SELL_TOKEN| L2[挂单卖出<br/>发挂单成功通知<br/>12 小时有效]

    subgraph match [撮合成交]
        M1[用户买币 GET /orders/seller/orders<br/>或商户内部买币] --> M2{撮合卖单<br/>极速单优先于挂单}
        M2 -->|命中| M3[扣减卖单 residual<br/>PARTIAL / FILLED]
        M3 --> M4[创建买单 UNPAID]
        M2 -->|未命中用户买| M5[走通道代收 + 系统卖单]
    end

    L1 --> M2
    L2 --> M2

    M4 --> N1[买家确认付款 → PAID]
    N1 --> N2[卖家确认放币<br/>或通道代收回调]
    N2 --> N3[资金交割 otcUserMatch 等<br/>卖家 USER_OTC → 买家]
    N3 --> N4[买单 SUCCEED<br/>卖单按 residual 更新]

    subgraph expire [过期兜底]
        E1[Expired5MinsSellerOrderJob<br/>极速单 OPEN/PARTIAL 超 5 分钟]
        E2[Expired12HoursSellerOrderJob<br/>挂单 OPEN/PARTIAL 超 12 小时]
        E1 --> E3[handleExpiredSellOrder]
        E2 --> E3
        E3 --> E4[系统买单吃剩余量<br/>通道代付打款给卖家]
        E4 --> E5[代付成功回调 → 确认放币]
        E4 -->|代付失败| E6[取消 + 手工补单]
    end

    L1 -.-> E1
    L2 -.-> E2
