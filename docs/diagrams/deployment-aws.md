# Diagrama de Deployment — C4 Nível 3
## FinTech Wallet — Infraestrutura AWS

Este diagrama detalha a infraestrutura de implantação na AWS, complementando o Diagrama de Containers do README.

---

## Deployment Diagram (C4 Nível 3)

```mermaid
C4Deployment
    title FinTech Wallet — Diagrama de Implantação AWS (C4 Nível 3)

    Deployment_Node(internet, "Internet", "Rede pública") {
        Person(user, "Usuário Final", "App mobile / browser")
    }

    Deployment_Node(aws, "Amazon Web Services", "Região: sa-east-1 (São Paulo)") {

        Deployment_Node(edge, "Edge Layer") {
            Deployment_Node(cloudfront_node, "Amazon CloudFront") {
                Container(cloudfront, "CDN / DDoS Shield", "CloudFront + AWS Shield Standard", "Proteção contra DDoS L3/L4 e aceleração global de entrega")
            }
            Deployment_Node(waf_node, "AWS WAF") {
                Container(waf, "Web Application Firewall", "AWS WAF v2", "Regras OWASP Top 10, bloqueio de IPs suspeitos, rate limiting L7")
            }
        }

        Deployment_Node(apigw_node, "API Layer", "AWS API Gateway (Regional)") {
            Container(apigw, "API Gateway", "REST API + Cognito Authorizer", "Throttling por rota, validação JWT, integração Lambda Proxy")
            Container(cognito, "Amazon Cognito", "User Pool + Identity Pool", "MFA obrigatório para operações financeiras, tokens JWT/OIDC")
        }

        Deployment_Node(compute, "Compute Layer", "AWS Lambda — VPC: fintech-vpc") {

            Deployment_Node(subnet_private_a, "Subnet Privada AZ-A (sa-east-1a)") {
                Container(wallet_a, "Wallet Lambda", "Node.js 20 — 512MB", "Pagamentos, transferências, saldo")
                Container(credit_a, "Credit Engine Lambda", "Python 3.11 — 1024MB", "Scoring, aprovação de crédito")
            }

            Deployment_Node(subnet_private_b, "Subnet Privada AZ-B (sa-east-1b)") {
                Container(auth_b, "Auth Lambda", "Node.js 20 — 256MB", "Validação de tokens, sessões")
                Container(audit_b, "Audit Lambda", "Node.js 20 — 256MB", "Trilha de auditoria imutável")
                Container(notif_b, "Notification Lambda", "Node.js 20 — 256MB", "Push, SMS, e-mail")
            }
        }

        Deployment_Node(messaging, "Messaging Layer") {
            Deployment_Node(eb_node, "Amazon EventBridge") {
                Container(eventbus, "Event Bus", "Custom Event Bus: fintech-events", "Roteamento de eventos de domínio por regras")
            }
            Deployment_Node(sqs_node, "Amazon SQS") {
                Container(sqs_audit, "Fila de Auditoria", "SQS FIFO — audit-events.fifo", "Ordenação garantida, exactly-once processing")
                Container(sqs_notif, "Fila de Notificação", "SQS Standard — notification-events", "Alta throughput, at-least-once delivery")
            }
        }

        Deployment_Node(data, "Data Layer", "Multi-AZ por padrão") {
            Deployment_Node(dynamo_node, "Amazon DynamoDB") {
                ContainerDb(wallet_db, "Wallet Table", "DynamoDB On-Demand + DAX", "Transações financeiras, saldos — ConsistentRead forte")
                ContainerDb(audit_db, "Audit Table", "DynamoDB On-Demand — append-only", "Imutável: sem DeleteItem/UpdateItem no IAM da função")
            }
            Deployment_Node(aurora_node, "Amazon Aurora Serverless v2") {
                ContainerDb(credit_db, "Credit DB", "Aurora PostgreSQL Serverless v2", "Contratos, parcelas, scoring — ACID completo. Scale: 0.5–64 ACUs")
            }
        }

        Deployment_Node(security, "Security Layer") {
            Container(secrets, "Secrets Manager", "AWS Secrets Manager", "Rotação automática de credenciais a cada 30 dias")
            Container(kms, "KMS", "AWS KMS — CMK por serviço", "Chave separada por domínio: wallet-key, credit-key, audit-key")
            Container(cloudtrail, "CloudTrail", "Trail multi-região", "Log de todas as chamadas de API AWS — retido 7 anos")
        }

        Deployment_Node(observability, "Observability Layer") {
            Container(xray, "AWS X-Ray", "Distributed Tracing", "Rastreamento de ponta a ponta por correlation_id")
            Container(cloudwatch, "CloudWatch", "Logs + Metrics + Alarms", "Dashboards por serviço, alertas de latência e erro")
        }
    }

    Deployment_Node(external, "Sistemas Externos") {
        Container(bacen_pix, "BACEN PIX API", "REST/mTLS", "Infraestrutura nacional de transferências instantâneas")
        Container(bureau, "Bureau de Crédito", "REST/mTLS", "Serasa Experian / SPC Brasil")
        Container(antifraud, "Motor Antifraude", "REST", "Detecção de fraude em tempo real")
    }

    Rel(user, cloudfront, "HTTPS", "TLS 1.3")
    Rel(cloudfront, waf, "Proxy")
    Rel(waf, apigw, "HTTPS")
    Rel(apigw, cognito, "JWT validation")
    Rel(apigw, wallet_a, "Lambda Invoke")
    Rel(apigw, credit_a, "Lambda Invoke")
    Rel(apigw, auth_b, "Lambda Invoke")

    Rel(wallet_a, wallet_db, "Read/Write", "DynamoDB SDK + DAX")
    Rel(wallet_a, eventbus, "PutEvents", "EventBridge SDK")
    Rel(wallet_a, bacen_pix, "HTTPS/mTLS")
    Rel(wallet_a, antifraud, "HTTPS")
    Rel(wallet_a, secrets, "GetSecretValue")

    Rel(credit_a, credit_db, "SQL/SSL", "JDBC via RDS Proxy")
    Rel(credit_a, bureau, "HTTPS/mTLS")
    Rel(credit_a, eventbus, "PutEvents")
    Rel(credit_a, kms, "Decrypt")

    Rel(eventbus, sqs_audit, "Route rule: audit.*")
    Rel(eventbus, sqs_notif, "Route rule: notification.*")
    Rel(sqs_audit, audit_b, "SQS Trigger")
    Rel(sqs_notif, notif_b, "SQS Trigger")

    Rel(audit_b, audit_db, "PutItem", "append-only")

    Rel(wallet_a, xray, "Trace segments")
    Rel(credit_a, xray, "Trace segments")
    Rel(cloudtrail, cloudwatch, "Log delivery")
```

