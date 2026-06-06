# ADR-0003 — Modelo de Comunicação entre Serviços

**Data:** Junho/2025  
**Status:** Aceito  
**Deciders:** Equipe FinTech Wallet  
**Projeto:** FinTech Wallet — Fase 3 (Cloud & Microsserviços)

---

## Contexto

Em uma arquitetura distribuída como a da FinTech Wallet, os serviços precisam se comunicar entre si e com sistemas externos. A decisão sobre **quando usar comunicação síncrona e quando usar comunicação assíncrona** é uma das mais impactantes na arquitetura: ela afeta diretamente latência, consistência, acoplamento entre serviços, resiliência e complexidade de desenvolvimento.

Dois perfis de operação coexistem nesta plataforma:

1. **Operações transacionais de missão crítica**: pagamento P2P, transferência PIX, aprovação de microcrédito. O usuário aguarda o resultado em tempo real. Requerem resposta rápida e consistente.

2. **Operações de suporte e registro**: auditoria de operações, envio de notificações push/SMS/e-mail, geração de relatórios. O usuário não precisa aguardar — o resultado pode ser processado em segundo plano.

Tratar esses dois perfis com o mesmo modelo de comunicação seria um erro de design: tornar pagamentos assíncronos aumentaria a percepção de insegurança pelo usuário; tornar auditoria síncrona acoplaria o caminho crítico de pagamentos à disponibilidade do serviço de logs.

---

## Decisão

Adotar um **modelo híbrido de comunicação**, com regras claras de aplicação:

**Comunicação Síncrona (Request/Response via Lambda Invoke direto ou HTTP):**
- Transações financeiras (pagamento, transferência, saque)
- Aprovação de microcrédito (solicitação → resposta com resultado)
- Autenticação e validação de sessão
- Consulta de saldo e extrato
- Integração com PIX/BACEN e motor antifraude

**Comunicação Assíncrona (Amazon EventBridge + SQS):**
- Registro de auditoria após cada operação financeira
- Envio de notificações ao usuário (push, SMS, e-mail)
- Publicação de eventos de domínio para sistemas analíticos futuros
- Reprocessamento de crédito em fila quando bureau está indisponível

---

## Justificativa

### Por que síncrono para transações financeiras?

Richardson (2018) define que a comunicação síncrona é adequada quando: (1) o chamador precisa do resultado para continuar, (2) o tempo de resposta é crítico para a experiência do usuário, e (3) a consistência imediata é um requisito de negócio.

As três condições se aplicam integralmente às transações da FinTech Wallet:

**Consistência**: Um pagamento de R$ 300,00 deve ser confirmado ou recusado de forma definitiva antes de retornar ao usuário. Uma resposta assíncrona criaria um estado intermediário ("pagamento pendente") que viola as expectativas do usuário e complica a consistência do saldo. O protocolo PIX do BACEN, por design, também opera de forma síncrona — a confirmação da liquidação é parte do fluxo de resposta.

**Latência e UX**: Conforme estudos de usabilidade citados por Pressman (2021), usuários de aplicações financeiras toleram no máximo 3 segundos de espera para operações de pagamento antes de perceber o sistema como lento ou inseguro. Um modelo assíncrono com polling ou webhook adiciona complexidade ao cliente e piora a percepção de responsividade.

**Rastreabilidade**: No fluxo síncrono, o `correlation_id` percorre toda a cadeia de chamadas, permitindo rastreamento distribuído via AWS X-Ray de ponta a ponta em uma única requisição.

### Por que assíncrono para auditoria e notificações?

Richards e Ford (2020) estabelecem que o acoplamento temporal — quando dois serviços precisam estar disponíveis simultaneamente para que uma operação seja bem-sucedida — é uma das formas mais prejudiciais de acoplamento em microsserviços.

Se a auditoria fosse síncrona no caminho crítico de pagamento, uma falha ou lentidão no serviço de auditoria atrasaria ou bloquearia pagamentos — o que é inaceitável. O pagamento já foi confirmado ao usuário; o registro de auditoria deve ocorrer, mas pode ocorrer **depois**, com garantia de entrega.

O uso do **SQS FIFO** para a fila de auditoria garante:
- **Ordered delivery**: eventos são processados na ordem em que ocorreram — essencial para reconstrução de trilha de auditoria
- **Exactly-once processing**: sem duplicatas na trilha de auditoria, o que é crítico para conformidade regulatória
- **Durabilidade**: mensagens persistidas por até 14 dias; em caso de falha do consumidor, o evento não se perde

Para notificações, o **SQS Standard** é suficiente: a ordem não é crítica (receber "pagamento recebido" 2 segundos antes ou depois é aceitável), e o sistema de notificação é idempotente — uma notificação duplicada é inofensiva.

### EventBridge como barramento de domínio

O Amazon EventBridge atua como barramento de eventos de domínio. Cada serviço publica eventos sem conhecer os consumidores — o Wallet Service publica `PaymentCompleted`; quem deve reagir a isso (auditoria, notificação, analytics) subscreve a regra correspondente. Esse desacoplamento segue o padrão **Publisher-Subscriber** descrito por Hohpe e Woolf (2003) em *Enterprise Integration Patterns*.

A principal vantagem é a **extensibilidade**: adicionar um novo consumidor de eventos (ex: um serviço de cashback, um agregador de relatórios) não requer modificar o serviço publicador — zero acoplamento de código.

---

