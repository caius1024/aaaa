```mermaid
flowchart TD
    A[用户请求充币地址<br/>GET /assets/recharge-address] --> B[UserAssetService 查 UserChainAddr]
    B --> C[blockchainClient 返回链上地址]
    C --> D[用户向该地址链上转账]

    D --> E[链上监听/区块链服务<br/>INTERNAL_SERVICE]
    E --> F[POST /assets/recharges/init<br/>RechargeReqDTO]

    F --> G[按 chainAddrId + symbol<br/>反查 userId]
    G --> H{recharge_order<br/>bizNo=txId 已存在?}
    H -->|是| I[幂等返回]
    H -->|否| J[插入 recharge_order<br/>status=PENDING]
    J --> K[推送近期交易缓存]
    K --> L[等待链上确认]

    L --> M[POST /assets/recharges/confirm<br/>批量 RechargeReqDTO]
    M --> N[逐笔: 反查 userId]
    N --> O[getOrCreateRechargeOrder<br/>无单则补建 PENDING]
    O --> P{status 已是 SUCCEED?}
    P -->|是| Q[幂等跳过]
    P -->|否| R[BizFinanceService.depositSuccess]
    R --> S[账户中心 DepositClient<br/>PLATFORM_DEPOSIT_CLEARING → USER_AVAILABLE]
    S --> T[订单 PENDING → SUCCEED<br/>写 confirmedAt]
    T --> U[刷新近期交易缓存]
    N -->|异常| V[txId 加入失败列表返回]

    subgraph reconcile [对账兜底 RechargeReconcileJob]
        W[扫描创建超过 5 分钟仍 PENDING 的单] --> X{账户中心<br/>depositSuccessExists?}
        X -->|否| Y[跳过]
        X -->|是| Z[补写 SUCCEED + 刷新缓存]
    end

    J -.-> W
    R -.-> X
