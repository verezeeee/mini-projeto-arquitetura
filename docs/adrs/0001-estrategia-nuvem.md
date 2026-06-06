# ADR-0001 — Estratégia de Nuvem e Escalabilidade

**Data:** Junho/2025  
**Status:** Aceito  
**Deciders:** Equipe FinTech Wallet  
**Projeto:** FinTech Wallet — Fase 3 (Cloud & Microsserviços)

---

## Contexto

Na Fase 2, a arquitetura da FinTech Wallet foi definida como um conjunto de microsserviços containerizados. Com a evolução para a Fase 3, o time precisa decidir **como implantar e operar esses serviços em ambiente de nuvem pública**, levando em conta os requisitos não funcionais críticos do sistema: segurança financeira (PCI DSS / BACEN), consistência de dados (ACID), auditabilidade e baixa latência.

O perfil de carga da plataforma é **altamente irregular**: picos ocorrem em horários comerciais e datas de pagamento (5º dia útil, por exemplo), com volumes potencialmente 10x maiores que a média. Em períodos de baixo uso (madrugada, fins de semana), a plataforma pode ter demanda próxima de zero.

A escolha do modelo de serviço em nuvem impacta diretamente: custo operacional, capacidade de escala, complexidade de operação (DevOps), segurança e conformidade regulatória.

---

## Decisão

Adotar **AWS Serverless como modelo principal de implantação**, com as seguintes tecnologias:

- **AWS Lambda** para todas as funções de negócio (Wallet, Credit Engine, Auth, Audit, Notification)
- **Amazon API Gateway** como ponto de entrada unificado, com WAF e Cognito Authorizer integrados
- **Amazon DynamoDB** (On-Demand) para dados transacionais de alta velocidade
- **Amazon Aurora Serverless v2** para o domínio de crédito, que exige transações ACID completas
- **Amazon EventBridge + SQS** para comunicação assíncrona entre domínios
- **AWS Secrets Manager + KMS** para gestão de segredos e criptografia

---

## Justificativa

### Por que Serverless (PaaS/FaaS) em vez de IaaS ou containers?

A decisão foi fundamentada em quatro eixos:

#### 1. Escalabilidade automática horizontal

Richardson (2018) e os princípios do *AWS Well-Architected Framework* apontam que sistemas de pagamento exigem escalabilidade **elástica e sem intervenção manual**, especialmente em picos imprevisíveis. O modelo IaaS (instâncias EC2) exigiria provisionamento antecipado ou uso de Auto Scaling Groups com latência de inicialização de 2 a 5 minutos — incompatível com a natureza transacional do sistema. Containers (ECS/EKS) reduzem esse tempo, mas ainda exigem gerenciamento de cluster, configuração de Health Checks e orquestração.

O Lambda, ao contrário, escala de **0 a milhares de execuções concorrentes em milissegundos**, sem nenhuma intervenção operacional. A escalabilidade horizontal nativa elimina a necessidade de dimensionamento manual.

#### 2. Modelo econômico alinhado ao perfil de uso

Conforme Richards e Ford (2020), a escolha de arquitetura deve considerar o *fitness function* econômico do sistema. O modelo **pay-per-use do Lambda** (cobrado por 1ms de execução) é drasticamente mais eficiente para cargas irregulares: durante a madrugada ou fins de semana de baixo volume, o custo é zero ou próximo disso. Um cluster ECS ou instâncias EC2 provisionadas pagariam pela capacidade ociosa 24/7.

#### 3. Redução da superfície operacional (Operações e Segurança)

Pressman (2021) destaca que a complexidade operacional é um fator de risco arquitetural frequentemente subestimado. No modelo Serverless, a AWS gerencia patches de SO, atualizações de runtime, balanceamento de carga e disponibilidade da infraestrutura. Para uma FinTech em estágio inicial, isso representa ganho direto: a equipe foca no domínio de negócio, não em operações de infraestrutura.

Do ponto de vista de segurança, cada Lambda roda em um ambiente de execução isolado e efêmero, dificultando ataques de persistência. Combinado com IAM Roles de menor privilégio por função, o modelo reduz o raio de impacto de uma eventual comprometimento.

#### 4. Conformidade com PCI DSS e BACEN

