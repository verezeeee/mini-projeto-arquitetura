# SAD — Software Architecture Document
## FinTech Wallet: Carteira Digital com Microcrédito Instantâneo
### Fase 3 — Cloud & Microsserviços Serverless

**Versão:** 3.0  
**Data:** Junho/2025  
**Status:** Aprovado

---

## 1. Introdução

### 1.1 Propósito

Este documento descreve a arquitetura de software da **FinTech Wallet** na Fase 3 do projeto, cobrindo a estratégia de implantação em nuvem, os padrões de resiliência adotados e as decisões de comunicação entre os componentes. Serve como referência técnica para a equipe de desenvolvimento, avaliadores e stakeholders.

### 1.2 Escopo

O sistema abrange:
- Carteira digital (pagamentos, transferências, histórico)
- Motor de microcrédito instantâneo com scoring automatizado
- Autenticação segura com MFA
- Auditoria imutável de todas as operações
- Integração com ecossistema PIX (BACEN) e bureaus de crédito externos

### 1.3 Definições e Siglas

| Sigla | Significado |
|-------|-------------|
| PCI DSS | Payment Card Industry Data Security Standard |
| BACEN | Banco Central do Brasil |
| ACID | Atomicity, Consistency, Isolation, Durability |
| ADR | Architecture Decision Record |
| SQS | Amazon Simple Queue Service |
| API GW | AWS API Gateway |
| Lambda | AWS Lambda (função serverless) |
| MFA | Multi-Factor Authentication |

---

## 2. Representação Arquitetural

A arquitetura da Fase 3 segue o **modelo Serverless na AWS**, organizado em torno de cinco funções Lambda com responsabilidades de domínio bem definidas, coordenadas por um API Gateway e um barramento de eventos (EventBridge + SQS).

### 2.1 Visão de Alto Nível

```
[Usuário / Analista]
        │
        ▼
[AWS API Gateway + WAF + Cognito]
        │
   ┌────┴─────────────────────┐
   ▼                          ▼
[Auth Lambda]         [Wallet Lambda] ──► [DynamoDB]
                             │           [PIX API]
                             │           [Antifraude]
                             ▼
                      [EventBridge]
                      /           \
              [Audit Lambda]  [Notification Lambda]
                    │
              [Audit DynamoDB]

[Credit Lambda] ──► [Aurora Serverless]
                 ──► [Bureau de Crédito]
                 ──► [EventBridge]
```

---

## 3. Objetivos e Restrições Arquiteturais

### 3.1 Requisitos Não Funcionais Críticos

| RNF | Meta | Estratégia |
|-----|------|------------|
| **Segurança** | PCI DSS Level 1 + BACEN 4.658 | Cognito MFA, KMS, mTLS, WAF, auditoria imutável |
| **Consistência** | ACID para transações financeiras | Aurora Serverless + transações distribuídas com idempotência |
| **Auditabilidade** | 100% das operações rastreadas | EventBridge → SQS FIFO → Audit Lambda → DynamoDB append-only |
| **Latência** | P99 < 500ms para pagamentos | Lambda @Edge, DynamoDB com cache DAX, Cold Start mitigation via Provisioned Concurrency |
| **Escalabilidade** | 0 a 10.000 req/s sem intervenção manual | Auto-scaling nativo do Lambda e DynamoDB On-Demand |

### 3.2 Restrições

- Conformidade obrigatória com regulação BACEN e LGPD
- Dados sensíveis (PAN, CPF) nunca em logs — mascaramento obrigatório
- Toda comunicação externa via mTLS
- Custo operacional deve escalar linearmente com o uso (pay-per-use)

---

## 4. Visão de Casos de Uso Relevantes

| ID | Caso de Uso | Componentes Envolvidos |
|----|-------------|----------------------|
| UC-01 | Realizar pagamento P2P | API GW → Wallet Lambda → DynamoDB → EventBridge |
| UC-02 | Transferência via PIX | API GW → Wallet Lambda → PIX API (BACEN) |
| UC-03 | Solicitar microcrédito | API GW → Credit Lambda → Aurora → Bureau |
| UC-04 | Consultar extrato | API GW → Wallet Lambda → DynamoDB |
| UC-05 | Autenticar com MFA | API GW → Auth Lambda → Cognito |
| UC-06 | Auditoria de operação | EventBridge → Audit Lambda → Audit DynamoDB |

---

## 5. Visão Lógica

### 5.1 Decomposição em Serviços (Domínios)

A decomposição segue o princípio de **Domain-Driven Design (DDD)**, com cada Lambda representando um Bounded Context:

**Wallet Domain**
- Gerencia saldo, transações, extrato
- Garantia de idempotência: toda transação possui `idempotencyKey` único
- Operações de débito/crédito são atômicas via transações condicionais do DynamoDB (`TransactWriteItems`)

**Credit Domain**
- Motor de scoring: analisa histórico transacional interno + consulta bureau externo
- Aprovação automática até R$ 500; acima disso, fila de revisão manual
- Contratos de crédito e parcelas em Aurora Serverless (ACID completo)

**Auth Domain**
- Amazon Cognito gerencia identidades, tokens JWT e MFA
- Lambda custom authorizer valida tokens em cada requisição

**Audit Domain**
- Consome 100% dos eventos do EventBridge
- Tabela DynamoDB com `append-only` policy (sem permissão de `DeleteItem` ou `UpdateItem` no IAM da função)
- Dados retidos por 5 anos conforme regulação BACEN

