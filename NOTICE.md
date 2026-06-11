# ⚠️ ATENÇÃO: Ao publicar um novo release

Este repositório fornece recursos para o **Makai Forger**.

## Regra obrigatória

**Sempre que você publicar uma nova versão (release) de QUALQUER recurso, você DEVE atualizar o arquivo `metadata.json` no repositório [`MakaiForger/makaiforger-resources`](https://github.com/MakaiForger/makaiforger-resources).**

### Por quê?

O aplicativo Makai Forger, ao iniciar, baixa o `metadata.json` do repositório `makaiforger-resources` e compara as versões locais com as versões remotas. Se o `metadata.json` não for atualizado, o aplicativo NUNCA saberá que uma nova versão existe e nunca baixará a atualização.

### O que fazer

1. Publique o release no GitHub (com a tag `vX.Y.Z`)
2. Faça upload do asset para o GitLab (project ID correspondente)
3. **Edite o `metadata.json`** em `MakaiForger/makaiforger-resources`:
   - Atualize `version` (incremente)
   - Atualize `tag` para a nova tag
   - Atualize `sha256` e `size` do novo asset
4. Commit e push

### Exemplo de entrada no metadata.json

```json
"nome-do-recurso.tar.gz": {
  "version": 2,
  "tag": "v0.0.2",
  "sha256": "<sha256_do_novo_arquivo>",
  "size": <tamanho_em_bytes>,
  "original_size": 0,
  "github_repo": "MakaiForger/nome-do-repo",
  "gitlab_project_id": <id_do_projeto_no_gitlab>
}
```

> ⚡ Dica: o campo `version` é numérico incremental (1, 2, 3...). O `tag` é a versão legível (v0.0.2). Nunca sobrescreva tags ou versões existentes.
