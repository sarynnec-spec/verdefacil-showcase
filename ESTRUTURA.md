# Estrutura do repositório

Gerado a partir dos **ficheiros versionados** do repositório privado (`git ls-files`),
não de uma listagem do disco. 455 ficheiros de código, excluindo dependências e
ferramentas de desenvolvimento.

```
verdefacil/
├── src/
│   ├── app/              194   App Router: interface + API
│   ├── lib/               72   domínio e infraestrutura
│   ├── components/        56   componentes de interface
│   ├── actions/           19   server actions
│   ├── workers/            6   processamento em fila
│   ├── types/              4
│   └── data/               4
├── prisma/
│   ├── schema.prisma           28 modelos · 18 enums · 45 relações · 35 índices
│   ├── migrations/        12   migrações ativas
│   └── migrations-archive/ 25  histórico consolidado
├── supabase/                   RLS, políticas e configuração local
├── tests/
│   ├── certification/     20   invariantes fiscais e de autorização
│   ├── unit/               8   cálculo fiscal, SAF-T, hash, ATCUD
│   └── e2e/                6   Playwright
├── scripts/                7   verificações de pré-deploy e seed
└── .github/workflows/      4   CI, smoke tests, monitorização, watchdog
```

---

## `src/lib` — domínio e infraestrutura (72 módulos)

O domínio fiscal é o núcleo do produto e está isolado do HTTP. Os provedores seguem
o padrão *Strategy* por trás de uma interface e de uma factory, o que permite trocar
o canal de comunicação com a Autoridade Tributária sem tocar nas rotas.

```
fiscal/                              20   núcleo do domínio
  at/at-soap.client.ts                    cliente SOAP da Autoridade Tributária
  providers/fiscal-provider.interface.ts  contrato comum
  providers/fiscal-provider.factory.ts    seleção por configuração da organização
  providers/at-direct.provider.ts         comunicação direta com a AT
  providers/invoicexpress.provider.ts     via InvoiceXpress
  providers/moloni.provider.ts            via Moloni
  providers/noop.provider.ts              sem comunicação (rascunhos, testes)
  atcud.ts                                código único de documento
  hash.ts                                 cadeia de assinatura RSA entre documentos
  saft.ts · saft-builder.ts               exportação SAF-T (XML)
  series.ts                               séries e numeração sequencial
  calculations.ts                         IVA, retenção IRS, Segurança Social
  fiscal-year.ts                          constantes do ano fiscal
  threshold-status.ts · threshold-forecast.ts   limiares de isenção
  cashflow-forecast.ts                    previsão de tesouraria
  validation.ts                           NIF, IBAN, códigos postais
  qrcode.ts                               QR Code fiscal obrigatório
  secrets.ts                              cifra de credenciais fiscais

auth/                                10   autenticação e autorização
  access-core.ts                          núcleo de verificação de acesso
  capabilities.ts                         capacidades por papel (CBAC)
  active-org.ts · org-account.ts          contexto multi-organização
  session.ts                              sessões
  access-log.ts                           registo de decisões de acesso
  supabase.ts · .server.ts · .browser.ts  clientes por ambiente
  client.ts

voice-invoice/                        7   faturação por voz e mensagem
  transcribe.ts                           áudio → texto (Whisper / Gemini)
  parser.ts                               texto → dados estruturados de fatura
  invoice-creator.ts                      criação a partir do resultado
  whatsapp.ts · telegram.ts               canais de entrada
  session.ts · access.ts                  estado da conversa e permissões

db/                                   6   acesso a dados
  index.ts                                cliente Prisma
  org-context.ts                          contexto de tenant para o RLS
  repositories/invoice.repository.ts
  repositories/client.repository.ts
  repositories/series.repository.ts
  repositories/audit.repository.ts

api/                                  4   API pública
  v1-auth.ts                              autenticação por chave de API
  api-keys.ts                             emissão e revogação
  webhook-dispatch.ts                     envio de eventos aos clientes
  webhook-url-validator.ts                validação do destino (SSRF)

payments/        easypay.ts · record-payment.ts
pdf/             invoice-pdf.ts · pdf-utils.ts
mfa/             totp.ts · constants.ts
email/           send.ts · admin-recipient.ts
expenses/        ocr-validation.ts        validação do upload antes de ir para a IA
notifications/   fiscal-deadlines.ts
audit/           logger.ts
tvde/            calculator.ts
fleet/           csv-parser.ts
invoices/        public-token.ts
professions/     config.ts
health/          checks.ts
utils/           format.ts · slug.ts

(raiz)           rate-limit.ts · cache.ts · cors.ts · logger.ts · sentry.ts
                 constants.ts · utils.ts
```

---

## `src/app/api` — 51 rotas REST

### API pública versionada — contrato em OpenAPI

```
/api/v1/openapi.json              especificação publicada
/api/v1/clients                   listar e criar clientes
/api/v1/invoices                  listar e emitir faturas
/api/v1/invoices/[id]             obter
/api/v1/invoices/[id]/cancel      cancelar
/api/v1/invoices/[id]/pdf         descarregar PDF
/api/v1/saft                      pedir exportação
/api/v1/saft/[id]                 obter resultado
/api/v1/webhooks                  registar e listar subscrições
/api/v1/webhooks/[id]             alterar e remover
```

### Trabalho agendado — 8 cron jobs

```
/api/cron/watchdog                  diário 06:00   verifica BD, Redis, SAF-T e erros fiscais
/api/cron/overdue-digest            diário 08:00   resumo de faturas em atraso
/api/cron/fiscal-reminders          diário 09:00   prazos fiscais
/api/cron/trial-expiring            diário 10:00   fim do período experimental
/api/cron/recurring-invoices        diário 07:00   emissão de faturas recorrentes
/api/cron/admin-compliance-alerts   mensal         limiares legais (RGPD, DPO)
/api/cron/gdpr-cleanup              mensal         expurgo de dados
/api/cron/fiscal-year-review        anual 15/12    revisão das constantes fiscais
```

