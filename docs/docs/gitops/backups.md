# Backups do Postgres (CNPG)

!!! note "Fonte da verdade"
    `iac.homelab-live-infra` (`aws/cmoreira-dev/us-east-1/s3-cnpg-backups`: bucket, usuário IAM e
    parâmetros SSM), `gitops.cnpg` (`kustomize/barman-cloud-plugin`, `kustomize/<banco>/`) e
    `gitops.backstage.homelab` (`kustomize/postgres`).

## Arquitetura

Os bancos são `Cluster`s do CloudNativePG (operador 1.30.1, 1 instância cada). O backup usa o **plugin
barman-cloud** (v0.15.1, namespace `cnpg-system`), que substitui o `barmanObjectStore` embutido,
depreciado. Ele faz dois trabalhos: **arquiva os WAL continuamente** e tira **backups base** para o S3.

- **Bucket** `cmoreira-dev-cnpg-backups-eu`, em `eu-west-1` (os backups têm dados pessoais): privado,
  versionado, criptografia SSE-S3, só TLS. Versões antigas expiram em 30 dias.
- **Usuário IAM** `cnpg-backups`, limitado a esse bucket. As chaves vão para o SSM em
  `/homelab/cnpg/backups-aws/{access-key,secret-key,region}` (o cluster não tem IRSA, então a chave é
  estática, entregue por `ExternalSecret`).
- **Por banco**, no namespace dele: `ExternalSecret` `cnpg-backups-aws`, `ObjectStore` `<banco>-backups`,
  `ScheduledBackup` `<banco>-daily` e o bloco `plugins` no `Cluster` (com `isWALArchiver: true`).
- **Retenção** de 14 dias (`retentionPolicy` do `ObjectStore`). O layout no S3 é
  `s3://cmoreira-dev-cnpg-backups-eu/<banco>/<banco>/{base,wals}/`.

## Cobertura

| Banco | Onde é definido | Agendamento (UTC) | Estado |
|---|---|---|---|
| n8n | `gitops.cnpg` (`kustomize/n8n`) | 03:00 | ativo; restore testado em 2026-10-03 |
| litellm | `gitops.cnpg` (`kustomize/litellm`) | 03:10 | ativo |
| backstage | `gitops.backstage.homelab` (`kustomize/postgres`) | 03:20 | ativo |
| teupadel | `gitops.cnpg` (`kustomize/teupadel`) | 03:30 | ativo |

O cluster antigo `cnpg-backstage` estava órfão (o app usa o banco do namespace `backstage`) e foi
desativado em 2026-10-03.

## Restaurar

Um restore cria **um cluster novo** a partir do backup; não sobrescreve o existente. Exemplo mínimo
(num namespace de teste, com o mesmo `ExternalSecret` e `ObjectStore` do banco original):

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: n8n-restore
spec:
  instances: 1
  storage:
    size: 5Gi
  bootstrap:
    recovery:
      source: origin
      database: n8ndb
      owner: n8nuser
  externalClusters:
    - name: origin
      plugin:
        name: barman-cloud.cloudnative-pg.io
        parameters:
          barmanObjectName: n8n-backups
          serverName: n8n
```

Para recuperar até um instante, acrescente `recoveryTarget.targetTime` em `bootstrap.recovery`.

**Teste feito (n8n, 2026-10-03):** o cluster restaurado subiu, e um Job leu o banco: o usuário owner, 275
migrations e as 4 linhas de `settings` estavam lá. Workflows e credenciais estavam em 0 porque ainda não
existiam no n8n; vale repetir o teste quando houver dados reais.

## Armadilhas

- **Ligar o plugin reinicia a instância uma vez** (injeta o sidecar). Com 1 instância, o app vê uma queda
  curta.
- **`immediate: true` no `ScheduledBackup` falha** quando o primeiro backup dispara durante esse
  reinício (`requested plugin is not available`). Por isso o agendamento não usa `immediate`; o primeiro
  backup base vem de um `Backup` pontual (`<banco>-initial`) aplicado depois que o cluster está saudável.
- **O operador preenche `plugins[].enabled: true`**. Declare no manifesto, senão o ArgoCD mostra o
  `Cluster` como OutOfSync.
- Cada namespace precisa do seu `ExternalSecret` e do seu `ObjectStore`; eles não atravessam namespace.

## Riscos conhecidos

- Um único bucket e um único usuário IAM para todos os bancos, na mesma conta AWS dos bancos.
- Sem alerta quando um backup falha; hoje se confere o status do `Backup` e a condição
  `LastBackupSucceeded` do `Cluster` à mão.
- O backup do n8n protege os dados, mas as credenciais salvas só são legíveis com a
  `N8N_ENCRYPTION_KEY`, que mora no SSM. Guarde uma cópia fora da AWS.
- Instância única: o RPO é o intervalo de arquivamento de WAL, e o RTO é o tempo de um restore manual.
