```mermaid 
flowchart TD
    A[GET /orders/seller/orders<br/>getAvailableSellOrders] --> B{实名认证?}
    B -->|否| B1[拒绝]
    B -->|是| C[X-Request-Id → orderNo]
    C --> D[getOrderForBuyer]

    D --> E[校验当日取消买单上限]
    E --> F[校验收款方式 PaymentInfo]
    F --> G{已有未完成买单<br/>UNPAID/PAID?}
    G -->|是| G1[直接返回已有单<br/>existingOrder=true]
    G -->|否| H[按法币算买汇率 → 代币数量<br/>算买家手续费]

    H --> I{撮合用户卖单?<br/>极速单优先 / 挂单次之}
    I -->|命中| J[扣减卖单 residual<br/>PARTIAL 或 FILLED]
    J --> K[创建买单 UNPAID<br/>refId=卖单<br/>带回卖家收款信息]
    K --> L[买家线下付款]
    L --> M[POST .../confirm-payment<br/>买单 → PAID]
    M --> N[通知卖家买家已付款]
    N --> O[卖家 POST /orders/seller/confirm<br/>确认放币]
    O --> P[otcUserMatch<br/>卖家 USER_OTC → 买家<br/>RESTRICTED_AVAILABLE]
    P --> Q[买单 SUCCEED<br/>卖单按 residual 更新<br/>写 order_trade]

    I -->|未命中| R[requestChannelForCollect<br/>通道代收]
    R --> S[保存 TokenOrderChannelRequest<br/>UNPAID + channelUrl]
    S --> T[创建系统卖单<br/>userId=SYSTEM]
    T --> U[otcPlatformSellFreeze<br/>PLATFORM_OTC_OUT → FROZEN]
    U --> V[创建用户买单 UNPAID<br/>挂 channelUrl]
    V --> W[买家打开通道链接付款]

    W --> X[通道代收回调<br/>channelCollectCallback]
    X --> Y{status?}
    Y -->|CANCELLED / 非 SUCCEED| Y1[跳过不处理]
    Y -->|SUCCEED| Z[按 channelRequestNo<br/>查代收记录 → 买单]
    Z --> AA[代收记录标 SUCCEED]
    AA --> AB[doConfirmBuyerOrder]
    AB --> AC[otcPlatformUserMatch<br/>PLATFORM_OTC_FROZEN → 买家]
    AC --> AD[买单 SUCCEED<br/>系统卖单收口<br/>可发首充奖励]

    subgraph cancel [取消]
        C1[买家 cancel] --> C2[买单 CANCELLED<br/>恢复卖单 residual]
        C2 --> C3{卖家是系统?}
        C3 -->|是| C4[系统卖单取消 + 解冻]
        C3 -->|否且卖单已超期| C5[系统买单 + 通道代付兜底]
    end

    K -.-> C1
    V -.-> C1
