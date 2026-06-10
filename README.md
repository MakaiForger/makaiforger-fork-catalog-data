# MakaiForger Fork Catalog Database

Banco de dados SQLite contendo o catálogo de forks de Proton disponíveis para download e suas classificações por tier.

## Estrutura

```
fork_catalog.db  →  fork_catalog.db.gz (comprimido para download)
```

Tabelas: forks, releases, tiers, compatibilidade.

## Como publicar uma nova versão

1. **Gere o banco** atualizado no projeto principal
2. **Compacte:**
   ```bash
   gzip -kf resources/fork_catalog.db
   ```
3. **Crie um release no GitHub** com tag incremental (v0.0.2, v0.0.3...)
   - Asset: `fork_catalog.db.gz`
   - Nunca apague versões anteriores
4. **Envie para o GitLab (fallback):**
   ```bash
   curl -X PUT "https://gitlab.com/api/v4/projects/83110945/packages/generic/makaiforger-fork-catalog-data/v0.0.2/fork_catalog.db.gz" \
     -H "PRIVATE-TOKEN: $GL_TOKEN" \
     -H "Content-Type: application/gzip" \
     --data-binary @fork_catalog.db.gz
   ```
5. **Atualize o metadata.json** no repositório `MakaiForger/makaiforger-resources`

## Como o aplicativo atualiza

Download automático ao iniciar via `resource-manager.ts`. Comparação de versão + verificação SHA256.

## Fallback

| Prioridade | Origem |
|------------|--------|
| 1ª | GitHub (`MakaiForger/makaiforger-fork-catalog-data`) |
| 2ª | GitLab (Project ID 83110945) |

## Versionamento

| Versão | Data | Descrição |
|--------|------|-----------|
| v0.0.1 | - | Primeira versão estável |