---

## Notas de Infraestrutura

### VPC e Isolamento de Rede

Todas as Lambdas operam dentro de uma **VPC dedicada** (`fintech-vpc`) com:
- Subnets privadas por AZ (sem acesso direto à internet)
- **NAT Gateway** para chamadas de saída (PIX, bureau, antifraude)
- **VPC Endpoints** para DynamoDB, SQS, Secrets Manager, KMS e EventBridge (tráfego não sai da rede AWS)
- Security Groups restritivos: cada Lambda tem acesso apenas aos recursos que precisa

### Alta Disponibilidade

| Componente | AZs | SLA AWS |
|------------|-----|---------|
| Lambda | Multi-AZ automático | 99,95% |
| DynamoDB | 3 AZs | 99,999% |
| Aurora Serverless v2 | Multi-AZ | 99,99% |
| SQS | Multi-AZ | 99,9% |
| API Gateway | Multi-AZ | 99,95% |

### Estimativa de Custo (baseline — 100k transações/mês)

| Serviço | Estimativa Mensal |
|---------|------------------|
| Lambda (Wallet + Credit) | ~$8 |
| API Gateway | ~$3,50 |
| DynamoDB On-Demand | ~$5 |
| Aurora Serverless v2 (0.5 ACU idle) | ~$12 |
| SQS + EventBridge | ~$1 |
| Secrets Manager + KMS | ~$3 |
| CloudWatch + X-Ray | ~$5 |
| **Total estimado** | **~$37,50/mês** |

> Custo escala linearmente com o volume. Para 1M transações/mês, estimativa ~$180/mês.
