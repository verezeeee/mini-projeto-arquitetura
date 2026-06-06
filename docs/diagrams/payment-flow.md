# Diagrama de Fluxo — Pagamento P2P e PIX
## FinTech Wallet

Este diagrama detalha o fluxo completo de uma transação financeira, desde a requisição do usuário até a confirmação final, incluindo os pontos de verificação de segurança, idempotência e resiliência.

---

## Fluxo de Pagamento P2P (Transferência entre usuários da plataforma)

```mermaid
sequenceDiagram
    actor U as Usuário
    participant GW as API Gateway + WAF
    participant AL as Auth Lambda
    participant WL as Wallet Lambda
    participant AF as Antifraude (ext.)
    participant DB as DynamoDB
    participant EB as EventBridge
    participant AUD as Audit Lambda
    participant NOT as Notification Lambda

    U->>GW: POST /payments {amount, toUserId, idempotencyKey}
    GW->>GW: WAF rules check (OWASP, rate limit)
    GW->>AL: Validate JWT + MFA token
    AL-->>GW: 200 OK — token válido, userId extraído

    GW->>WL: Invoke (payload completo)

    Note over WL: 1. Verificar idempotencyKey no DynamoDB
    WL->>DB: GetItem (idempotencyKey)
    alt Chave já existe
        DB-->>WL: {status: "completed", result: ...}
        WL-->>GW: 200 OK (resultado cacheado — sem reprocessamento)
        GW-->>U: 200 OK ✓ (idempotente)
    else Chave não existe
        DB-->>WL: null

        Note over WL: 2. Verificar saldo do remetente
        WL->>DB: GetItem (USER#{senderId}) — ConsistentRead=true
        DB-->>WL: {balance: 500.00}

        alt Saldo insuficiente
            WL-->>GW: 422 Insufficient Funds
            GW-->>U: 422 ✗
        else Saldo suficiente

            Note over WL: 3. Verificação antifraude (síncrona)
            WL->>AF: POST /analyze {userId, amount, toUserId, deviceId}

            alt Antifraude indisponível (Circuit Breaker OPEN)
                AF-->>WL: timeout / 5xx
                Note over WL: Fallback: aprova se amount ≤ R$500
                WL->>WL: amount ≤ 500? → prossegue com flag "fraud_check_pending"
            else Antifraude disponível
                AF-->>WL: {risk: "low" | "medium" | "high"}
                alt Risco alto
                    WL-->>GW: 403 Transaction Blocked
                    GW-->>U: 403 ✗ (suspeita de fraude)
                end
            end

            Note over WL: 4. Débito atômico (TransactWriteItems)
            WL->>DB: TransactWriteItems<br/>- Débito USER#{senderId}<br/>- Crédito USER#{toUserId}<br/>- Insert TXN#{txnId}<br/>- Insert IDEMPOTENCY#{key}
            
            alt Transação falhou (ex: saldo mudou — condição de corrida)
                DB-->>WL: TransactionCanceledException
                WL-->>GW: 409 Conflict — tente novamente
                GW-->>U: 409 ✗
            else Transação confirmada
                DB-->>WL: OK

                Note over WL: 5. Publicar eventos de domínio
                WL->>EB: PutEvent: PaymentCompleted {txnId, senderId, toUserId, amount}
                EB->>AUD: Route → audit-events.fifo
                EB->>NOT: Route → notification-events

                WL-->>GW: 200 OK {txnId, status: "completed", timestamp}
                GW-->>U: 200 OK ✓

                Note over AUD: Assíncrono — não bloqueia resposta
                AUD->>DB: PutItem audit-table (append-only)

                Note over NOT: Assíncrono — não bloqueia resposta
                NOT->>U: Push notification "Você enviou R$ X para Y"
            end
        end
    end
```

---

## Fluxo de Transferência via PIX

