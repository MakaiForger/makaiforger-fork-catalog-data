# MakaiForger Fork Catalog Database

Banco de dados SQLite contendo o catálogo completo de todos os forks de Proton, Wine, DXVK e VKD3D disponíveis, com suas respectivas versões e metadados. Serve como **cache local** para evitar chamadas excessivas às APIs do GitHub/GitLab/Forgejo.

## Objetivo

Em vez de consultar a API de cada repositório (GitHub, GitLab, Forgejo) toda vez que o usuário abre a lista de versões disponíveis, o app usa este banco local. Isso reduz drasticamente o número de requisições de API, evita rate limits, e acelera a interface.

O banco é atualizado periodicamente (pelo desenvolvedor) conforme novas versões são lançadas.

## Estrutura do banco

```
fork_catalog.db  →  fork_catalog.db.gz (comprimido para download pelo app)
```

### Tabela principal: `fork_tools`

Define cada ferramenta (Proton, Wine, DXVK, VKD3D) disponível: de onde baixar releases, como nomear a pasta de instalação, filtros de assets, etc.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `id` | TEXT PK | Identificador único (ex: `proton-ge`, `wine-staging`) |
| `title` | TEXT | Nome legível (ex: `Proton-GE`, `Wine-Staging`) |
| `category` | TEXT | `proton`, `wine`, `dxvk`, `vkd3d` |
| `endpoint` | TEXT | URL da API de releases (GitHub, GitLab, Forgejo) |
| `directory_name_format` | TEXT | Template do nome da pasta (`$version` é substituído) |
| `support_latest` | INTEGER | 1 = suporta "latest", 0 = versão fixa |
| `prefer_tarball` | INTEGER | 1 = prefere .tar.gz, 0 = .tar.xz ou outro |
| `asset_position` | INTEGER | Posição do asset na lista (0 = primeiro) |
| `asset_filter` | TEXT | Regex para filtrar assets (opcional) |
| `asset_exclude` | TEXT | Regex para excluir assets (opcional) |
| `keywords` | TEXT | JSON array com sinônimos pra busca |
| `sort_order` | INTEGER | Ordem de exibição |
| `author`, `license`, `repo_url` | TEXT | Metadados |

**Exemplo:**
```json
{
  "id": "proton-ge",
  "title": "Proton-GE",
  "category": "proton",
  "endpoint": "https://api.github.com/repos/GloriousEggroll/proton-ge-custom/releases",
  "directory_name_format": "GE-Proton-$version",
  "support_latest": 1,
  "prefer_tarball": 0,
  "asset_position": 0,
  "keywords": "[\"proton-ge\",\"protonge\",\"ge-proton\",\"geproton\",\"ge\"]"
}
```

### Tabela: `fork_releases`

Cache das versões disponíveis de cada fork. É a tabela que o app consulta para exibir a lista de versões.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `id` | INTEGER PK | |
| `fork_id` | TEXT | Referencia `fork_tools.id` |
| `nome_forquilha` | TEXT | Nome do fork |
| `categoria` | TEXT | `proton`, `wine`, etc |
| `versao` | TEXT | Nome da versão (ex: `GE-Proton9-25`) |
| `tag` | TEXT | Tag do release (ex: `GE-Proton9-25`) |
| `publicacao` | TEXT | Data ISO (ex: `2026-06-01T12:00:00Z`) |
| `download_url` | TEXT | URL direta do asset |
| `tamanho_bytes` | INTEGER | Tamanho do arquivo |
| `release_url` | TEXT | URL da página do release |
| `body` | TEXT | Notas de release |
| `autor` | TEXT | Autor do release |
| `licenca` | TEXT | Licença |
| `prerelease` | INTEGER | 1 = pré-release |
| `repositorio` | TEXT | URL do repositório |
| `formato_diretorio` | TEXT | Formato do diretório |
| `assets_json` | TEXT | JSON com lista completa de assets |

