# Checklist de Conformidade — PCI DSS & BACEN
## Gold Plating: Mapeamento Arquitetural de Controles Regulatórios

> Este documento mapeia como cada decisão arquitetural da FinTech Wallet atende aos requisitos do **PCI DSS v4.0** e da **Resolução BACEN 4.658/2018**, servindo como evidência técnica para auditorias regulatórias.

---

## PCI DSS v4.0 — Mapeamento de Requisitos

### Requisito 1 — Instalar e Manter Controles de Segurança de Rede

| Controle PCI DSS | Status | Implementação Arquitetural |
|------------------|--------|---------------------------|
| 1.2.1 — Restrição de tráfego inbound/outbound | ✅ | Security Groups por Lambda: apenas portas/origens necessárias |
| 1.3.1 — Proibir acesso direto à internet para componentes do cardholder data | ✅ | Lambdas em subnets privadas + VPC Endpoints para serviços AWS |
| 1.4.1 — NSC entre redes confiáveis e não-confiáveis | ✅ | WAF + API Gateway como DMZ; rede interna isolada na VPC |

### Requisito 2 — Aplicar Configurações Seguras

| Controle PCI DSS | Status | Implementação Arquitetural |
|------------------|--------|---------------------------|
| 2.2.1 — Configurações padrão do fornecedor alteradas | ✅ | IAM Roles mínimas por Lambda; sem credenciais default |
| 2.3.1 — Componentes sem função desnecessária | ✅ | Lambda single-responsibility; sem portas abertas além do necessário |
| 2.3.2 — Chaves criptográficas gerenciadas | ✅ | AWS KMS CMK por domínio; rotação automática a cada 365 dias |

### Requisito 3 — Proteger Dados de Conta Armazenados

| Controle PCI DSS | Status | Implementação Arquitetural |
|------------------|--------|---------------------------|
| 3.3.1 — SAD não retido após autorização | ✅ | Dados sensíveis nunca persistidos; apenas hash + tokens |
| 3.4.1 — PAN mascarado quando exibido | ✅ | Mascaramento obrigatório em logs e respostas de API |
| 3.5.1 — PAN protegido com criptografia forte | ✅ | KMS AES-256 em repouso; TLS 1.3 em trânsito |
| 3.6.1 — Procedimentos para gestão de chaves | ✅ | Secrets Manager + KMS: rotação automática, auditoria via CloudTrail |

### Requisito 4 — Proteger Dados do Titular com Criptografia em Trânsito

| Controle PCI DSS | Status | Implementação Arquitetural |
|------------------|--------|---------------------------|
| 4.2.1 — Criptografia forte em redes públicas | ✅ | TLS 1.3 obrigatório no API Gateway; TLS 1.2 deprecated |
| 4.2.1.1 — Inventário de certificados | ✅ | AWS Certificate Manager com renovação automática |
| 4.2.2 — PANs nunca enviados por canais não seguros | ✅ | mTLS para integrações com BACEN e bureaus |

### Requisito 6 — Desenvolver e Manter Sistemas Seguros

| Controle PCI DSS | Status | Implementação Arquitetural |
|------------------|--------|---------------------------|
| 6.2.4 — Prevenir vulnerabilidades de software comuns | ✅ | SAST via Trivy no CI/CD; npm audit em cada build |
| 6.3.2 — Inventário de software de terceiros | ✅ | `package-lock.json` e `requirements.txt` versionados |
| 6.4.1 — WAF em aplicações web públicas | ✅ | AWS WAF v2 com regras OWASP Top 10 gerenciadas |

### Requisito 7 — Restringir Acesso a Componentes do Sistema

| Controle PCI DSS | Status | Implementação Arquitetural |
|------------------|--------|---------------------------|
| 7.2.1 — Acesso negado por padrão | ✅ | IAM Deny por default; permissões explícitas por função Lambda |
| 7.3.1 — Controle de acesso baseado em necessidade | ✅ | IAM Roles mínimas: cada Lambda acessa apenas seus recursos |

### Requisito 8 — Identificar Usuários e Autenticar Acesso

| Controle PCI DSS | Status | Implementação Arquitetural |
|------------------|--------|---------------------------|
| 8.3.6 — Senhas com complexidade mínima | ✅ | Cognito User Pool com política de senha forte |
| 8.4.2 — MFA para acesso a CDE | ✅ | MFA obrigatório para operações financeiras (TOTP via Cognito) |
| 8.6.1 — Controle de contas de sistema e aplicação | ✅ | Sem credenciais hardcoded; toda autenticação via IAM Roles + Cognito |

### Requisito 10 — Registrar e Monitorar Acessos

| Controle PCI DSS | Status | Implementação Arquitetural |
|------------------|--------|---------------------------|
| 10.2.1 — Trilha de auditoria para todas as ações | ✅ | Audit Lambda → DynamoDB append-only; 100% das operações rastreadas |
| 10.3.2 — Logs protegidos contra modificação | ✅ | Tabela DynamoDB sem DeleteItem/UpdateItem; CloudTrail imutável |
| 10.5.1 — Retenção de logs por 12 meses (3 meses online) | ✅ | DynamoDB retém 12 meses; export mensal para S3 Glacier para 5 anos |
| 10.6.3 — Sincronização de tempo | ✅ | Lambdas usam NTP da AWS automaticamente |

