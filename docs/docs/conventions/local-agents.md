# Agentes locais

Servidor MCP local que delega trabalho operacional barato a um modelo rodando no Ollama. A saída bruta
(logs, `kubectl`, `gh`) fica entre o servidor e o modelo local; só um resultado curto e estruturado volta
ao contexto do Claude Code. Objetivo: economizar tokens sem perder controle. O Claude continua sendo o
agente principal para raciocínio, código, arquitetura e decisões.

!!! note "Fonte de verdade"
    Código e README em `~/Projecs/local-agents` (fora de `cmoreira-dev/`, não é um repositório git).
    Mudou uma ferramenta, a config ou o roteamento? Atualize esta página (PT e EN) e o `CLAUDE.md` da raiz.

## Arquitetura

```text
Claude Code
    | MCP via stdio (registrado em ~/Projecs/.mcp.json)
    v
local-agents (Node, src/server.mjs)
    +-- workers determinísticos: coletam evidência com allowlists, reduzem e sanitizam
    +-- validação de JSON e limite de tamanho da resposta
    v
Ollama (http://localhost:11434, OLLAMA_HOST) -> qwen2.5:14b
```

Os workers fazem o máximo em código (status de sync, health, revisão, contagens vêm do JSON, não do
modelo); o modelo só explica, resume ou redige. O modelo local **nunca executa comandos**: quem executa é
o worker, via `spawn` sem shell, depois de validar argumentos estruturados contra uma allowlist.

## Ferramentas

Todas são somente leitura; as `draft_*` só devolvem rascunhos (nada é escrito em disco).

| Ferramenta | Entrada | Uso | Medido (qwen2.5:14b) |
|---|---|---|---|
| `check_actions` | `repos?`, `limit?` (1-20, padrão 3), `include_failure_logs?`, `verbose?` | Estado do GitHub Actions nos 14 repos configurados; para runs com falha, causa provável e dica | 1,2 s, 87 B quando tudo verde |
| `check_argocd` | `namespace?` (`argocd`), `only_problems?` (`true`) | Apps Argo CD fora de Synced+Healthy, com explicação de cada uma | 22,7 s, 1,9 KB (46 apps, 3 sinalizadas) |
| `investigate_infra` | `resource`, `namespace?`, `helm_release?`, `include_logs?`, `include_events?`, `include_metrics?`, `verbose?` | Pod/workload/Helm: estado, eventos Warning, logs (só para `tipo/nome`) | 11,8 s, 273 B |
| `investigate_gitops` | `application`, `namespace?` (`argocd`), `repository_path?`, `verbose?` | Git vs Argo CD vs cluster de uma app. Sync, health e revisão vêm do código; a CLI `argocd` é opcional | 10,9 s, 892 B |
| `analyze_logs` | `text`, `source?`, `context?` | Diagnóstico estruturado de logs (erros, padrões, causas, próximos passos) | depende do tamanho |
| `draft_docs_update` | `repository_path`, `docs_page`, `base_ref?` (`HEAD~1`), `instructions?` | Lê o diff do repo e a página atual; propõe seções de markdown a mudar | 60,3 s, 1,4 KB |
| `draft_code` | `task`, `files` (1-5), `constraints?` | Tarefa pequena e bem especificada: devolve o conteúdo proposto dos arquivos | 17,5 s, 768 B |

Carregar as ferramentas na sessão (são deferidas):
`ToolSearch("select:mcp__local-agents__check_actions,mcp__local-agents__check_argocd,mcp__local-agents__investigate_infra,mcp__local-agents__investigate_gitops,mcp__local-agents__analyze_logs,mcp__local-agents__draft_docs_update,mcp__local-agents__draft_code")`.

## Quando usar (roteamento)

Regra curta na seção 5 do `CLAUDE.md` da raiz. Em resumo: CI verde? → `check_actions`. Argo sincronizado? →
`check_argocd`. Pod ou Helm com problema → `investigate_infra`. Drift de uma app → `investigate_gitops`.
Log ou saída com mais de ~30 linhas → `analyze_logs`. Doc desatualizada depois de uma mudança →
`draft_docs_update`. Tarefa mecânica pequena → `draft_code`. Sempre revisar antes de aplicar.

## Limites conhecidos do modelo (qwen2.5:14b)

- Faz só parte de edições com vários passos e reformata código sem pedir (`draft_code`).
- Confunde nomes de componentes em docs (`draft_docs_update`) e produz explicações genéricas de Argo CD e Actions.
- Por isso: o que decide algo é verificado com um comando direto, e `draft_*` nunca é aplicado às cegas.
- `qwen2.5-coder:14b` é candidato natural para o agente `code` (`ollama pull qwen2.5-coder:14b`, ~9 GB,
  depois trocar `models.code.model` na config). Ainda não foi baixado.

## Configuração (`config/local-agents.yaml`)

