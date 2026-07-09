```mermaid
flowchart TD
    A[Expired5MinsSellerOrderJob] --> B[扫描条件]
    B --> B1[type = SELL_TOKEN_QUICK]
    B --> B2[status = OPEN / PARTIAL]
    B --> B3[createdAt 早于 now-5min]

    A --> C[逐单 getById 取最新]
    C --> D[handleExpiredSellOrder]

    D --> E{residualAmount / residualFiat<br/>都大于 0?}
    E -->|否| E1[跳过：无剩余量]
    E -->|是| F[CAS 扣减卖单剩余量<br/>updateSellerResidual]
    F -->|失败/已被占用| F1[跳过：防并发重复代付]
    F -->|成功| G[创建系统买单 SYS_<br/>userId=SYSTEM<br/>amount=剩余量<br/>status=UNPAID]

    G --> H[merchantChannelClient.payout<br/>打款到卖家收款账户]
    H --> I{通道返回 PAYING?}
    I -->|否/异常| J[回滚卖单 residual<br/>rollbackSellerResidualClaim]
    I -->|是| K[系统买单写入 channelName]

    K --> L{虚拟用户环境?}
    L -->|是| M[立刻 doConfirmBuyerOrder]
    L -->|否| N[等待通道代付回调<br/>channelPayoutCallback]

    N --> O{回调 status?}
    O -->|SUCCEED| P[doConfirmBuyerOrder]
    O -->|CANCELLED| Q[取消系统买单<br/>创建手工补单<br/>卖单可能标 SUCCEED]
    O -->|其他| R[跳过]

    P --> S[otcPlatformBuyCollect<br/>卖家 USER_OTC → PLATFORM_OTC_IN]
    S --> T[系统买单 SUCCEED<br/>卖单按 residual/关联买单收口]

    subgraph reconcile [兜底 ChannelOrderReconcileJob]
        U[扫描系统买单仍 UNPAID] --> V[查通道订单状态]
        V -->|无记录/CANCELLED| W[cancelOrder]
        V -->|SUCCEED| X[confirmBuyerOrder]
    end

    N -.-> U
