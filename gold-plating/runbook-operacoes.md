# Runbook de Operações — FinTech Wallet
## Gold Plating: Guia de Resposta a Incidentes

> **Finalidade:** Este runbook descreve os procedimentos operacionais para monitoramento, diagnóstico e resposta a incidentes na plataforma FinTech Wallet em ambiente de produção. Deve ser seguido pela equipe de plantão (on-call).

---

## 1. Níveis de Severidade

| Severidade | Critério | SLA de Resposta | SLA de Resolução |
|------------|----------|-----------------|-----------------|
| **SEV-1 — Crítico** | Pagamentos bloqueados / dados financeiros comprometidos | 5 minutos | 1 hora |
| **SEV-2 — Alto** | Taxa de erro > 5% em qualquer Lambda / latência P99 > 3s | 15 minutos | 4 horas |
| **SEV-3 — Médio** | Bureau de crédito indisponível (circuit breaker aberto) | 30 minutos | 8 horas |
| **SEV-4 — Baixo** | Lentidão em notificações / falhas isoladas não financeiras | 2 horas | 24 horas |

---

## 2. Monitoramento — Links Rápidos

```
CloudWatch Dashboard:  https://console.aws.amazon.com/cloudwatch/home#dashboards:name=fintech-wallet-prod
X-Ray Service Map:     https://console.aws.amazon.com/xray/home#/service-map
CloudWatch Alarms:     https://console.aws.amazon.com/cloudwatch/home#alarmsV2
Lambda Console:        https://console.aws.amazon.com/lambda/home?region=sa-east-1
```

---

## 3. Runbooks por Cenário

---

### 3.1 SEV-1 — Pagamentos Bloqueados (Wallet Lambda com alta taxa de erro)

**Sintomas:**
- Alarme `fintech-wallet-errors-production` disparado
- Usuários relatando "erro ao realizar pagamento"
- Taxa de erro no CloudWatch > 10 erros/minuto

**Diagnóstico — passo a passo:**

```bash
# 1. Verificar taxa de erro nos últimos 15 minutos
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=fintech-wallet-production-WalletFunction \
  --start-time $(date -u -d '15 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 900 --statistics Sum

# 2. Buscar logs de erro recentes
aws logs filter-log-events \
  --log-group-name /aws/lambda/fintech-wallet-production-WalletFunction \
  --start-time $(date -d '15 minutes ago' +%s)000 \
  --filter-pattern "ERROR" \
  --query 'events[*].message' \
  --output text | head -50

# 3. Verificar se é problema de DynamoDB (throttling)
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name SystemErrors \
  --dimensions Name=TableName,Value=fintech-wallet \
  --start-time $(date -u -d '15 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 900 --statistics Sum

# 4. Verificar concorrência disponível
aws lambda get-function-concurrency \
  --function-name fintech-wallet-production-WalletFunction
```

**Ações de mitigação:**

```bash
# OPÇÃO A: Rollback imediato para versão anterior (se deploy recente)
# Identificar versão estável anterior
aws lambda list-aliases \
  --function-name fintech-wallet-production-WalletFunction

# Redirecionar alias 'live' para versão anterior
aws lambda update-alias \
  --function-name fintech-wallet-production-WalletFunction \
  --name live \
  --function-version <VERSAO_ANTERIOR> \
  --routing-config '{"AdditionalVersionWeights": {}}'

# OPÇÃO B: Aumentar reserved concurrency temporariamente
aws lambda put-function-concurrency \
  --function-name fintech-wallet-production-WalletFunction \
  --reserved-concurrent-executions 200

# OPÇÃO C: Se o problema for no DynamoDB — verificar e restaurar PITR
aws dynamodb describe-continuous-backups \
  --table-name fintech-wallet
```

**Comunicação:**
1. Postar no canal `#incidents-prod` no Slack: "SEV-1 declarado — pagamentos com falha. Investigando."
2. Notificar PO e tech lead
3. Abrir postmortem em 24h após resolução

---

### 3.2 SEV-2 — Latência Alta no API Gateway (P99 > 3 segundos)

**Diagnóstico:**

```bash
# 1. Verificar latência por rota no API Gateway
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApiGateway \
  --metric-name IntegrationLatency \
  --dimensions Name=ApiName,Value=FinTechWalletAPI \
  --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics p99

# 2. Verificar cold starts
aws logs insights query \
  --log-group-name /aws/lambda/fintech-wallet-production-WalletFunction \
  --start-time $(date -d '30 minutes ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'filter @type = "REPORT" | stats avg(@initDuration) as avgInitMs by bin(5m)'

# 3. X-Ray — identificar gargalo
aws xray get-service-graph \
  --start-time $(date -u -d '30 minutes ago' +%s) \
  --end-time $(date -u +%s)
```

