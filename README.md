# Blueprint de Arquitetura — Padrão Saga no E-commerce

## 1. Contexto de negócio
A plataforma de e-commerce está migrando de um monólito para microsserviços com bancos de dados independentes. O fluxo de compra envolve `Order Service`, `Inventory Service` e `Payment Service`. Como não existe uma transação ACID única entre os três bancos, a solução utiliza **consistência eventual** e o padrão **Saga** para concluir ou compensar a compra.

## 2. Decisão arquitetural: Saga por Orquestração
Foi escolhida a **Orquestração**, com um `Saga Orchestrator` responsável por coordenar etapas, persistir o estado do processo e decidir as compensações.

**Justificativas**
- **Fluxo explícito:** a sequência de pedido, reserva, pagamento e confirmação é centralizada e fácil de auditar.
- **Menor acoplamento temporal:** os serviços não precisam conhecer os eventos de negócio de todos os demais participantes; respondem a comandos e publicam resultados.
- **Tratamento de falhas mais simples:** o orquestrador controla retries, timeouts, compensações e estado terminal.
- **Escalabilidade:** orquestradores podem ser executados em múltiplas instâncias, usando persistência compartilhada e controle de concorrência.
- **Trade-off:** o orquestrador é um componente crítico e precisa de alta disponibilidade, persistência e observabilidade. Não deve ser um ponto único de falha.

A comunicação entre o orquestrador e os serviços é assíncrona via broker de mensagens. O fluxo de cada Saga é persistido; nenhuma transação distribuída é mantida aberta enquanto se aguarda uma resposta.

## 3. Participantes e responsabilidades
- **Order Service:** cria o pedido em `PENDING`, confirma como `CONFIRMED` ou cancela como `CANCELLED`.
- **Inventory Service:** reserva unidades com prazo de expiração e libera a reserva em compensação.
- **Payment Service:** autoriza/captura pagamento usando chave de idempotência; informa `PaymentAuthorized` ou `PaymentRejected`.
- **Saga Orchestrator:** mantém o estado da Saga, envia comandos, aplica timeout/retry e conduz compensações.
- **Message Broker:** transporta comandos e eventos de forma assíncrona.
- **Outbox/Inbox:** garante publicação confiável e deduplicação de mensagens.

## 4. Fluxo de sucesso
1. O cliente solicita a compra.
2. O Orchestrator cria a Saga e envia `CreatePendingOrder`.
3. O Order Service cria o pedido em estado `PENDING` e responde `OrderCreated`.
4. O Orchestrator envia `ReserveStock`.
5. O Inventory Service reserva o estoque e responde `StockReserved`.
6. O Orchestrator envia `AuthorizePayment`.
7. O Payment Service autoriza/captura o pagamento no gateway e responde `PaymentAuthorized`.
8. O Orchestrator envia `ConfirmOrder`.
9. O Order Service altera o pedido para `CONFIRMED` e responde `OrderConfirmed`.
10. A Saga termina em `COMPLETED` e o cliente recebe confirmação.

## 5. Fluxo de falha e compensação
Cenário: o pedido foi criado e o estoque reservado, mas o pagamento foi recusado.

1. O Payment Service responde `PaymentRejected`.
2. O Orchestrator marca a Saga como `COMPENSATING`.
3. Envia `ReleaseStock` ao Inventory Service e aguarda `StockReleased`.
4. Envia `CancelOrder` ao Order Service e aguarda `OrderCancelled`.
5. Registra a Saga como `COMPENSATED` e informa que a compra não foi concluída.

A compensação é uma nova transação de negócio, não um rollback ACID. Ela deve ser idempotente. Se a liberação de estoque falhar temporariamente, o orquestrador repete a operação; não marca a Saga como compensada antes das confirmações. Uma reserva também possui TTL como proteção adicional, mas a expiração não substitui a compensação explícita.

## 6. Contratos de comunicação
Cada mensagem inclui, no mínimo:
- `messageId`: identificador único da mensagem;
- `sagaId`: identificador da instância da Saga;
- `correlationId`: correlação com a solicitação original;
- `orderId`: identificador do pedido;
- `idempotencyKey`: chave estável para a operação de negócio;
- `occurredAt`, `schemaVersion` e `traceId`.

Exemplos de comandos: `CreatePendingOrder`, `ReserveStock`, `AuthorizePayment`, `ConfirmOrder`, `ReleaseStock`, `CancelOrder`.
Exemplos de resultados: `OrderCreated`, `StockReserved`, `PaymentAuthorized`, `PaymentRejected`, `OrderConfirmed`, `StockReleased`, `OrderCancelled`.

## 7. Estado da Saga
Estados sugeridos: `STARTED`, `ORDER_PENDING`, `STOCK_RESERVED`, `PAYMENT_PENDING`, `PAYMENT_REJECTED`, `COMPENSATING`, `COMPENSATED`, `COMPLETED`, `MANUAL_REVIEW`.

A tabela persistente de controle contém `saga_id`, `order_id`, `current_state`, `last_command`, `attempt_count`, `deadline_at`, `last_error`, `version` e timestamps. Atualizações usam controle otimista de versão. Estados terminais são `COMPLETED`, `COMPENSATED` e `MANUAL_REVIEW`.

## 8. Idempotência e timeouts
- Cada consumidor mantém uma Inbox/tabela de mensagens processadas com unicidade por `messageId` ou por chave de negócio.
- Operações mutáveis recebem uma chave idempotente; repetir `ReserveStock` não cria duas reservas e repetir `AuthorizePayment` não cobra duas vezes.
- Use Transactional Outbox para gravar mudança de estado e evento a publicar na mesma transação local.
- Em timeout, o resultado pode ser desconhecido: consultar o estado da operação ou reenviar com a mesma chave idempotente; nunca presumir que o serviço não executou.
- Retry com backoff exponencial e jitter para falhas transitórias; limite de tentativas e prazo máximo.
- Erros permanentes, mensagens inválidas ou tentativas esgotadas seguem para DLQ, com alerta e procedimento de replay controlado.
- Se o pagamento ficar ambíguo após timeout, consultar o gateway antes de compensar ou tentar nova cobrança. Isso evita liberar estoque enquanto um pagamento pode ter sido aprovado.
- Circuit Breaker evita sobrecarregar dependências indisponíveis; após recuperação, o tráfego é retomado gradualmente.

## 9. Observabilidade e segurança operacional
Propagar `traceId`, `correlationId`, `sagaId` e `orderId` nos logs. Monitorar idade das Sagas em andamento, taxa de compensação, retries, mensagens na DLQ, tempo de reserva e falhas do gateway. Criar alertas para Sagas presas em `COMPENSATING` ou `MANUAL_REVIEW`.

## 10. Diagramas
- [`diagrams/saga-architecture-overview.png`](diagrams/saga-architecture-overview.png) — visão geral.
- [`diagrams/sequence-success-flow.png`](diagrams/sequence-success-flow.png) — caminho de sucesso.
- [`diagrams/sequence-compensation-flow.png`](diagrams/sequence-compensation-flow.png) — recusa de pagamento e compensação.
- Os arquivos `.puml` correspondentes são fontes editáveis PlantUML.

## 11. Limites e riscos residuais
Saga não oferece isolamento global nem rollback instantâneo. Outros processos podem observar estados intermediários. Para reduzir risco, o pedido permanece `PENDING` até confirmação; reserva tem TTL; pagamento usa idempotência e consulta de status; compensações têm retry, DLQ e escalonamento operacional. Reembolsos após captura financeira podem exigir fluxo compensatório próprio e não devem ser tratados como se fossem um simples rollback.