**Exemplo:**
```json
{
  "fork_id": "proton-ge",
  "versao": "GE-Proton9-25",
  "tag": "GE-Proton9-25",
  "publicacao": "2026-05-15T10:30:00Z",
  "download_url": "https://github.com/GloriousEggroll/proton-ge-custom/releases/download/GE-Proton9-25/GE-Proton9-25.tar.gz",
  "tamanho_bytes": 482345678,
  "prerelease": 0
}
```

### Tabela: `fork_overview`

Resumo de cada fork com informações estáticas.

| Coluna | Descrição |
|--------|-----------|
| `Fork_ID` | ID do fork |
| `Nome` | Nome legível |
| `Categoria` | proton / wine / dxvk / vkd3d |
| `Autor` | Mantenedor |
| `Licenca` | Licença |
| `Repositorio` | URL do repositório |
| `Formato_Diretorio` | Template do nome da pasta |
| `Recursos` | JSON com recursos especiais |

### Tabelas auxiliares (legado / migração)

| Tabela | Conteúdo |
|--------|----------|
| `fork_versions` | Versão anterior do cache de releases (mesmo que `fork_releases`) |
| `version_changelogs` | Changelogs de cada versão |
| `catalogo_proton` | Catálogo legado |
| `catalogo_proton_completo` | Dados completos de cada release |
| `fork_releases_fts` | Índice FTS5 para busca textual em `fork_releases` |

## Como o app usa este banco

O fluxo no Makai Forger:

```
Usuário abre "Gerenciar Proton" ou "Downloads"
    │
    ▼
tools.ts: getTools()
    │ SELECT * FROM fork_tools ORDER BY category, sort_order
    │ Retorna lista de ferramentas disponíveis (30 forks)
    ▼
Para cada fork, o app pode:
  ├── Exibir versões disponíveis → consulta fork_releases
  ├── Baixar uma versão → usa URL de fork_releases + formato de diretório
  └── Busca textual → usa fork_releases_fts (FTS5)
    │
    ▼
downloader.ts: baixa o asset da URL
    │
    ▼
installer.ts: extrai na pasta ~/games/ProtonForger/<categoria>/
    │ com nome formatado por directory_name_format
```

### Para que serve a `directory_name_format`

Quando o app baixa e instala um fork, ele precisa saber o nome da pasta de instalação. O template usa `$version` como placeholder:

| id | directory_name_format | Exemplo com v9-25 |
|----|----------------------|-------------------|
| `proton-ge` | `GE-Proton-$version` | `GE-Proton-9-25` |
| `wine-staging` | `wine-$version-staging-amd64` | `wine-9.25-staging-amd64` |
| `dxvk` | `dxvk-$version` | `dxvk-2.5` |

### Para que serve `keywords`

O app usa `keywords` para identificar qual fork corresponde a um diretório já instalado no sistema. Por exemplo, se o usuário já tem uma pasta `GE-Proton-9-25`, o app procura nos `keywords` de cada fork até achar um match:

```json
"keywords": "[\"proton-ge\",\"protonge\",\"ge-proton\",\"geproton\",\"ge\"]"
```

## Como adicionar um novo fork

Para adicionar um novo Proton/Wine/DXVK ao catálogo, um desenvolvedor precisa inserir registros em 2 tabelas:

### 1. Adicione em `fork_tools`

```sql
INSERT INTO fork_tools (id, title, category, endpoint, directory_name_format,
                        support_latest, asset_position, type, keywords, author, repo_url)
VALUES (
  'meu-novo-proton',                          -- id único
  'Meu Novo Proton',                           -- título legível
  'proton',                                    -- categoria
  'https://api.github.com/repos/usuario/repo/releases',  -- endpoint da API
  'MeuProton-$version',                        -- formato do diretório (use $version)
  1,                                           -- suporta "latest"
  0,                                           -- posição do asset
  'github',                                    -- tipo (github, gitlab, forgejo)
  '["meu-novo-proton","meuproton","novo"]',    -- keywords para busca
  'Autor',                                     -- autor
  'github.com/usuario/repo'                    -- URL do repo
);
```

### 2. Adicione releases em `fork_releases`