```mermaid
sequenceDiagram
    actor U as Usuário
    participant GW as API Gateway
    participant WL as Wallet Lambda
    participant DB as DynamoDB
    participant PIX as BACEN PIX API
    participant EB as EventBridge

    U->>GW: POST /pix/transfer {pixKey, amount, idempotencyKey}
    GW->>WL: Invoke

    WL->>DB: Verificar idempotencyKey
    WL->>DB: Verificar saldo (ConsistentRead=true)

    Note over WL: Montar payload PIX (DICT lookup + SPI)
    WL->>PIX: POST /pix/initiation (mTLS)<br/>{chave, valor, endToEndId}

    alt PIX API timeout (> 10s)
        PIX-->>WL: timeout
        WL->>DB: Marcar TXN como "pending_external"
        WL-->>GW: 202 Accepted {txnId, status: "processing"}
        GW-->>U: 202 — "Transferência em processamento"
        Note over WL: Webhook do BACEN confirma depois
    else PIX retorna sucesso
        PIX-->>WL: {endToEndId, status: "ACCC", timestamp}
        WL->>DB: TransactWriteItems — débito + registro TXN
        WL->>EB: PutEvent: PixTransferCompleted
        WL-->>GW: 200 OK {txnId, endToEndId, status: "completed"}
        GW-->>U: 200 OK ✓
    else PIX retorna erro
        PIX-->>WL: {status: "RJCT", reason: "..."}
        WL-->>GW: 422 {reason: motivo da rejeição}
        GW-->>U: 422 ✗
    end
```

---

## Fluxo de Aprovação de Microcrédito

```mermaid
flowchart TD
    A([Usuário solicita crédito\nR$ X por Y meses]) --> B[API Gateway\nvalida JWT + MFA]
    B --> C[Credit Engine Lambda]
    
    C --> D{Usuário tem\ncontrato ativo?}
    D -- Sim --> E[Retorna 409\nConflito — limite já em uso]
    D -- Não --> F[Consulta histórico\ntransacional interno\nDynamoDB]
    
    F --> G{Circuit Breaker\nBureau OPEN?}
    
    G -- Sim → Fallback --> H[Usa score interno\n0–1000 baseado em\nhistórico próprio]
    G -- Não → Normal --> I[POST bureau externo\nSerasa/SPC - mTLS]
    I --> J[Score externo\n+ dados CADSIN BACEN]
    J --> K[Score combinado\n70% externo + 30% interno]
    H --> K
    
    K --> L{Score\n≥ 600?}
    
    L -- Não --> M[Retorna 403\nCrédito negado\n+ motivo]
    
    L -- Sim --> N{Valor solicitado\n≤ R$ 500?}
    
    N -- Sim --> O[Aprovação automática\nAurora: INSERT contract]
    N -- Não --> P{Score\n≥ 800?}
    
    P -- Sim --> O
    P -- Não --> Q[Fila de revisão\nmanual — SQS]
    Q --> R[Retorna 202 Accepted\nEm análise — 24h]
    
    O --> S[EventBridge:\nCreditApproved event]
    S --> T[Notification Lambda:\nSMS + Push ao usuário]
    S --> U[Audit Lambda:\nregistra decisão + score]
    O --> V[Retorna 201 Created\ncontrato + parcelas]
```

---

## Estados do Circuit Breaker — Bureau de Crédito

```mermaid
stateDiagram-v2
    [*] --> Fechado : Inicialização

    Fechado --> Aberto : 5 falhas consecutivas\nOU taxa erro > 50% em 60s

    Aberto --> SemiAberto : Timeout de espera\n(30 segundos)

    SemiAberto --> Fechado : Requisição de teste\nbem-sucedida

    SemiAberto --> Aberto : Requisição de teste\nfalhou

    state Fechado {
        [*] --> Monitorando
        Monitorando --> Monitorando : Requisição OK\n(reset contador)
    }

    state Aberto {
        [*] --> Bloqueando
        Bloqueando --> Bloqueando : Retorna fallback\nimediatamente\n(sem chamar bureau)
    }

    state SemiAberto {
        [*] --> Testando
        Testando --> Testando : 1 requisição\npermitida por vez
    }
```