A AWS mantém certificação PCI DSS Level 1 — o mais alto nível — e oferece serviços como KMS, CloudTrail, AWS Config e Macie que facilitam a conformidade com a Resolução BACEN 4.658/2018, que trata de segurança de dados para instituições financeiras. O uso de serviços gerenciados reduz o escopo de auditoria do time, pois a responsabilidade pela infraestrutura subjacente é do provedor.

---

## Alternativas Consideradas e Rejeitadas

### Alternativa 1 — IaaS com EC2 + Auto Scaling

**Descrição:** Implantar os microsserviços em instâncias EC2 com Auto Scaling Groups e balanceador de carga (ALB).

**Motivo da rejeição:**
- Latência de escala horizontal (2–5 minutos para novas instâncias) incompatível com picos súbitos de transações financeiras
- Custo fixo de instâncias ociosas em períodos de baixa demanda
- Alta complexidade operacional: patching de SO, gerenciamento de capacidade, configuração de rede
- Maior superfície de ataque (SO exposto, portas abertas)

### Alternativa 2 — PaaS com Containers (ECS Fargate / EKS)

**Descrição:** Empacotar os serviços em containers Docker e orquestrar com ECS Fargate ou EKS.

**Motivo da rejeição:**
- ECS Fargate oferece boa abstração de infraestrutura, mas o tempo de cold start de containers (10–30 segundos) é inaceitável para transações financeiras em horários de pico após períodos de inatividade
- EKS adiciona complexidade operacional de cluster Kubernetes não justificada para o tamanho atual da equipe
- O custo é proporcional ao tempo de execução dos tasks, não ao uso efetivo — penaliza workloads com baixa utilização noturna

**Nota:** Containers podem ser revisitados na Fase 5, quando o volume justificar workloads de longa duração (ex: processamento batch de relatórios regulatórios).

### Alternativa 3 — Multicloud (AWS + GCP)

**Descrição:** Distribuir serviços entre dois provedores para evitar lock-in.

**Motivo da rejeição:**
- Aumenta drasticamente a complexidade operacional, de networking e de segurança
- Exige equipe com expertise em dois ecossistemas distintos
- Os trade-offs de lock-in não justificam o overhead para o estágio atual do produto. O risco de lock-in é mitigado via interfaces abstratas no código de domínio (sem chamadas diretas ao SDK nos serviços de negócio)

---

## Trade-offs Aceitos

| Trade-off | Descrição | Mitigação |
|-----------|-----------|-----------|
| **Cold Start** | Funções Lambda podem ter latência inicial de 100–500ms após períodos de inatividade | Provisioned Concurrency nas funções críticas (Wallet, Auth) |
| **Tempo máximo de execução** | Lambda tem limite de 15 minutos por execução | Processos longos (relatórios, batch) serão tratados com Step Functions ou ECS Fargate quando necessário |
| **Vendor Lock-in** | Forte dependência de serviços proprietários AWS (DynamoDB, EventBridge, Cognito) | Abstrações via interfaces; migração possível mas custosa |
| **Debugging distribuído** | Rastreamento de chamadas entre Lambdas é mais complexo que monolitos | AWS X-Ray para rastreamento distribuído; structured logging com correlation IDs |

---

## Consequências

- **Positivas:** Escalabilidade automática, custo proporcional ao uso, menor carga operacional, conformidade facilitada com PCI DSS
- **Negativas:** Risco de cold start em funções não provisionadas, complexidade de debugging distribuído, dependência do ecossistema AWS
- **Neutras:** A equipe precisará desenvolver competência em desenvolvimento e testes de funções Lambda localmente (AWS SAM)

---

## Referências

- Pressman, R. S. (2021). *Engenharia de Software: Uma Abordagem Profissional*. McGraw Hill. Cap. 14 (Estilos Arquiteturais).
- Richards, M., & Ford, N. (2020). *Fundamentals of Software Architecture*. O'Reilly Media. Cap. 17 (Microservices Architecture).
- Richardson, C. (2018). *Microservices Patterns*. Manning Publications.
- AWS Well-Architected Framework — Performance Efficiency Pillar. Amazon Web Services, 2023.
- Resolução BACEN 4.658/2018 — Segurança Cibernética para Instituições Financeiras.
