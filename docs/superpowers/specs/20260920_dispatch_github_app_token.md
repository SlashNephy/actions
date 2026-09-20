# dispatch を GitHub App トークン方式へ移行する

- Issue: https://github.com/SlashNephy/actions/issues/46
- 作成日: 2026-09-20

## 背景

`build-docker-image.yml` の `repository_dispatch` ステップは、呼び出し側から渡される
`secrets.dispatch-github-token`（Bot アカウントの PAT `BOT_GITHUB_TOKEN`）を使って
`SlashNephy/infrastructure` に `update-image-digest` イベントを送っている。

PAT は失効期限の管理が必要で、スコープもリポジトリ単位に絞りにくい。
GitHub App のインストールアクセストークンへ移行し、有効期限 1 時間・
単一リポジトリ・単一権限のトークンで dispatch する。

## 現状の呼び出し元

`dispatch-github-token` を渡しているのは 3 リポジトリ・8 ワークフロー。

| リポジトリ | ワークフロー数 | 備考 |
| --- | --- | --- |
| `SlashNephy/infrastructure` | 6 | 自リポジトリへの dispatch |
| `SlashNephy/kuroda-bot` | 1 | |
| `SlashNephy/mackerel-plugin-switchbot` | 1 | |

いずれも `uses:` を SHA でピン留めしているため、破壊的変更を入れても
参照を更新するまで既存の動作は壊れない。

## 設計

### トークンの発行場所

再利用可能ワークフローは自前の secret を持てず、必ず呼び出し側から渡される。
また呼び出し側はジョブレベルの `uses:` であるため前段にステップを挿入できず、
別ジョブの output 経由ではマスク済みの値が GitHub に削除される。
したがって `actions/create-github-app-token` は再利用可能ワークフローの内部で実行し、
private key のみを secret として受け取る。

### インターフェース

`workflow_call` の `secrets` を次のように変更する。

- 削除: `dispatch-github-token`
- 追加: `dispatch-app-private-key`（`required: false`）

`dispatch-update-image-digest: true` かつ private key 未設定の場合は
`actions/create-github-app-token` が `private-key` 必須エラーで失敗する。
dispatch が静かに欠落するよりよいため、これを期待する挙動とする。

### ステップ

```yaml
- uses: actions/create-github-app-token@bcd2ba49218906704ab6c1aa796996da409d3eb1 # v3.2.0
  id: dispatch-app-token
  if: inputs.dispatch-update-image-digest
  with:
    client-id: Iv23liKChiFOA2aFE8Wa # slashnephy-infrastructure
    private-key: ${{ secrets.dispatch-app-private-key }}
    owner: SlashNephy
    repositories: infrastructure
    permission-contents: write
- uses: peter-evans/repository-dispatch@28959ce8df70de7be546dd1250a005dd32156697 # v4.0.1
  if: inputs.dispatch-update-image-digest
  with:
    token: ${{ steps.dispatch-app-token.outputs.token }}
    # repository 以下は変更しない
```

- `app-id` は v3.2.0 で deprecated であり `client-id` が推奨されるため、Client ID を使う。
- Client ID は秘密情報ではないため、公開リポジトリへのハードコードで問題ない。
  dispatch 先の `SlashNephy/infrastructure` が既にハードコードされているのと同じ方針を採る。
- `owner` / `repositories` / `permission-contents` でトークンを
  `SlashNephy/infrastructure` の contents write のみに絞る。
  トークンはジョブ終了時の post ステップで自動的に revoke される。

### 未確定事項

`POST /repos/{owner}/{repo}/dispatches` に必要な fine-grained 権限は
REST ドキュメントに明記がない（classic token の `repo` スコープのみ記載）。
`permission-contents: write` で足りるかは実 dispatch で検証し、
不足する場合は `permission-contents` の指定を外す（App が持つ全権限のトークンになる）。

## 受信側への影響

`infrastructure/.github/workflows/update-image-digest.yml` は変更不要。
このワークフローは dispatch の sender を参照せず、`client_payload` の
`images` / `digest` / 呼び出し元の `github` コンテキストのみを使う。

## スコープ外

### `BOT_GITHUB_TOKEN` の全廃

`BOT_GITHUB_TOKEN` は受信側の `update-image-digest.yml`（checkout、
create-pull-request、`gh pr merge`）でも使われ、さらに `TVTest-builder`、
`anime-titles-map`、`divination-distrib`、`anime-vod-data` などでも使われている。
本 issue で PAT を外せるのは dispatch 経路のみであり、secret を削除できるのは
`kuroda-bot` と `mackerel-plugin-switchbot` の 2 リポジトリに留まる。
受信側の PAT 排除は別 issue とする。

### コンテナスキャンの SARIF アップロード

`crazy-max/ghaction-container-scan` のステップに `id: scan` が無いため、
後続の `steps.scan.outputs.sarif != ''` が常に偽となり
`github/codeql-action/upload-sarif` が実行されていない。既存の不具合であり、
本 issue では触れず別途対応する。

## 検証

このリポジトリ単体では `workflow_call` を実行できないため、実経路で検証する。

1. `SlashNephy/infrastructure` に `DISPATCH_APP_PRIVATE_KEY` を設定する。
2. `infrastructure/.github/workflows/deploy-curl-jq.yml` を本ブランチの SHA と
   新しい secret 名に一時的に向け、`workflow_dispatch` で実行する。
3. 次の証跡を取得する。
   - 呼び出し元 run のトークン発行ログと dispatch ステップの成功
   - `infrastructure` 側で起動した `update-image-digest` run

呼び出し側の編集が本リポジトリの PR マージより先になるが、
`uses:` を SHA でピン留めしているため他の呼び出し元には影響しない。

## ロールアウト

1. 本 PR をマージし、`v0.0.8` タグを打つ。
2. `kuroda-bot` と `mackerel-plugin-switchbot` に `DISPATCH_APP_PRIVATE_KEY` を設定する。
3. 8 ワークフローの `uses:` と `secrets:` を更新する。
4. `kuroda-bot` と `mackerel-plugin-switchbot` の `BOT_GITHUB_TOKEN` を削除する。
