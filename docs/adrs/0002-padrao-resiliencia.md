# ADR-0002 — Padrões de Resiliência

**Data:** Junho/2025  
**Status:** Aceito  
**Deciders:** Equipe FinTech Wallet  
**Projeto:** FinTech Wallet — Fase 3 (Cloud & Microsserviços)

---

## Contexto

Em uma plataforma financeira, falhas não são uma eventualidade — são uma certeza. O sistema da FinTech Wallet depende de integrações com serviços externos (bureau de crédito, PIX/BACEN, motor antifraude) que podem ficar indisponíveis, lentos ou retornar erros transitórios. Internamente, as funções Lambda também podem falhar por timeouts, erros de rede ou limites de concorrência.

O risco central é o **efeito cascata**: uma falha no bureau de crédito não deve derrubar o serviço de pagamentos; uma lentidão no motor antifraude não deve bloquear transferências urgentes. Da mesma forma, uma transação financeira duplicada — causada por um retry sem controle — pode resultar em débito duplicado, o que representa prejuízo direto ao usuário e risco regulatório grave.

O sistema precisa, portanto, de um conjunto coeso de **padrões de resiliência** que garanta: (1) isolamento de falhas, (2) degradação elegante, e (3) consistência das transações mesmo em cenários de falha parcial.

---

## Decisão

Adotar a seguinte combinação de padrões de resiliência:

1. **API Gateway** como camada de proteção da borda (throttling, WAF, timeout controlado)
2. **Circuit Breaker** para chamadas a serviços externos (bureau, antifraude)
3. **Idempotência** com chave de idempotência obrigatória para todas as operações de escrita financeira
4. **Retry com Exponential Backoff** para falhas transitórias em SQS e chamadas externas
5. **Bulkhead** lógico via separação de filas SQS por domínio (auditoria vs. notificação)

---

## Justificativa

### 1. API Gateway como camada de resiliência da borda

O Amazon API Gateway atua como a primeira linha de defesa. Conforme Newman (2019), um gateway de API bem configurado previne que picos de tráfego externos propaguem-se para os serviços internos. As configurações implementadas são:

- **Throttling**: limite de 1.000 req/s por rota por padrão, configurável por endpoint (ex: rota de crédito limitada a 100 req/s para evitar abuso)
- **Timeout**: timeout máximo de 29 segundos no API Gateway — requisições que excedam este limite são rejeitadas com 504, evitando que conexões pendentes consumam concorrência do Lambda
- **WAF (Web Application Firewall)**: proteção contra ataques de injeção, DDoS Layer 7 e regras específicas do PCI DSS

Esse padrão implementa o conceito de **fail-fast na borda**: rejeitar cedo é melhor que deixar a requisição degradar o sistema internamente.

### 2. Circuit Breaker para integrações externas

Richards e Ford (2020) e Nygard (2018) descrevem o Circuit Breaker como essencial para arquiteturas distribuídas com dependências externas. O padrão funciona em três estados:

- **Fechado (normal)**: requisições passam normalmente
- **Aberto (falha detectada)**: requisições são bloqueadas imediatamente, retornando um fallback sem chamar o serviço falho
- **Semi-aberto (sondagem)**: após um tempo de espera, uma requisição de teste verifica se o serviço externo se recuperou

**Implementação na FinTech Wallet:**

Para o **bureau de crédito**: se 5 falhas consecutivas ou taxa de erro > 50% em uma janela de 60 segundos forem detectadas, o circuito abre. O fallback consiste em usar o **score interno** (histórico transacional do usuário na própria plataforma) para decisões de microcrédito até R$ 200,00. Acima deste valor, a solicitação é enfileirada para reavaliação quando o circuito fechar.

Para o **motor antifraude**: circuito abre com 3 falhas consecutivas. Fallback: aprovação conservadora — transações acima de R$ 500,00 são retidas em fila de revisão manual; abaixo disso, procedem com flag de "revisão pendente" registrado na auditoria.

Essa abordagem segue o princípio de **degradação elegante** (graceful degradation): o sistema continua funcionando com capacidade reduzida, ao invés de falhar completamente.

### 3. Idempotência obrigatória para operações financeiras

Em sistemas distribuídos, **retries são inevitáveis**. Um pagamento de R$ 200,00 que falha com timeout pode já ter sido processado no backend — se o cliente retentar sem controle, o débito ocorre duas vezes.

Conforme descrito por Kleppmann (2017) em *Designing Data-Intensive Applications*, a **idempotência** garante que a mesma operação, executada múltiplas vezes, produz o mesmo resultado que executada uma única vez.

**Implementação:**

Toda requisição de escrita financeira (pagamentos, transferências, solicitações de crédito) deve incluir um `idempotencyKey` no header (`X-Idempotency-Key: <uuid-v4>`). O Wallet Lambda armazena este identificador em uma tabela DynamoDB com TTL de 24 horas. Antes de processar qualquer operação, verifica-se se a chave já existe:

- Se **não existe**: processa e armazena o resultado
- Se **existe**: retorna o resultado armazenado sem reprocessar

O API Gateway também suporta idempotência nativa para algumas operações — usada como camada adicional.

### 4. Retry com Exponential Backoff e Jitter

