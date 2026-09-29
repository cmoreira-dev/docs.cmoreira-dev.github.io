# Manutenção da documentação

Como manter este site atualizado como parte do trabalho, não depois dele.

!!! note "Fonte de verdade"
    Regra curta no `CLAUDE.md` da raiz (protocolo de sessão). Aqui está o procedimento completo.

## Quando atualizar

Depois de qualquer mudança **funcional** (endpoint, contrato, variável de ambiente, rota, infraestrutura,
processo de deploy, decisão de arquitetura) e ao fechar ou descobrir uma pendência de produto. A
atualização vai no mesmo trabalho que a mudança, idealmente na mesma janela de PRs.

## Qual página

| Pilar | Páginas em `docs/docs/` |
|---|---|
| Cluster, rede, secrets | `architecture/*` |
| IaC | `iac/*` |
| GitOps, addons, Burrito, Renovate, chart genérico | `gitops/*`, `kubernetes/generic-app-chart` |
| CI/CD, registry, Argo CD | `cicd/*`, `argocd` |
| Convenções de repositório e fluxo | `repos`, `conventions/*` |
| teupadel.com | `apps/teupadel`, `apps/teupadel-{api,ui,processor}`, `products/teupadel-{backlog,brand,email-ses}` |
| Sara | `apps/sara`, `apps/sara-{api,ui,backlog}` |

## Como

1. Editar a página em português (`x.md`) e o gêmeo em inglês (`x.en.md`) com a mesma estrutura.
2. Página nova: incluir no `nav:` de `docs/mkdocs.yml` e a tradução do título em
   `plugins.i18n.languages[en].nav_translations`.
3. Todo início de página carrega o aviso "Fonte de verdade" indicando o que a alimenta.
4. Validar: `cd docs && ../venv/bin/mkdocs build --strict` (falha em link quebrado ou entrada de nav ausente).
5. Ramo + PR no repo `docs.cmoreira-dev.github.io`; o workflow `deploy-docs.yml` publica em `docs.cmoreira.dev`.

## Rascunho com o agente local

`draft_docs_update` lê o diff de um repo e a página atual e devolve as seções que precisam mudar como
markdown proposto. Ele não escreve nada: revisar e aplicar com Edit. Detalhes em [Agentes locais](local-agents.md).

## Páginas de backlog

`products/teupadel-backlog` e `apps/sara-backlog` são a única fonte de pendências e status. Atualizar o
item no mesmo trabalho que o resolve ou o cria, com a data. Não criar arquivos de status soltos nas
pastas dos produtos (não são repositórios e não são versionados).

## O que não vai para os `CLAUDE.md`

Tabelas de env vars, contratos JSON, árvores de diretório, detalhes de deploy e histórico de incidentes
vivem aqui. O `CLAUDE.md` do repo guarda só comandos, armadilhas com o motivo e um ponteiro para a página.
