```mermaid
flowchart TD
    A[商户服务 INTERNAL_SERVICE<br/>getMerchantAvailableSellOrders] --> B[getOrderForMerchant]
    B --> C[校验法币 FiatCurrency]
    C --> D{同 orderNo 买单已存在?}
    D -->|是| D1[幂等返回已有买单]
    D -->|否| E[selectMerchantMatchingSellerOrder]

    E --> F{撮合条件}
    F --> F1[优先极速卖单 type=5<br/>创建 5 分钟内 / OPEN]
    F --> F2[其次挂单 type=4<br/>创建 48 小时内 / OPEN]
    F --> F3[金额完全一致<br/>amount/fiat 原始=剩余<br/>且 amount ≤ 500]
    F --> F4[币种 + 支付方式一致]

    E -->|未命中| G[返回 null<br/>Assembler: success=false]
    E -->|命中| H[组装商户买单请求<br/>ownerType=MERCHANT<br/>userId=merchantNo]

    H --> I[processDealOrder]
    I --> J[CAS 扣减卖单 residual<br/>PARTIAL / FILLED]
    J --> K[创建买单 UNPAID<br/>refId=卖单]
    K --> L[confirmPaid<br/>UNPAID → PAID<br/>写 paidAt]
    L --> M[MerchantBuyerOrderAssembler]
    M --> N[加密买卖双方信息<br/>生成 cashierUrl]
    N --> O[返回 success=true<br/>+ 卖单 orderNo + cashierUrl]

    O --> P[打开 /merchants/cashier?d=...<br/>展示卖家收款信息]
    P --> Q[商户侧按卖家账户付款]
    Q --> R[卖家确认放币<br/>POST /orders/seller/confirm]
    R --> S{买单已 SUCCEED?}
    S -->|是| S1[幂等跳过]
    S -->|否| T[doConfirmBuyerOrder]
    T --> U[otcMerchantMatch<br/>卖家 USER_OTC → MERCHANT_COLLECT<br/>fee → PLATFORM_OTC_FEE]
    U --> V[买单 SUCCEED<br/>卖单状态收口<br/>写 order_trade]