Para falhas transitórias (erros de rede, timeouts, throttling do DynamoDB), aplica-se retry com **exponential backoff com jitter**, conforme recomendado pelo AWS Well-Architected Framework.

A fórmula aplicada é:

```
wait = min(cap, base * 2^attempt) + random(0, jitter)
```

Com `base = 100ms`, `cap = 30s` e `jitter = 500ms`. Isso evita o fenômeno de **thundering herd**: múltiplos clientes retentando simultaneamente após uma falha, sobrecarregando o serviço no momento exato de sua recuperação.

### 5. Bulkhead via isolamento de filas SQS

O padrão **Bulkhead** (anteparo de navio), descrito por Nygard (2018), propõe isolar recursos para que a falha em um componente não consuma os recursos de outro.

**Implementação:**

- Fila SQS FIFO exclusiva para auditoria: garantia de ordem e entrega exatamente uma vez (`exactly-once processing`), crítica para trilha de auditoria regulatória
- Fila SQS Standard exclusiva para notificações: alta throughput, tolera duplicatas (o sistema de notificação é idempotente por design)
- Lambda de auditoria e Lambda de notificação têm concorrências reservadas separadas (`reserved concurrency`): uma pico de notificações não pode esgotar a concorrência disponível para o serviço de auditoria

---

## Alternativas Consideradas e Rejeitadas

### Alternativa 1 — Retry simples (sem circuit breaker)

**Descrição:** Retentar chamadas com backoff fixo, sem monitorar a saúde do serviço externo.

**Motivo da rejeição:**
- Em falhas prolongadas (bureau fora por 10 minutos), retries contínuos criam **thundering herd** contra o serviço na recuperação
- Sem estado do circuito, não há como diferenciar falha transitória de falha permanente
- Aumenta latência percebida pelo usuário: requisição fica aguardando retries enquanto poderia receber fallback imediato

### Alternativa 2 — Saga Pattern para consistência distribuída

**Descrição:** Usar o padrão Saga para coordenar transações que envolvem múltiplos serviços, com compensações em caso de falha.

**Motivo da rejeição:**
- Saga (orquestrado ou coreografado) é adequado para fluxos de longa duração com múltiplos participantes. Na FinTech Wallet, a maioria das transações envolve no máximo 2 serviços, tornando a complexidade do Saga desproporcionai
- A idempotência + transações condicionais do DynamoDB resolve o problema de consistência de forma mais simples para o escopo atual
- Saga será revisitado se o fluxo de crédito evoluir para envolver múltiplos serviços (ex: seguro atrelado ao crédito, cobrança separada)

### Alternativa 3 — Service Mesh (AWS App Mesh / Istio)

**Descrição:** Usar um service mesh para gerenciar circuit breaking, retry e observabilidade na camada de infraestrutura.

**Motivo da rejeição:**
- Service mesh faz mais sentido em arquiteturas baseadas em containers (EKS). No modelo Serverless com Lambda, a comunicação não ocorre via rede entre serviços em execução contínua — cada invocação é efêmera
- AWS X-Ray e CloudWatch cobrem as necessidades de observabilidade sem a complexidade operacional de um service mesh

---

## Trade-offs Aceitos

| Trade-off | Descrição | Mitigação |
|-----------|-----------|-----------|
| **Complexidade de implementação** | Circuit Breaker e idempotência adicionam código e infraestrutura | Biblioteca `aws-lambda-powertools` encapsula boa parte da complexidade |
| **Latência adicional** | Verificação de idempotência adiciona uma leitura ao DynamoDB por operação | Leitura com `ConsistentRead=false` + DAX cache minimiza impacto (~1ms) |
| **Falsos positivos no Circuit Breaker** | Em condições de rede instável, o circuito pode abrir prematuramente | Tuning cuidadoso dos limiares; alertas no CloudWatch ao abrir |
| **Capacidade reduzida no fallback** | Quando o bureau está fora, crédito acima de R$ 200 é bloqueado | Comunicação proativa ao usuário; estimativa de tempo de recuperação via status page |

---

## Consequências

- **Positivas:** Sistema tolerante a falhas externas, proteção contra transações duplicadas, isolamento de domínios evita cascata de falhas
- **Negativas:** Maior complexidade de código e de testes; necessidade de monitoramento ativo dos estados dos circuit breakers
- **Neutras:** A equipe precisará estabelecer procedimentos de operação para situações de circuito aberto e fallback ativo

---

## Referências

- Pressman, R. S. (2021). *Engenharia de Software: Uma Abordagem Profissional*. McGraw Hill.
- Richards, M., & Ford, N. (2020). *Fundamentals of Software Architecture*. O'Reilly Media. Cap. 9 (Foundations).
- Newman, S. (2019). *Building Microservices, 2nd ed*. O'Reilly Media. Cap. 12 (Resiliency).
- Nygard, M. T. (2018). *Release It! Design and Deploy Production-Ready Software, 2nd ed*. Pragmatic Bookshelf. Cap. 5 (Stability Patterns).
- Kleppmann, M. (2017). *Designing Data-Intensive Applications*. O'Reilly Media. Cap. 9 (Consistency and Consensus).
- AWS Well-Architected Framework — Reliability Pillar. Amazon Web Services, 2023.