### Webhooks e integrações recebidas

```
/api/webhooks/stripe              subscrições e pagamentos
/api/webhooks/easypay             Multibanco e MB WAY
/api/telegram/webhook             faturação por mensagem
/api/whatsapp/webhook             faturação por mensagem
/api/vies/validate                validação de NIF intracomunitário
```

### IA aplicada

```
/api/ia/ask                       assistente fiscal (cadeia Gemini→Groq→OpenRouter→Claude)
/api/chat/public                  assistente público, rate limit por IP
/api/expenses/ocr                 fotografia de recibo → campos estruturados
```

### Restantes

```
/api/invoices · /api/invoices/[id] · /api/pdf/[invoiceId] · /api/preview
/api/saft · /api/saft/batch · /api/saft/download
/api/payments/mbway · /api/payments/multibanco
/api/stripe/create-checkout-session · create-portal-session · upgrade-subscription
/api/irs/modelo3 · /api/tvde/export · /api/dashboard/summary · /api/notifications
/api/account/export-data          portabilidade de dados (RGPD)
/api/audit/verify                 verificação de integridade do registo de auditoria
/api/auth/logout · /api/auth/post-login · /api/profile/avatar
/api/email/send-receipt · /api/waitlist · /api/health · /api/csp-report
```

---

## `src/actions` — 19 server actions

Escrita com validação Zod. Os schemas são partilhados com os formulários, pelo que
cliente e servidor validam pela mesma definição.

```
invoice.actions.ts + invoice.schemas.ts         emissão, cancelamento, rascunhos
client.actions.ts + client.schemas.ts           clientes
organization.actions.ts + organization.schemas.ts
delegation.actions.ts        convites e capacidades de contabilista
apikey.actions.ts            chaves da API pública
mfa.actions.ts               ativação e verificação TOTP
sessions.actions.ts          sessões ativas e revogação
recurring.actions.ts         faturação recorrente
expense.actions.ts           despesas
series.actions.ts            séries de numeração
voice-invoice.actions.ts     faturação por voz
public-payment.actions.ts    pagamento por link público
account.actions.ts           conta e cancelamento
org-switch.actions.ts        troca de organização ativa
fleet.actions.ts · tvde.actions.ts
```

---

## `src/workers` — processamento em fila (BullMQ + Redis)

```
bootstrap.ts        arranque dos consumidores
queues.ts           definição das filas
pdf.worker.ts       geração de PDF
saft.worker.ts      exportação SAF-T
fiscal.worker.ts    comunicação com a Autoridade Tributária
email.worker.ts     envio de email
```

---

## `tests` — 199 testes

### Certificação (20 suites) — invariantes, não implementação

Correm contra um PostgreSQL real e efémero (PGlite).

```
numbering/emission-invariants.test.ts      nunca dois documentos com o mesmo número;
                                           apagar rascunho não deixa buraco
numbering/cr5-cr7-real-flow.test.ts        fluxo fiscal completo
numbering/concurrency.test.ts              emissão concorrente
hash-chain/hash-invariants.test.ts         cadeia segue a ordem de emissão
signature/cr6-integration.test.ts          assinatura RSA == declarado no SAF-T
signature/cr6-systementrydate.test.ts
signature/systementrydate-persistence.test.ts
signature/voice-systementrydate.test.ts
auth/access-core.test.ts                   núcleo de autorização
auth/delegation-access-matrix.test.ts      matriz de 5 perfis × capacidades
auth/delegation-capability.test.ts
auth/delegation-invite.test.ts
auth/api-key.test.ts
auth/member-management.test.ts
auth/org-account.test.ts
auth/account-state.test.ts                 conta bloqueada, organização cancelada
auth/voice.test.ts
stripe/stripe-reliability.test.ts          idempotência de webhooks
easypay/easypay-reliability.test.ts
harness.smoke.test.ts
```

### Unitários (8) e E2E (6)

```
unit/fiscal/      calculations · saft · hash · atcud · validation
                  at-soap · cashflow-forecast · expense-ocr-validation
e2e/              auth · invoice · expenses · mfa · recurring · security-regression
```

---

## `prisma/migrations` — evolução da estrutura

12 migrações ativas, consolidadas a partir de 25 arquivadas. A sequência mostra como
o modelo evoluiu:

```
0_init
add_invoice_recurring_link              faturação recorrente
add_recurring_pending_approval          aprovação antes de emitir
add_invoice_public_payment_link         pagamento por link
add_accountant_auto_forward             encaminhamento para o contabilista
add_expenses                            despesas
force_rls_phase5                        FORCE RLS no isolamento de tenant
sprint2_delegation_security             endurecimento da delegação
add_can_record_payments                 nova capacidade
add_invoice_system_entry_date           data de entrada no sistema (requisito AT)
draft_no_fiscal_identity                rascunhos sem identidade fiscal
delegation_can_issue_invoices           nova capacidade
```

Do arquivo, as que importam para segurança e desempenho:

```
performance_indexes · trgm_indexes            índices, incluindo trigramas para pesquisa
audit_logs_immutable                          auditoria imutável
revoke_audit_logs_mutation                    privilégios de alteração revogados na BD
rls_policies · force_rls                      Row Level Security
phase5_tenant_isolation_policies              isolamento entre organizações
lock_down_session_definer_functions           funções SECURITY DEFINER restringidas
rls_system_sentinel                           sentinela de contexto de sistema
api_keys_webhooks                             API pública e webhooks
```
