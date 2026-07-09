```mermaid
flowchart TD
    A[POST /assets/withdrawals<br/>X-Request-Id] --> B{实名认证?}
    B -->|否| B1[拒绝]
    B -->|是| C[校验交易密码]
    C --> D[必须是外部地址]
    D --> E[地址风险校验<br/>仅低危通过]
    E --> F{金额达安全认证阈值?}
    F -->|是| F1[校验 X-Security-Auth-Token]
    F -->|否| G
    F1 --> G[RequestId → bizNo]
    G --> H[WithdrawService.withdraw]

    H --> I{bizNo 订单已存在?}
    I -->|是| I1[幂等返回已有单]
    I -->|否| J[校验 TRON 地址 / 查链地址<br/>算手续费 / 校验可用余额]
    J --> K{金额达审批阈值?}
    K -->|是| L[建单 PENDING_APPROVAL]
    K -->|否| M[建单 PENDING]
    L --> N[账户中心 withdrawFreeze<br/>USER_AVAILABLE → USER_WITHDRAW<br/>fee → PLATFORM_WITHDRAW_FEE]
    M --> N
    N --> O{需审批?}
    O -->|是| P[返回待审批]
    O -->|否| Q[blockchainClient.sendToken 发链]

    P --> R[Admin 审批]
    R -->|通过| S[状态 → PENDING<br/>再 sendToken]
    R -->|驳回| T[withdrawReject 解冻<br/>状态 → CANCELLED]

    Q --> U[WithdrawConfirmJob 轮询]
    S --> U
    U --> V{链上状态 SUCCEED?}
    V -->|否| U
    V -->|是| W[withdrawSuccess<br/>USER_WITHDRAW → PLATFORM_WITHDRAW_CLEARING]
    W --> X[订单 → SUCCEED]

    subgraph reconcile [WithdrawReconcileJob 对账]
        Y[创建超 5 分钟仍 PENDING] --> Z{冻结流水存在?}
        Z -->|否| Z1[订单 → CANCELLED]
        Z -->|是| Z2[重试 sendToken]
    end

    M -.-> Y
    Q -.-> Y