| Chave | Efeito |
|---|---|
| `ollama.host`, `timeout_ms` (180000), `max_input_bytes`, `num_ctx` (16384) | Conexão e janela de contexto (sem `num_ctx` o Ollama pode truncar a entrada em silêncio) |
| `models.{logs,infra,gitops,actions,argocd,docs,code}.model` | Modelo por agente (todos `qwen2.5:14b` hoje) |
| `actions.org`, `actions.repos` | Org e repos que `check_actions` aceita |
| `security.allowed_paths` | Raízes legíveis (`/Users/oliveirac/Projecs`); caminhos resolvidos com realpath |
| `security.max_output_chars` (6000), `max_draft_output_chars` (30000) | Tamanho máximo do resultado; acima disso vira um envelope `{truncated, partial, note}` |

`LOCAL_AGENTS_CONFIG` aponta para outro arquivo; `OLLAMA_HOST` para outro host.

## Segurança

- Somente leitura. `kubectl`/`helm`/`argocd`/`git`: allowlist de subcomandos e bloqueio de verbos que
  mudam estado (`apply`, `delete`, `patch`, `scale`, `sync`, `commit`, `push` ...).
- `gh`: três formas exatas (`run list`, `run view --log-failed`, `pr list`), campos `--json` fixos, ids
  numéricos, `org/repo` restrito a `actions.org`. `merge`, `rerun`, `cancel`, `comment` e `api` são impossíveis.
- Argumentos com sintaxe de shell são rejeitados; nada de `eval`, `sh -c` ou comando montado pelo modelo.
- Entrada sanitizada antes de ir ao modelo (Authorization, Bearer, senhas, API keys, connection strings,
  JWTs). É uma barreira prática, não garantia formal.
- `draft_*` rejeitam caminhos relativos, `..`, symlinks, `.env*`, `*.pem`, `*.key`, `secret*`, `*credentials*`,
  e revalidam os caminhos que o modelo propõe na saída.

## Operação

- Ollama roda como serviço do usuário: `brew services start ollama`; verificar com
  `curl -s localhost:11434/api/version` e `ollama list`.
- Depois de mudar ferramentas ou config: reconectar o servidor (`/mcp` no Claude Code) para carregar a nova versão.
- Testes: `cd ~/Projecs/local-agents && npm test` (31 testes, sem rede nem Ollama).
- Logs do próprio servidor vão para `stderr` em JSON (agente, modelo, duração, tamanhos; sem conteúdo bruto).

## Como acrescentar uma ferramenta

1. Worker em `src/workers/<nome>-worker.mjs` (coleta determinística, redução, uma chamada ao modelo só se necessário).
2. Validador da saída em `src/structured.mjs`/`output-validator.mjs` e entrada de modelo em `models.<nome>` na config.
3. Registro em `src/server.mjs` com descrição dizendo QUANDO usar e que é somente leitura.
4. Testes com executor e modelo injetados (sem rede) e um teste vivo pelo `fixtures/smoke.mjs`.
5. Atualizar esta página e a tabela de roteamento do `CLAUDE.md` da raiz.

## Verificar que está funcionando

Não confie na resposta do Claude para saber se os agentes foram chamados: use estas provas, do mais
simples ao mais independente.

1. **Conexão:** `/mcp` no Claude Code deve listar `local-agents` conectado com 7 ferramentas. Sem isso,
   nenhuma sessão consegue chamá-las (o `ToolSearch` da seção Ferramentas não encontra nada).
2. **Registro de uso no servidor:** `cd ~/Projecs/local-agents && npm run stats`. Mostra arranques do servidor
   (sessões que conectaram), chamadas por ferramenta, erros, latência média e tokens estimados mantidos fora do
   contexto. O log é `logs/usage.jsonl`, uma linha JSON por chamada, sem conteúdo bruto.
3. **Prova do lado do Ollama (independe do servidor):**
   `grep '/api/chat' /opt/homebrew/var/log/ollama.log | tail` lista cada chamada com hora e duração;
   `ollama ps` mostra o modelo carregado enquanto uma chamada roda.
4. **Teste ativo:** numa sessão nova, pedir "estado do CI" ou "health check do Argo CD"; a chamada
   `mcp__local-agents__...` aparece no transcript e ganha uma linha nova em `usage.jsonl`.
5. **Se não conecta:** o registro atual vive em `~/Projecs/.mcp.json` (escopo de projeto: só vale para sessões
   abaixo de `~/Projecs` e pode pedir aprovação de novo). Alternativa robusta, em escopo de usuário:
   `claude mcp add --scope user local-agents -- node /Users/oliveirac/Projecs/local-agents/src/server.mjs`.
   O stderr do servidor fica em `~/Library/Caches/claude-cli-nodejs/<projeto>/mcp-logs-local-agents/`.

**Como ler os números:** `est_tokens_kept_out` é um limite inferior (entrada enviada ao modelo local menos o
resultado devolvido) e chamadas pequenas dão 0. O ganho aparece com logs longos, `kubectl describe` e
`check_actions` sobre muitos repos; o `gh`/`kubectl` bruto que o worker reduz antes do modelo não entra na conta.
