## 概要
技術ドキュメント及び技術記事（.md）の管理


## ディレクトリ

```
.devcontainer/zenn
├── Dockerfile
├── articles -> /workspaces/etc_docs/articles
└── devcontainer.json
```

## ローカルでの起動方法
VSCodeの`Remote Container: Open Folder in Container...`を実行し、コンテナ環境でVSCodeを開く。
コンテナからzenn-cliを使って記事を作成する。

```bash
npx zenn new:article
npx zenn preview
```
