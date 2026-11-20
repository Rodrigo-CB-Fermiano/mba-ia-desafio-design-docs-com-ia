# ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint

## Status

Aceito

- **Data:** decisão registrada na reunião técnica de quinta-feira (ver `TRANSCRICAO.md`); ADR consolidado em 20/11/2026
- **Decisores:** Sofia (Engenharia de Segurança), Larissa (Tech Lead), Diego (Engenharia Sênior)
- **Contexto da decisão:** reunião técnica do Sistema de Webhooks de Notificação de Pedidos

## Contexto

Os webhooks são **outbound**: a plataforma envia dados de pedidos para um endpoint HTTPS fora da infraestrutura da empresa. O cliente precisa conseguir validar que a requisição veio realmente da plataforma e que o payload não foi adulterado no caminho ([09:19] Sofia).

O padrão discutido foi HMAC: assinar o payload com uma secret compartilhada entre a plataforma e o cliente, enviar a assinatura em um header e o cliente verificar do lado dele ([09:20] Sofia). O algoritmo escolhido foi **HMAC-SHA256**, por ser padrão de mercado e ter biblioteca na maioria das stacks ([09:20] Sofia).

Também foi decidido que **cada endpoint de webhook do cliente tem uma secret única**, e não uma secret global da plataforma: se uma vazar, não vaza todas ([09:21] Sofia). A secret é rotacionável; ao rotacionar, a antiga continua válida por **24 horas em paralelo**, para o cliente migrar os sistemas dele, e depois morre ([09:21] Sofia). A motivação inclui um caso real de cliente que vazou secret em log de aplicação ([09:22] Diego).

Complementarmente, a URL do webhook deve ser **HTTPS obrigatório**; cadastro com `http` é recusado com erro de validação no schema Zod ([09:23] Sofia). O payload tem limite de **64 KB**, com erro caso ultrapasse; não há truncamento ([09:23]–[09:24] Sofia/Diego/Larissa).

## Decisão

1. Assinar o **corpo exato da requisição** com `HMAC-SHA256` usando a secret do endpoint.
2. Enviar a assinatura no header `X-Signature`.
3. Cada endpoint de webhook possui **uma secret própria**, gerada pela plataforma e devolvida ao cliente **apenas na criação** e na rotação ([09:31] Marcos).
4. A secret é **rotacionável** por endpoint. Na rotação:
   - a secret atual vira `previousSecret`;
   - `previousSecretExpiresAt` é definido para `agora + 24h`;
   - a nova secret passa a ser usada para assinar novas entregas;
   - o cliente pode validar a assinatura com a secret atual ou com a anterior durante a janela de 24h.
5. Após 24h, a secret anterior não é mais válida.
6. A URL do webhook deve usar `https://`; cadastros e edições com `http://` são rejeitados.
7. O payload máximo é de **64 KB** (65.536 bytes); se ultrapassar, o evento é rejeitado com erro, sem truncar.
8. Secrets e assinaturas **nunca** devem ser gravadas em log. O logger Pino deve ter os caminhos de redaction correspondentes.
9. A assinatura é calculada sobre o corpo **como será enviado**; o worker deve assinar a string serializada, não uma re-serialização posterior do objeto.
10. O header `X-Timestamp` acompanha o envio para o cliente poder detectar replay attack se quiser; a assinatura, conforme decidido, cobre o corpo do request ([09:44] Diego).

> **Detalhe de implementação a validar na revisão de segurança:** a reunião definiu o algoritmo e o header, mas não o encoding textual da assinatura. O FDD adota **hexadecimal minúsculo** e deve ser confirmado com Sofia antes do deploy. A janela de revisão de segurança de pelo menos 2 dias úteis já está prevista ([09:46] Sofia).

## Alternativas Consideradas

### Alternativa A — Secret global da plataforma

- **Trade-off que levou ao descarte:** se uma secret vazasse, todos os clientes e endpoints estariam comprometidos; o raio de explosão seria total ([09:21] Sofia).
- **Custo evitado:** gestão simplificada de uma única secret, ao preço de segurança inaceitável.

### Alternativa B — Apenas TLS, sem assinatura

- **Trade-off que levou ao descarte:** TLS protege o transporte, mas não permite ao cliente verificar a origem da requisição nem detectar adulteração caso a terminação TLS ou um proxy seja comprometido; a reunião exigiu validação de origem e integridade ([09:19] Sofia).
- **Custo evitado:** simplicidade para o cliente, ao preço de não atender ao requisito de segurança.

### Alternativa C — OAuth 2.0 / mTLS

- **Trade-off:** são mecanismos mais fortes, porém mais complexos de operar e de integrar para clientes B2B; o padrão de mercado para webhooks é HMAC com secret compartilhada ([09:20] Sofia).
- **Custo evitado:** complexidade de certificados e fluxos de autorização.

### Alternativa D — HMAC-SHA1 ou MD5

- **Trade-off:** algoritmos legados e inadequados para novas integrações; SHA-256 é o padrão de mercado e tem suporte amplo ([09:20] Sofia).
- **Custo evitado:** compatibilidade com stacks antigas, ao preço de segurança.

## Consequências

### Positivas

- **Autenticidade e integridade:** o cliente consegue verificar que a requisição veio da plataforma e que o corpo não foi alterado.
- **Raio de explosão limitado:** uma secret vazada afeta apenas um endpoint.
- **Rotação segura:** o grace period de 24h permite migração sem downtime para o cliente.
- **HTTPS obrigatório:** reduz interceptação e downgrade.
- **Limite de payload:** evita eventos anormalmente grandes e protege o cliente e a outbox.

### Negativas / trade-offs

- **Responsabilidade no cliente:** o cliente precisa implementar verificação de HMAC, guardar a secret com segurança e tratar a rotação.
- **Gestão de secrets:** a plataforma precisa armazenar e rotacionar secrets por endpoint; a secret precisa ser recuperável para assinar, não apenas hash.
- **Complexidade de rotação:** duas secrets válidas por 24h exigem controle de expiração e limpeza.
- **Replay detection parcial:** como a assinatura cobre apenas o corpo, a validação de `X-Timestamp` é uma proteção adicional do lado do cliente, não uma garantia da plataforma.
- **Encoding não decidido na reunião:** a representação textual da assinatura precisa ser confirmada na revisão de segurança.

## Referências

- `TRANSCRICAO.md` — [09:19] Sofia; [09:20] Sofia/Bruno; [09:21] Sofia/Bruno; [09:22] Sofia/Diego; [09:23] Sofia; [09:24] Diego/Larissa; [09:31] Marcos; [09:44] Diego; [09:46] Sofia
- `src/shared/logger/index.ts` — redaction de dados sensíveis
- `src/modules/orders/order.schemas.ts` — padrão de validação com Zod
- `src/shared/errors/http-errors.ts` — padrão de erros de validação