```sql
INSERT INTO fork_releases (fork_id, versao, tag, publicacao, download_url,
                           tamanho_bytes, release_url, body, prerelease)
VALUES (
  'meu-novo-proton',
  'MeuProton-1.0',
  'MeuProton-1.0',
  '2026-06-10T12:00:00Z',
  'https://github.com/usuario/repo/releases/download/MeuProton-1.0/MeuProton-1.0.tar.gz',
  123456789,
  'https://github.com/usuario/repo/releases/tag/MeuProton-1.0',
  'Notas do release...',
  0
);
```

### Boas práticas para adicionar forks

1. **Teste o endpoint** antes — faça um curl na URL e veja se retorna releases válidos
2. **Descubra o `directory_name_format`** — olhe o nome do asset nos releases do repositório original
3. **Keywords abrangentes** — inclua variações com e sem hífen, maiúsculo/minúsculo
4. **Mantenha `support_latest` como 1** a menos que o fork exija versão fixa (ex: Kron4ek)
5. **Adicione pelo menos 3-5 releases** recentes para o banco ser útil

## Como publicar uma nova versão do banco

Procedimento para o desenvolvedor:

1. **Atualize o banco** com novos forks ou novas releases
   ```bash
   sqlite3 resources/fork_catalog.db < suas_atualizacoes.sql
   ```

2. **Compacte**
   ```bash
   gzip -kf resources/fork_catalog.db
   ```

3. **Crie um release no GitHub** com tag incremental
   - Tag: `v0.0.2`, `v0.0.3`, etc. — nunca sobrescreva tags
   - Asset: `fork_catalog.db.gz`
   - Todas as versões anteriores são mantidas

4. **Envie para o GitLab (fallback)**
   ```bash
   curl -X PUT \
     "https://gitlab.com/api/v4/projects/83110945/packages/generic/makaiforger-fork-catalog-data/v0.0.2/fork_catalog.db.gz" \
     -H "PRIVATE-TOKEN: $GL_TOKEN" \
     -H "Content-Type: application/gzip" \
     --data-binary @fork_catalog.db.gz
   ```

5. **Atualize o metadata.json** em `MakaiForger/makaiforger-resources`:
   ```json
   "fork_catalog.db.gz": {
     "version": 2,
     "tag": "v0.0.2",
     "sha256": "<sha256>",
     "size": <tamanho_gz>,
     "original_size": <tamanho_db>,
     "github_repo": "MakaiForger/makaiforger-fork-catalog-data",
     "gitlab_project_id": 83110945
   }
   ```

## Como o app atualiza automaticamente

O `resource-manager.ts` no Makai Forger:

1. Ao iniciar, baixa `metadata.json` de `MakaiForger/makaiforger-resources`
2. Compara versão local vs remota do `fork_catalog.db.gz`
3. Se `localVersion < remoteVersion`, baixa o novo arquivo
4. Verifica SHA256 — se falhar, loga erro e pula
5. Extrai com `gzip -d` para `resources/database/fork_catalog.db`
6. O cache em `tools.ts` é invalidado na próxima consulta

## Sistema de fallback

| Prioridade | Origem | URL |
|------------|--------|-----|
| 1ª | GitHub | `MakaiForger/makaiforger-fork-catalog-data` releases |
| 2ª | GitLab | Project ID 83110945 (generic packages) |

## Versionamento

| Versão | Data | Descrição |
|--------|------|-----------|
| v0.0.1 | - | Primeira versão — 30 forks, 59 releases |

Todas as versões são mantidas nos releases do GitHub e GitLab. Nunca apague versões anteriores.

## Manutenção

- **Periodicidade:** atualizar semanalmente ou conforme novos releases de Proton/Wine surgirem
- **Validação:** verificar se as URLs dos endpoints ainda respondem antes de publicar
- **Consistência:** todo `fork_id` em `fork_releases` deve existir em `fork_tools.id`
- **Tamanho:** o banco é pequeno (~300 KB descomprimido), mas releases acumulados podem crescer — limite a ~30 releases por fork
- **Remoção:** se um fork for descontinuado, mantenha no banco mas marque como inativo (nunca remova, pois usuários podem ter versões instaladas)

Projeto mantido por **desenvolvedor** independente.
