# Padrões de Repositório

A org GitHub `cmoreira-dev` é dividida em categorias — cada pasta de nível
superior no workspace de desenvolvimento agrupa repositórios relacionados, mas
**cada repositório é independente**, com seu próprio histórico, CI e ciclo de
release.

| Categoria | Conteúdo |
|---|---|
| `infra-as-code/` | Terraform/Terragrunt — infraestrutura provisionada |
| `platform-engineering/` | Addons de cluster (`gitops.*`) e portal interno (Backstage) |
| `teupadel.com/` | Aplicação de análise de movimento de padel |
| `sara/` | Aplicação de prática de cifras/letras |
| `documentation/` | Este site |

## Convenção de prefixo

O prefixo do nome do repositório indica o que ele é e como é operado:

| Prefixo | Significado | Como é aplicado no ambiente |
|---|---|---|
| `gitops.*` | Workload gerenciado por ArgoCD | Auto-descoberto pelo `ApplicationSet/gitops-repos`, sincronizado continuamente |
| `iac.*` | Estado de infraestrutura viva (Terragrunt) | CI: `plan` em PR, `apply` no merge, `drift` nightly (ver [Dois níveis de IaC](iac/tiers.md)) |
| `api.*` | Serviço de backend | Build de imagem Docker via CI, deploy via um `gitops.*` correspondente |
| `ui.*` | Aplicação de frontend | Idem — build via CI, deploy via `gitops.*` |
| `docs.*` | Site de documentação | Build local com MkDocs |
| `backstage.*` | Portal interno (Backstage) | Deploy via `gitops.*` como qualquer app |

## Estrutura interna esperada de um repo `gitops.<app>`

```
gitops.<app>/
├── argocd/
│   └── <app>.yaml         ← Application(s) — ArgoCD aponta pra cá
├── helm/
│   ├── Chart.yaml          ← depende de gitops.generic-app-chart
│   └── values.yaml
├── catalog-info.yaml       ← registro no Backstage
└── renovate.json           ← bump automático de versão de chart
```

Duas variações desse padrão coexistem:

- **App própria** (`gitops.template`, `gitops.teupadel.com`,
  `gitops.local-sara`) — depende do
  [chart genérico](kubernetes/generic-app-chart.md), `argocd/` normalmente
  vazio (o `ApplicationSet` gera a `Application` sozinho) — só é populado
  manualmente quando o repo entrega **mais de um componente** (ex.: api + ui no
  mesmo repo), como é o caso de `gitops.teupadel.com` e `gitops.local-sara`.
- **Addon de terceiros** (`gitops.core-addons`, `gitops.ai-core-addons`,
  `gitops.cnpg`, `gitops.monitoring`, `gitops.headlamp`) —
  `helm/<addon>/Chart.yaml` depende diretamente do chart upstream do projeto
  (cert-manager, CNPG, Burrito, etc.), sem passar pelo chart genérico.

## Repos de aplicação (`api.*` / `ui.*`)

Convenção comum aos dois:

```
api.<app>/ ou ui.<app>/
├── Dockerfile
├── CLAUDE.md          ← contexto de arquitetura/decisões daquele componente
├── README.md
└── (código-fonte)
```

- **`api.*`**: Python 3.11 + FastAPI + Uvicorn é o padrão atual em todos os
  serviços de backend.
- **`ui.*`**: Next.js (App Router) é o padrão atual; build sempre via Docker
  multi-stage (nunca `npm install` local publicado) — o `Dockerfile` define os
  valores de produção via `ARG`, já que o workflow de CI reutilizável não passa
  `--build-arg`.

## Fluxo de trabalho para uma mudança

1. **Escopar a mudança à categoria certa** — confirmar qual pasta é dona do
   arquivo antes de tocar nele.
2. **Aplicar a mudança dentro do repo** — cada subpasta é um repo Git
   independente.
3. **Atualizar a documentação** — este site (`docs.cmoreira-dev`), se a mudança
   afeta algo descrito aqui.
4. **Checar impacto entre repos** — alguns acoplamentos importantes:
    - `iac.homelab-live-infra` consome `iac-proxmox-lxc` e
      `iac-aws-ecr-pipeline` via `ref` de Git — uma mudança de módulo pode
      precisar de bump de versão no consumidor.
    - Componentes `api.*`/`ui.*` de uma mesma app compartilham contrato de API
      e, às vezes, o mesmo repo `gitops.*` — uma mudança de rota/env var num
      lado pode exigir mudança correspondente no outro.
    - `gitops.core-addons` é dependência de praticamente todo outro `gitops.*`
      (Gateway API, cert-manager, External Secrets) — mudanças ali têm raio de
      impacto no nível do cluster inteiro.

## Regras gerais

- Nunca commitar direto em `main` sem sinalizar — preferir branch + PR.
- `terraform apply` / `terragrunt apply` sempre sob confirmação explícita.
- `kubectl apply` direto não é usado em repos GitOps — o ArgoCD é quem aplica.