### Requisito 11 — Testar Segurança de Sistemas e Redes

| Controle PCI DSS | Status | Implementação Arquitetural |
|------------------|--------|---------------------------|
| 11.3.1 — Scan de vulnerabilidades trimestrais | ⚠️ | Trivy no CI/CD; scan manual trimestral ainda não agendado |
| 11.4.1 — Penetration testing anual | ⚠️ | Pendente contratação de empresa especializada |
| 11.6.1 — Mecanismo de detecção de mudanças | ✅ | AWS Config com regras de conformidade; alertas de drift |

### Requisito 12 — Manter Política de Segurança da Informação

| Controle PCI DSS | Status | Implementação Arquitetural |
|------------------|--------|---------------------------|
| 12.3.2 — Avaliação de risco anual | ⚠️ | Processo a ser formalizado |
| 12.6.2 — Treinamento de conscientização de segurança | ⚠️ | Pendente para toda a equipe |

> **Legenda:** ✅ Implementado | ⚠️ Parcialmente implementado / planejado | ❌ Não implementado

---

## Resolução BACEN 4.658/2018 — Política de Segurança Cibernética

A Resolução 4.658/2018 do Banco Central do Brasil exige que instituições financeiras estabeleçam **Política de Segurança Cibernética** com controles específicos. O mapeamento abaixo demonstra a aderência arquitetural.

### Art. 6 — Requisitos Mínimos da Política

| Requisito BACEN | Status | Implementação |
|-----------------|--------|---------------|
| I — Autenticação, criptografia, prevenção e detecção de intrusão | ✅ | Cognito MFA + KMS + WAF OWASP + CloudTrail |
| II — Prevenção contra vazamento de dados | ✅ | Mascaramento de dados sensíveis; VPC Endpoints; criptografia total |
| III — Realização de backups e testes de restauração | ✅ | DynamoDB PITR + Aurora automated backups; testes de restauração mensais |
| IV — Proteção contra softwares maliciosos | ✅ | Trivy SAST no CI/CD; imagens efêmeras do Lambda (sem SO persistente) |
| V — Controles de acesso e autenticação | ✅ | IAM Least Privilege + Cognito + MFA obrigatório |
| VI — Criptografia de dados | ✅ | KMS AES-256 em repouso; TLS 1.3 em trânsito |
| VII — Monitoramento e rastreamento de acessos | ✅ | CloudTrail + Audit Lambda + X-Ray |
| VIII — Classificação de dados | ⚠️ | Mapeamento de dados sensíveis realizado; política formal em elaboração |

### Art. 11 — Requisitos para Uso de Serviços em Nuvem

| Requisito BACEN | Status | Implementação |
|-----------------|--------|---------------|
| I — Contratos com fornecedor de nuvem | ✅ | AWS BAA (Business Associate Agreement) + contratos regulatórios |
| II — Auditoria e fiscalização pela instituição | ✅ | CloudTrail + AWS Config + AWS Audit Manager |
| III — Segregação de dados de clientes | ✅ | KMS CMK por domínio; VPC isolada; sem compartilhamento de dados entre tenants |
| IV — Localização dos dados no Brasil ou países autorizados | ✅ | Região AWS sa-east-1 (São Paulo) como região primária |
| V — Portabilidade dos dados | ⚠️ | DynamoDB export para S3 disponível; processo de portabilidade a formalizar |

### Art. 16 — Plano de Continuidade de Negócios

| Requisito BACEN | Status | Implementação |
|-----------------|--------|---------------|
| RTO (Recovery Time Objective) | ✅ | < 5 minutos — Lambda + DynamoDB Multi-AZ automático |
| RPO (Recovery Point Objective) | ✅ | < 1 minuto — DynamoDB PITR contínuo |
| Testes de continuidade semestrais | ⚠️ | Procedimento definido no Runbook; testes formais a agendar |
| Plano documentado e aprovado | ⚠️ | Runbook técnico disponível; plano formal em elaboração |

---

## Resumo de Gaps e Plano de Ação

| Gap Identificado | Severidade | Prazo Sugerido | Responsável |
|------------------|-----------|----------------|-------------|
| Scan de vulnerabilidades trimestral agendado | Médio | 30 dias | DevOps |
| Penetration testing anual contratado | Alto | 60 dias | CTO |
| Política formal de classificação de dados | Médio | 45 dias | Security |
| Treinamento de conscientização de segurança | Alto | 30 dias | RH + Security |
| Plano formal de continuidade de negócios | Alto | 60 dias | CTO + Juridico |
| Portabilidade de dados formalizada | Baixo | 90 dias | Produto + Juridico |

---

## Referências Regulatórias

- PCI Security Standards Council. (2022). *PCI DSS v4.0*. Disponível em: pcisecuritystandards.org
- Banco Central do Brasil. (2018). *Resolução CMN 4.658/2018 — Política de Segurança Cibernética*.
- Banco Central do Brasil. (2021). *Resolução BCB 85/2021 — Continuidade de Serviços Críticos*.
- LGPD — Lei Geral de Proteção de Dados Pessoais. Lei nº 13.709/2018.
- NIST. (2018). *Cybersecurity Framework v1.1*. National Institute of Standards and Technology.