**Ações de mitigação:**

```bash
# Se cold starts forem o problema — habilitar Provisioned Concurrency
aws lambda put-provisioned-concurrency-config \
  --function-name fintech-wallet-production-WalletFunction \
  --qualifier live \
  --provisioned-concurrent-executions 10

# Se for DynamoDB — verificar se DAX está respondendo
aws dax describe-clusters --cluster-names fintech-wallet-dax
```

---

### 3.3 SEV-3 — Circuit Breaker do Bureau de Crédito Aberto

**Sintomas:**
- Log: `[CircuitBreaker] Bureau circuit OPEN — using internal fallback`
- Aprovações de crédito > R$ 200 sendo enfileiradas

**Diagnóstico:**

```bash
# Verificar logs do Credit Engine
aws logs filter-log-events \
  --log-group-name /aws/lambda/fintech-wallet-production-CreditEngineFunction \
  --start-time $(date -d '1 hour ago' +%s)000 \
  --filter-pattern "CircuitBreaker" \
  --query 'events[*].message' \
  --output text

# Testar conectividade com bureau (executar Lambda de health check)
aws lambda invoke \
  --function-name fintech-wallet-production-CreditEngineFunction \
  --payload '{"action": "healthcheck", "target": "bureau"}' \
  response.json && cat response.json
```

**Ações:**
1. Verificar status page do bureau (Serasa: status.serasaexperian.com.br)
2. Se bureau confirmado instável: manter fallback ativo, monitorar a cada 5 minutos
3. Quando bureau recuperar: o circuit breaker transiciona automaticamente para Semi-Aberto
4. Processar fila de créditos represados após normalização

```bash
# Verificar tamanho da fila de crédito represado
aws sqs get-queue-attributes \
  --queue-url https://sqs.sa-east-1.amazonaws.com/<ACCOUNT>/credit-retry-queue \
  --attribute-names ApproximateNumberOfMessages
```

---

### 3.4 Procedimento de Rollback Manual

```bash
# 1. Identificar versão estável (último deploy bem-sucedido)
aws lambda list-versions-by-function \
  --function-name fintech-wallet-production-WalletFunction \
  --query 'Versions[-5:].[Version,LastModified,Description]' \
  --output table

# 2. Executar rollback para versão específica
STABLE_VERSION="42"  # substituir pelo número correto

aws lambda update-alias \
  --function-name fintech-wallet-production-WalletFunction \
  --name live \
  --function-version $STABLE_VERSION

# 3. Confirmar rollback
aws lambda get-alias \
  --function-name fintech-wallet-production-WalletFunction \
  --name live

# 4. Validar com smoke test
curl -f https://api.fintech-wallet.com/health
```

---

## 4. Queries CloudWatch Logs Insights Úteis

```sql
-- Top 10 erros nas últimas 1 hora
fields @timestamp, @message
| filter @message like /ERROR/
| stats count(*) as occurrences by @message
| sort occurrences desc
| limit 10

-- Latência P99 por endpoint (últimos 30 min)
filter @type = "REPORT"
| stats
    pct(@duration, 99) as p99_ms,
    pct(@duration, 95) as p95_ms,
    avg(@duration) as avg_ms,
    count() as invocations
  by bin(5m)

-- Transações financeiras com erro de idempotência
fields @timestamp, correlationId, userId, amount
| filter eventType = "IDEMPOTENCY_CONFLICT"
| sort @timestamp desc
| limit 20

-- Cold starts por função
filter @type = "REPORT" and @initDuration > 0
| stats
    count() as coldStarts,
    avg(@initDuration) as avgInitMs
  by bin(5m)
```

---

## 5. Checklist Pós-Incidente (Postmortem)

Após resolução de qualquer SEV-1 ou SEV-2, abrir documento de postmortem contendo:

- [ ] Timeline precisa do incidente (detecção → mitigação → resolução)
- [ ] Causa raiz identificada (5 Whys)
- [ ] Impacto: número de usuários afetados, volume de transações falhas
- [ ] O que funcionou bem na resposta
- [ ] O que pode melhorar
- [ ] Ações de prevenção com responsável e prazo (máx. 3 itens priorizados)
- [ ] Comunicação enviada para usuários afetados (se aplicável)

> Referência: Template de postmortem baseado no modelo Google SRE (Beyer et al., 2016).

---

## Referências

- Beyer, B. et al. (2016). *Site Reliability Engineering: How Google Runs Production Systems*. O'Reilly Media.
- Kim, G. et al. (2018). *The Site Reliability Workbook*. O'Reilly Media.
- AWS Documentation: *AWS Lambda — Monitoring and observability* (2024).
- Nygard, M. T. (2018). *Release It!, 2nd ed*. Pragmatic Bookshelf. Cap. 8 (Processes on Machines).
