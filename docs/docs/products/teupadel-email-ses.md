# teupadel: e-mail e SES

!!! info "Fonte de verdade / atualizar aqui"
    Runbook do e-mail transacional (Amazon SES, `us-east-1`) do teupadel.com. Atualizar (PT + EN) quando o
    estado do SES, o tratamento de bounces ou a política DMARC mudarem. Itens de acompanhamento em
    [backlog](teupadel-backlog.md) (6, 7 e 8).

## Estado atual (2026-09-29)

| Ponto | Estado |
|---|---|
| Modo | **Sandbox** |
| Acesso a produção | **Negado pela AWS em 2026-09-29** (case `179054498900338`). Reabrir em 2 a 3 semanas |
| Uso | Só e-mail transacional: o magic link de login (o login Google não envia e-mail) |
| Remetente | `TeuPadel <noreply@teupadel.com>` (`SES_SENDER`), configuration set `teupadel-transactional` |
| Domínio | `teupadel.com` verificado, com DKIM, SPF, DMARC e MAIL FROM próprio (segundo o texto enviado à AWS) |
| DMARC | `p=none` hoje; plano abaixo |
| Magic link em produção | **Nunca exercitado** (backlog 6) |

## Endereços verificados e testadores

No sandbox o SES só entrega para identidades verificadas. A API só envia o magic link para `@teupadel.com`
ou para identidades verificadas; qualquer outro pedido mantém a resposta `202` sem enviar e-mail
(anti-enumeração). Para autorizar um testador:

```bash
aws sesv2 create-email-identity --email-identity <email> --region us-east-1
# a pessoa clica no e-mail da AWS; depois confirmar:
aws sesv2 list-email-identities --region us-east-1
```

A lista atual de identidades não está registada aqui; consultar com o segundo comando.

## O que a API faz com bounces e reclamações

Fluxo: SES -> SNS -> fila SQS `teupadel-ses-events` -> worker `ses_events.py` (task de fundo da API, só
ativo se `SES_EVENTS_QUEUE_URL` estiver definida).

- Bounce **Permanent** e reclamação: gravados em `email_suppressions` (`ON CONFLICT DO NOTHING`).
- Bounces transitórios e `Delivery`: ignorados.
- Mensagem ilegível: apagada da fila. Erro de BD: a mensagem fica na fila (reentrega).
- Antes de qualquer envio, a API consulta `email_suppressions`; endereço suprimido não recebe e-mail.
- Com 2 réplicas ambas consomem a mesma fila; o SQS entrega cada mensagem a uma só.
- Precisa de `sqs:ReceiveMessage`/`DeleteMessage` nas credenciais AWS da API (IaC `iac-mail-routing`, módulo
  `aws-ses-smtp-user`).
- Outras defesas do envio: 5 pedidos/hora por IP em `/auth/magic-link`, `EMAIL_LINKS_PER_HOUR` (5) por
  e-mail, domínios descartáveis rejeitados, Turnstile no pedido.

## Plano DMARC e pós-aprovação

1. Depois de sair do sandbox, enviar um e-mail real a Gmail e Outlook e validar SPF, DKIM e DMARC nos
   cabeçalhos.
2. Configurar o envio como `contato@` (SMTP em `/teupadel/ses/gmail-smtp`).
3. Manter `p=none` e ler os relatórios agregados durante 2 a 4 semanas.
4. Com relatórios limpos, subir para `p=quarantine`.
5. Monitorizar a taxa de bounce (alerta > 2 %, ainda por criar, backlog 9) e de reclamações.
6. Testar a receção de `contato@`, `suporte@` e `privacidade@` (backlog 24).

## Reabrir o pedido de produção

Reabrir o case em 2 a 3 semanas, com o worker de bounces validado. Descrever o uso real: só transacional,
volume baixo, envio apenas a quem pede login, tratamento de bounces e reclamações. Atualizar o estado aqui
e no backlog com a data da nova resposta.

## Apêndice: texto para reutilizar no pedido

Texto enviado à AWS (em inglês, que é a língua do pedido). Ajustar antes de reenviar: a linha "Password
reset emails, if enabled" não se aplica hoje, porque não há login por senha, e convém citar o worker de
bounces já validado.

> Hello,
>
> Thank you for following up.
>
> TeuPadel is a web application that uses Amazon SES exclusively for transactional email. We are requesting production access for the us-east-1 region.
>
> Our initial email types will be:
>
> - Email verification links
> - Passwordless login (magic) links
> - Password reset emails, if enabled
> - Account and security notifications
>
> These emails are sent only after a user explicitly requests an action in the application. We do not send marketing campaigns, newsletters, cold email, or messages to purchased or third-party lists.
>
> At launch, we expect low volume, typically tens of emails per day, with occasional short bursts. We expect to remain well below the default SES quota.
>
> Our sending domain, teupadel.com, is verified in SES. DKIM, SPF, DMARC, and a custom MAIL FROM domain are configured. Emails will be sent from noreply@teupadel.com, with replies directed to our support/contact address.
>
> We have configured SES bounce, complaint, and delivery notifications through SNS and SQS. We will use these events to monitor delivery, suppress addresses that bounce or generate complaints, and prevent further sending to problematic recipients.
>
> We do not currently send marketing emails. If we introduce optional marketing communication in the future, it will require explicit opt-in, include an unsubscribe mechanism, and be managed separately from transactional messages.
>
> We will monitor bounce and complaint rates and investigate delivery issues promptly. We will also test the SES mailbox simulator before launching.
>
> Website:
> https://teupadel.com
>
> Please let us know if you need any additional information.
>
> Kind regards,
> Cassio Oliveira
> TeuPadel