## Alternativas Consideradas e Rejeitadas

### Alternativa 1 — Comunicação assíncrona para tudo (Full Event-Driven)

**Descrição:** Todas as operações, inclusive transações financeiras, operam via eventos. O cliente envia uma solicitação, recebe um `requestId`, e consulta o resultado via polling ou recebe via webhook.

**Motivo da rejeição:**
- Viola a expectativa do usuário de confirmação imediata em pagamentos financeiros
- Complexidade de implementação no cliente mobile (polling vs. WebSocket vs. webhook)
- O protocolo PIX do BACEN é síncrono por especificação — a integração exigiria uma camada de adaptação complexa
- Em caso de falha no processamento assíncrono, o usuário fica em estado indeterminado — especialmente problemático para débitos

Newman (2019) alerta: *"Evite comunicação assíncrona quando o chamador precisa do resultado para tomar uma decisão ou quando a garantia de entrega imediata é parte do contrato com o usuário."*

### Alternativa 2 — gRPC para comunicação interna entre Lambdas

**Descrição:** Usar gRPC (Protocol Buffers) para comunicação entre serviços, em vez de invocação direta de Lambda.

**Motivo da rejeição:**
- Lambdas são invocadas por eventos ou via API Gateway — não mantêm conexões persistentes de rede, o que é requisito do modelo gRPC
- A comunicação Lambda-to-Lambda é mais eficiente via invocação direta do SDK AWS (`InvokeFunction`), que evita latência de rede adicional
- gRPC faz mais sentido em arquiteturas de containers com serviços de longa duração
- A complexidade de serialização/deserialização com Protocol Buffers não traz benefício proporcional no contexto atual

### Alternativa 3 — Apache Kafka para mensageria

**Descrição:** Substituir SQS/EventBridge por Amazon MSK (Kafka gerenciado) para todas as comunicações assíncronas.

**Motivo da rejeição:**
- Kafka é adequado para streaming de eventos de alto volume com múltiplos consumidores e necessidade de reprocessamento de histórico — características não presentes no escopo atual
- MSK tem custo de operação significativamente maior que SQS, mesmo gerenciado
- A complexidade operacional de Kafka (consumer groups, offsets, particionamento) é desproporcional para filas com volume moderado
- SQS + EventBridge cobrem 100% dos casos de uso identificados com menor custo e complexidade

---

## Matriz de Decisão de Comunicação

| Operação | Tipo | Justificativa |
|----------|------|---------------|
| Pagamento P2P | Síncrono | Usuário aguarda confirmação; ACID necessário |
| Transferência PIX | Síncrono | Protocolo BACEN é síncrono; confirmação imediata |
| Aprovação de crédito | Síncrono | Usuário aguarda aprovação/recusa em tempo real |
| Consulta de saldo | Síncrono | Leitura simples; latência < 50ms esperada |
| Registro de auditoria | Assíncrono | Não bloqueia o caminho crítico; SQS FIFO garante ordem |
| Notificação push/SMS | Assíncrono | Não crítico para o fluxo financeiro; tolerante a atraso |
| Evento de domínio (analytics) | Assíncrono | Desacoplamento total; consumidores futuros |
| Consulta bureau de crédito | Síncrono + Fallback | Resultado imediato necessário; circuit breaker em caso de falha |

---

## Trade-offs Aceitos

| Trade-off | Descrição | Mitigação |
|-----------|-----------|-----------|
| **Complexidade do modelo híbrido** | Equipe precisa conhecer dois paradigmas e quando aplicar cada um | Documentação desta ADR + guidelines claros na documentação técnica do projeto |
| **Eventual consistency na auditoria** | Registro de auditoria pode chegar com delay de 1–2 segundos após a transação | Aceitável: auditoria não precisa ser consultada em tempo real. Janela de 2s é imperceptível para fins regulatórios |
| **Debugging de fluxos assíncronos** | Rastrear uma notificação que não chegou exige correlacionar logs de múltiplos serviços | `correlation_id` propagado em todos os eventos; CloudWatch Logs Insights para consulta centralizada |

---

## Consequências

- **Positivas:** Baixo acoplamento entre domínios, resiliência no caminho assíncrono, experiência de usuário otimizada para transações em tempo real, conformidade com o modelo síncrono do PIX
- **Negativas:** Necessidade de disciplina na equipe para seguir as regras do modelo híbrido; testes de integração precisam cobrir ambos os fluxos
- **Neutras:** Eventual futura migração para Event Sourcing completo no domínio de crédito seria facilitada pela base assíncrona já estabelecida

---

## Referências

- Pressman, R. S. (2021). *Engenharia de Software: Uma Abordagem Profissional*. McGraw Hill.
- Richards, M., & Ford, N. (2020). *Fundamentals of Software Architecture*. O'Reilly Media. Cap. 14 (Event-Driven Architecture).
- Richardson, C. (2018). *Microservices Patterns*. Manning Publications. Cap. 3 (Interprocess Communication).
- Newman, S. (2019). *Building Microservices, 2nd ed*. O'Reilly Media. Cap. 4 (Communication Styles).
- Hohpe, G., & Woolf, B. (2003). *Enterprise Integration Patterns*. Addison-Wesley.
- Kleppmann, M. (2017). *Designing Data-Intensive Applications*. O'Reilly Media. Cap. 11 (Stream Processing).
- Banco Central do Brasil. Manual de Padrões para Iniciação do PIX. Versão 2.8, 2023.
