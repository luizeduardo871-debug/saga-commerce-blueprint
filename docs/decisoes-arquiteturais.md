# Decisões arquiteturais — Saga de compra

## DA-01 — Orquestração
**Decisão:** utilizar Saga orquestrada por um componente dedicado.

**Motivo:** o fluxo possui ordem definida e compensações dependentes do estado anterior. A coordenação explícita facilita auditoria, monitoramento, manutenção e tratamento uniforme de falhas. Coreografia reduziria o componente central, mas distribuiria a lógica de processo pelos consumidores, tornando o fluxo e as compensações mais difíceis de rastrear.

## DA-02 — Mensageria assíncrona
Comandos e resultados trafegam por broker. Os serviços não mantêm conexões síncronas abertas durante o processo. A entrega pode ser *at least once*, portanto consumidores precisam ser idempotentes.

## DA-03 — Persistência confiável
Cada serviço grava suas alterações e o registro Outbox na mesma transação local. Um publicador envia as mensagens da Outbox ao broker. Consumidores registram mensagens recebidas na Inbox com restrição de unicidade. Isso reduz perda de eventos e efeitos duplicados.

## DA-04 — Idempotência
| Operação | Chave/controle | Comportamento repetido |
|---|---|---|
| Criar pedido | `orderId` + `CreatePendingOrder` | Retorna o pedido já criado |
| Reservar estoque | `sagaId` + `orderId` + itens | Retorna a reserva existente; não decrementa novamente |
| Autorizar pagamento | `idempotencyKey` persistida no gateway/serviço | Retorna o resultado original; não cria nova cobrança |
| Confirmar pedido | `orderId` + estado esperado | Mantém `CONFIRMED` sem duplicar efeitos |
| Liberar estoque | `reservationId` | Libera uma única vez |
| Cancelar pedido | `orderId` | Mantém `CANCELLED` sem efeitos adicionais |

A idempotência deve ser garantida na camada de persistência, não apenas em memória. Uma restrição única evita corridas entre instâncias concorrentes.

## DA-05 — Timeouts e resultados ambíguos
Um timeout significa que a resposta não chegou, não que a operação falhou. O orquestrador persiste `deadline_at`, comando pendente e número de tentativas. Ao expirar:
1. consultar o estado no serviço participante, quando disponível;
2. reenviar o mesmo comando com a mesma chave idempotente;
3. aplicar backoff exponencial com jitter para falhas transitórias;
4. ao exceder o limite, encaminhar para DLQ/alerta ou `MANUAL_REVIEW`.

Para pagamento, verificar o estado junto ao Payment Service/gateway antes de disparar compensação. Uma autorização bem-sucedida com resposta perdida não pode ser confundida com recusa.

## DA-06 — Retry, Circuit Breaker e DLQ
- **Retry:** apenas para erros transitórios (indisponibilidade, throttling, falha temporária de rede); backoff exponencial, jitter e limite de tentativas.
- **Circuit Breaker:** abre após falhas consecutivas, evita chamadas contínuas a uma dependência indisponível e permite teste gradual de recuperação.
- **Dead-Letter Queue (DLQ):** recebe mensagens que falharam após as tentativas previstas ou que não podem ser processadas por erro de contrato. A mensagem deve manter metadados de correlação e motivo da falha.
- **Replay:** não reprocessar a DLQ indiscriminadamente. Corrigir causa raiz, validar estado atual da Saga, manter as chaves idempotentes e reexecutar com auditoria.
- **Poison messages:** mensagens incompatíveis com o schema são isoladas e geram alerta; não devem bloquear indefinidamente a fila principal.

## DA-07 — Compensação
A ordem de compensação é inversa à ordem das ações concluídas: liberar reserva e cancelar pedido. O orquestrador só marca `COMPENSATED` após receber as confirmações. Falhas de compensação são repetidas e, se persistirem, geram alerta e intervenção manual. Se uma operação financeira já foi capturada, pode ser necessário executar reembolso em vez de apenas cancelar autorização.

## DA-08 — Concorrência e recuperação
A tabela da Saga usa `version` para controle otimista e evita que duas instâncias avancem o mesmo processo simultaneamente. Ao reiniciar, o orquestrador retoma Sagas não terminais a partir do estado persistido. Locks distribuídos, se usados, devem ter lease e não substituir idempotência.

## DA-09 — Consistência eventual
O pedido fica `PENDING` enquanto a Saga está em andamento. O usuário recebe estados claros como “processando”, “confirmado” ou “não foi possível concluir”. A arquitetura não promete atomicidade global: durante o fluxo, serviços podem observar estados intermediários.

## DA-10 — Observabilidade
Propagar `sagaId`, `orderId`, `messageId`, `correlationId` e `traceId`. Métricas mínimas: duração da Saga, Sagas pendentes por idade, falhas de compensação, retries, taxa de pagamento recusado, tamanho/idade da DLQ e tempo de reserva. Alertar para `COMPENSATING` prolongado e `MANUAL_REVIEW`.
