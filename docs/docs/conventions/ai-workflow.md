# Fluxo de trabalho com IA

Como as sessões do Claude Code trabalham neste workspace: onde cada tipo de
conhecimento vive, o que é lido no início, o que é atualizado no fim e o que é
delegado aos agentes locais para poupar tokens.

!!! note "Fonte de verdade"
    As regras curtas que valem em toda sessão estão no `CLAUDE.md` da raiz de
    `cmoreira-dev/`. Esta página detalha o porquê e a manutenção. Mudou o fluxo?
    Atualize as duas.

## Onde cada coisa vive

| Tipo de conhecimento | Onde | Por quê |
|---|---|---|
| Regras que valem em **toda** sessão (como trabalhar, guardrails, roteamento para agentes locais) | `CLAUDE.md` da raiz | Carregado sempre; cada linha custa tokens em toda sessão |
| Essencial do repo (comandos, armadilhas com o motivo, convenções, acoplamento entre repos) | `CLAUDE.md` do repo | Carregado quando se trabalha naquele repo; substitui a raiz no seu escopo |
| Arquitetura, contratos de API, variáveis de ambiente, rotas, deploy, runbooks | Este site (`docs/docs/`) | Não precisa estar vivo em toda sessão; lido por pilar quando necessário |
| Pendências e status de trabalho em andamento | Páginas de backlog (`products/teupadel-backlog`, `apps/sara-backlog`) | Uma única fonte; sobrevive entre sessões e é versionada |
| Preferências e decisões do usuário entre sessões | Memória do Claude Code (`~/.claude/projects/.../memory`) | Não é derivável do código nem da documentação |
| Notas soltas na pasta de um produto (`teupadel.com/`, `sara/`) | Nenhum lugar: essas pastas não são repositórios | Arquivos ali não são versionados e se perdem |

## Protocolo de sessão

1. **Identificar o pilar** da tarefa (tabela do `CLAUDE.md` da raiz).
2. **Ler a página de docs do pilar** antes de mudar contratos, rotas, infraestrutura ou deploy.
3. **Ler o `CLAUDE.md` e o `README.md` do repo** (o do repo prevalece sobre o da raiz).
4. **Delegar o trabalho operacional barato** aos agentes locais (ver [Agentes locais](local-agents.md)).
5. **Ao terminar**: atualizar a página de docs (PT e EN) e o backlog, no mesmo trabalho.
   Se não existe página, criá-la ou registrar a lacuna explicitamente.

## Fluxo de entrega

Ramo de feature → PR → merge em `main` → GitHub Actions builda a imagem e envia ao ECR (OIDC) →
argocd-image-updater atualiza o tag no `gitops.<produto>` → Argo CD sincroniza sozinho.
A maioria dos repos não tem CI de PR (só build no push para `main`); por isso o build local ou
em Docker e os testes são a validação antes do merge. Detalhes: [Do push ao cluster](../cicd/deploy-flow.md).

## Guardrails

- Nunca commitar direto em `main` sem avisar; preferir ramo + PR.
- `terraform apply` / `terragrunt apply` e merges em `iac.homelab-live-infra` (que dispara o apply com
  aprovação) só com confirmação explícita.
- Sem `kubectl apply/patch/delete` para mudar estado de aplicações: a mudança vai para o repo GitOps.
- Bumps major do Renovate são projetos próprios: testar, não mergear em lote.
- Dependências acopladas (ex.: `onnxruntime-gpu` + tag `nvidia/cuda` + `numpy` + `opencv` do processor)
  sobem juntas.

## Delegação e economia de tokens

Coleta e leitura de saída bruta (logs, `kubectl`, status de Actions e Argo CD) e rascunhos de
docs e de código simples vão para o modelo local via MCP; só um resultado curto e estruturado
volta ao contexto do Claude. O modelo local pode errar e não age: o que decide algo é verificado com
um comando direto. Ferramentas, limites e configuração: [Agentes locais](local-agents.md).