**Notification Domain**
- Consume fila SQS Standard
- Envia notificações push (SNS), SMS e e-mail (SES)

### 5.2 Modelo de Dados Simplificado

**DynamoDB — Wallet Table**
```
PK: USER#{userId}
SK: TXN#{timestamp}#{txnId}
Atributos: amount, type, status, idempotencyKey, createdAt
```

**Aurora — Credit Table (simplificado)**
```sql
contracts(id, user_id, amount, status, score, created_at, updated_at)
installments(id, contract_id, due_date, amount, paid_at)
```

---

## 6. Visão de Implantação (Cloud)

### 6.1 Infraestrutura AWS

| Componente | Serviço AWS | Justificativa |
|------------|-------------|---------------|
| Funções de negócio | AWS Lambda | Escalabilidade automática, pay-per-use |
| API unificada | AWS API Gateway | Throttling, WAF, Cognito Authorizer |
| Identidade | Amazon Cognito | MFA nativo, padrão OAuth 2.0 / OIDC |
| Dados transacionais | Amazon DynamoDB | Latência sub-milissegundo, escalabilidade horizontal |
| Dados relacionais (crédito) | Aurora Serverless v2 | ACID completo, scale-to-zero |
| Mensageria | Amazon SQS FIFO + Standard | Ordenação garantida (FIFO) para auditoria |
| Eventos | Amazon EventBridge | Desacoplamento entre domínios |
| Segredos | AWS Secrets Manager + KMS | Rotação automática, criptografia em repouso |
| Monitoramento | CloudWatch + X-Ray | Rastreamento distribuído e alertas |

### 6.2 Estratégia de Escalabilidade

- **Lambda**: escala automaticamente de 0 a 1.000 execuções concorrentes por padrão; Provisioned Concurrency para funções críticas (Wallet, Auth) elimina cold starts
- **DynamoDB On-Demand**: escala leitura/escrita sem provisionamento manual
- **Aurora Serverless v2**: escala de 0.5 a 64 ACUs automaticamente
- **API Gateway**: sem limite de requisições por padrão; throttling configurável por rota

### 6.3 Multi-AZ e Disaster Recovery

- DynamoDB e Aurora replicam automaticamente em 3 Availability Zones
- RTO (Recovery Time Objective): < 5 minutos
- RPO (Recovery Point Objective): < 1 minuto (DynamoDB Point-in-Time Recovery)

---

## 7. Visão de Segurança

### 7.1 Camadas de Segurança

```
Internet → CloudFront (DDoS) → WAF (OWASP rules) → API Gateway → Cognito Authorizer → Lambda
```

### 7.2 Controles Implementados

| Controle | Implementação |
|----------|---------------|
| Autenticação | JWT + MFA obrigatório para operações financeiras |
| Autorização | IAM Roles mínimas por Lambda (Princípio do Menor Privilégio) |
| Criptografia em trânsito | TLS 1.3 para todas as conexões externas; mTLS para integrações com BACEN/bureaus |
| Criptografia em repouso | KMS Customer Managed Keys para DynamoDB, Aurora e S3 |
| Dados sensíveis | Mascaramento de CPF e PAN nos logs; nunca persistidos em texto claro |
| Auditoria | 100% das operações rastreadas em tabela imutável |

---

## 8. Decisões Arquiteturais (Referências)

As principais decisões desta fase estão documentadas como ADRs:

- **[ADR-0001](../adrs/0001-estrategia-nuvem.md)** — Estratégia de Nuvem: AWS Serverless (Lambda + API Gateway)
- **[ADR-0002](../adrs/0002-padrao-resiliencia.md)** — Padrões de Resiliência: API Gateway + Circuit Breaker + Idempotência
- **[ADR-0003](../adrs/0003-modelo-comunicacao.md)** — Modelo de Comunicação: Síncrono para transações, Assíncrono para eventos

---

## 9. Riscos e Mitigações

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| Cold start em horário de pico | Média | Alto | Provisioned Concurrency nas funções críticas |
| Falha do bureau de crédito externo | Média | Médio | Circuit Breaker + fallback de score interno |
| Vazamento de dados sensíveis | Baixa | Crítico | KMS, mascaramento, auditoria imutável, VPC Endpoints |
| Lock-in na AWS | Baixa | Médio | Interfaces abstratas nos serviços; código de domínio sem SDK direto |
| Custo explosivo em pico inesperado | Baixa | Médio | AWS Budgets alerts + throttling no API Gateway |

---

## 10. Evolução Futura (Roadmap)

- **Fase 4**: Implementação de Event Sourcing completo para o domínio de crédito
- **Fase 5**: Multi-região ativo-ativo (us-east-1 + sa-east-1) para continuidade de negócio
- **ML Integration**: Motor de scoring próprio com Amazon SageMaker substituindo bureau externo para perfis recorrentes

---

## 11. Referências

- Pressman, R. S. (2021). *Engenharia de Software: Uma Abordagem Profissional*. McGraw Hill.
- Richards, M., & Ford, N. (2020). *Fundamentals of Software Architecture*. O'Reilly Media.
- Newman, S. (2019). *Building Microservices, 2nd ed*. O'Reilly Media.
- AWS Well-Architected Framework — Security Pillar. Amazon Web Services, 2023.
- C4 Model: https://c4model.com/
