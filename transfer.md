```mermaid
flowchart TD
    A[POST /assets/transfers<br/>X-Request-Id] --> B{实名认证?}
    B -->|否| B1[拒绝]
    B -->|是| C[校验交易密码]
    C --> D[RequestId → orderNo/bizNo]
    D --> E[TransferService.transferByAddress]

    E --> F[requireInternalOwner<br/>toAddress 必须是平台内部地址]
    F --> G[按 chainAddrId + symbol<br/>解析收款 toUserId]
    G --> H[组装 TransferCreateReqDTO<br/>USER → USER]
    H --> I[executeTransferFlow]

    I --> J{不能转给自己<br/>校验最小金额 NB≥10 / USDT≥1}
    J --> K{bizNo 订单已存在?}
    K -->|已 SUCCEED| K1[ORDER_ALREADY_PAID]
    K -->|其他状态| K2[ORDER_STATUS_NOT_CORRECT]
    K -->|不存在| L[计算用户转账手续费]
    L --> M[插入 transfer_order<br/>status=PENDING]
    M --> N[推送双方近期交易缓存]

    N --> O[BizFinanceService.userTransfer<br/>账户中心]
    O --> P[from USER_AVAILABLE → to USER_AVAILABLE<br/>fee → PLATFORM_FEE]

    P -->|成功| Q[订单 PENDING → SUCCEED]
    Q --> R[通知付款方转出成功<br/>通知收款方到账成功]
    R --> S[返回 AssetTxSubmitRespVO]

    P -->|BizErr| T[订单 → CANCELLED 并抛错]
    P -->|其他异常| U[保持 PENDING 抛错<br/>留给对账]

    subgraph reconcile [TransferReconcileJob]
        V[创建超 5 分钟仍 PENDING] --> W{账户中心<br/>userTransferExists?}
        W -->|否| X[订单 → CANCELLED]
        W -->|是| Y[补写 SUCCEED<br/>刷新缓存]
    end

    U -.-> V
