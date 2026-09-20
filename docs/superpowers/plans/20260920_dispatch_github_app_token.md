# dispatch を GitHub App トークン方式へ移行する実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `build-docker-image.yml` の `repository_dispatch` を Bot の PAT から GitHub App のインストールアクセストークンに置き換える。

**Architecture:** 再利用可能ワークフローの内部で `actions/create-github-app-token` を実行し、`SlashNephy/infrastructure` の contents write のみに絞ったトークンを発行して `peter-evans/repository-dispatch` に渡す。呼び出し側から受け取る secret は PAT から App の private key へ差し替える。

**Tech Stack:** GitHub Actions (reusable workflow), actions/create-github-app-token v3.2.0, peter-evans/repository-dispatch v4.0.1, actionlint

**Spec:** `docs/superpowers/specs/20260920_dispatch_github_app_token.md`

## Global Constraints

- 設計書: `docs/superpowers/specs/20260920_dispatch_github_app_token.md`
- Issue: https://github.com/SlashNephy/actions/issues/46
- 作業ブランチ: `feat/dispatch-github-app-token`（作成済み、設計書コミット済み）
- 対象ファイル: `.github/workflows/build-docker-image.yml` のみ
- GitHub App: `slashnephy-infrastructure` / Client ID `Iv23liKChiFOA2aFE8Wa`
- dispatch 先: `SlashNephy/infrastructure`（既存のハードコードを踏襲）
- `actions/create-github-app-token` のピン: `bcd2ba49218906704ab6c1aa796996da409d3eb1 # v3.2.0`
- すべての `uses:` は SHA ピン留め＋バージョンコメントを維持する
- コミットメッセージは Conventional Commits、日本語、`Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>` 付き
- ログ・エラーメッセージは英語、コメントは日本語（リポジトリの慣習）
- 公開リポジトリのため、private key その他の秘密情報を成果物に含めない（Client ID は秘密情報ではない）
- 品質ゲート: `actionlint`（`/home/spica/.local/share/mise/installs/actionlint/1.7.12/actionlint`、CI には未設定のためローカル実行）

---

### Task 1: `build-docker-image.yml` を App トークン方式に変更する

**Files:**
- Modify: `.github/workflows/build-docker-image.yml`（`secrets:` ブロックと `repository-dispatch` ステップ周辺）

**Interfaces:**
- Consumes: なし（このリポジトリで最初のタスク）
- Produces: `workflow_call` の secret 名 `dispatch-app-private-key`（`required: false`）。Task 2 以降の検証・ロールアウトはこの名前を使う。`dispatch-github-token` は廃止され、以後どこからも参照されない。

このリポジトリにはテストランナーが無く、`workflow_call` ワークフローは単体で実行できない。
したがって TDD のレッド・グリーンは `actionlint` による静的検証と、Task 2 の実 dispatch で代替する。

- [ ] **Step 1: 変更前の actionlint がクリーンであることを確認する**

Run:
```bash
actionlint .github/workflows/build-docker-image.yml
```
Expected: 出力なし・終了コード 0（変更後の差分が actionlint 由来であると切り分けるためのベースライン）

- [ ] **Step 2: `workflow_call` の secret を差し替える**

`.github/workflows/build-docker-image.yml` の `secrets:` ブロックで
`dispatch-github-token` を削除し、`dispatch-app-private-key` を追加する。

変更前:
```yaml
      build-secrets:
        required: false
      dispatch-github-token:
        required: false
```

変更後:
```yaml
      build-secrets:
        required: false
      dispatch-app-private-key:
        required: false
```

- [ ] **Step 3: トークン発行ステップを追加し、dispatch のトークン参照を差し替える**

`peter-evans/repository-dispatch` のステップの直前にトークン発行ステップを挿入し、
`token:` の参照を発行済みトークンに変更する。

変更前:
```yaml
      - uses: peter-evans/repository-dispatch@28959ce8df70de7be546dd1250a005dd32156697 # v4.0.1
        if: inputs.dispatch-update-image-digest
        with:
          token: ${{ secrets.dispatch-github-token }}
          repository: SlashNephy/infrastructure
```

変更後:
```yaml
      # dispatch 先のリポジトリに限定した、有効期限 1 時間のトークンを発行する。
      # ジョブ終了時に post ステップが自動で revoke する。
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
          repository: SlashNephy/infrastructure
```

`event-type` 以下の `client-payload` は変更しない。

- [ ] **Step 4: 旧 secret 名が残っていないことを確認する**

Run:
```bash
grep -rn 'dispatch-github-token' .github/
```
Expected: ヒットなし・終了コード 1

- [ ] **Step 5: actionlint を通す**

Run:
```bash
actionlint .github/workflows/build-docker-image.yml
```
Expected: 出力なし・終了コード 0

指摘が出た場合は設定変更やコメントによる抑制をせず、ユーザーに対応方針を確認する。

- [ ] **Step 6: 差分を目視確認する**

Run:
```bash
git diff -- .github/workflows/build-docker-image.yml
```
Expected: `secrets:` の 1 行差し替えと、トークン発行ステップの追加＋`token:` の参照変更のみ。
`build-args` / `secret-envs` / `build-secrets` / スキャン関連のステップに差分が無いこと。

- [ ] **Step 7: コミットする**

```bash
git add .github/workflows/build-docker-image.yml
git commit -m "$(cat <<'MSG'
feat: dispatch に GitHub App のトークンを使う

Bot の PAT ではなく、dispatch 先のリポジトリに限定した
GitHub App のインストールアクセストークンで repository_dispatch する。

Refs #46

Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
MSG
)"
```

- [ ] **Step 8: push して Draft PR を作成する**

Task 2 の実経路検証が未完了のため、この時点では Draft とする。

```bash
git push -u origin feat/dispatch-github-app-token
gh pr create --draft --assignee SlashNephy --title "feat: dispatch に GitHub App のトークンを使う" --body-file -
```

PR 本文には次を含める。
- 変更の要約（PAT → GitHub App トークン）
- `Close #46`
- 呼び出し側に必要な変更（`dispatch-github-token` → `dispatch-app-private-key`）
- `🤖 Generated with [Claude Code](https://claude.com/claude-code)`

---

### Task 2: 実経路で dispatch を検証する

> **注意:** このタスクは `SlashNephy/infrastructure` を変更する。着手前にユーザーの承認を得ること。

**Files:**
- Modify（一時的・`SlashNephy/infrastructure` 側）: `.github/workflows/deploy-curl-jq.yml`

**Interfaces:**
- Consumes: Task 1 が定義した secret 名 `dispatch-app-private-key` と、push 済みブランチのコミット SHA
- Produces: PR に添付する before / after の証跡（run URL とログ）

呼び出し側の編集が本リポジトリの PR マージより先になるが、他の呼び出し元は
`uses:` を SHA でピン留めしているため影響を受けない。

- [ ] **Step 1: 検証に使うコミット SHA を控える**

Run:
```bash
git rev-parse HEAD
```
Expected: Task 1 でコミットした SHA（以下 `<SHA>`）

- [ ] **Step 2: `infrastructure` に App の private key を設定する**

private key の PEM ファイルはユーザーが用意する。エージェントは値を要求も表示もしない。

```bash
gh secret set DISPATCH_APP_PRIVATE_KEY -R SlashNephy/infrastructure < /path/to/private-key.pem
```
Expected: `✓ Set Actions secret DISPATCH_APP_PRIVATE_KEY for SlashNephy/infrastructure`

- [ ] **Step 3: `deploy-curl-jq.yml` を検証用に向け直す**

`SlashNephy/infrastructure` に短命のブランチ（例: `test/dispatch-app-token`）を切り、
`.github/workflows/deploy-curl-jq.yml` の `build` ジョブを次のように変更して push する。
`workflow_dispatch` はワークフローファイルを含む任意のブランチから実行できるため、
default branch を触る必要はない。検証後はブランチを削除すれば元に戻る。

変更前:
```yaml
    uses: SlashNephy/actions/.github/workflows/build-docker-image.yml@cf12e2d382df42d9fcfc4ee94f901661d8255d19 # v0.0.7
```
```yaml
    secrets:
      dispatch-github-token: ${{ secrets.BOT_GITHUB_TOKEN }}
```

変更後（`<SHA>` は Step 1 の値）:
```yaml
    uses: SlashNephy/actions/.github/workflows/build-docker-image.yml@<SHA> # feat/dispatch-github-app-token
```
```yaml
    secrets:
      dispatch-app-private-key: ${{ secrets.DISPATCH_APP_PRIVATE_KEY }}
```

- [ ] **Step 4: 検証実行する**

```bash
gh workflow run deploy-curl-jq.yml -R SlashNephy/infrastructure --ref test/dispatch-app-token
gh run list -R SlashNephy/infrastructure --workflow deploy-curl-jq.yml --limit 1
```
Expected: 新しい run が queued / in_progress で表示される

- [ ] **Step 5: 呼び出し元 run の成功を確認する**

```bash
gh run watch <run-id> -R SlashNephy/infrastructure --exit-status
gh run view <run-id> -R SlashNephy/infrastructure --json jobs \
  --jq '.jobs[].steps[] | select(.name | test("create-github-app-token|repository-dispatch")) | .name + " " + .conclusion'
```
Expected: run が success。`Run actions/create-github-app-token`、
`Run peter-evans/repository-dispatch`、および post ステップ
`Post Run actions/create-github-app-token`（トークンの revoke）がいずれも `success`。

`permission-contents: write` が不足して dispatch が 403 / 404 で失敗した場合は、
`permission-contents` の指定を外して Task 1 の Step 5 以降をやり直し、
設計書の「未確定事項」に結果を反映する。

- [ ] **Step 6: 受信側 run の起動を確認する**

```bash
gh run list -R SlashNephy/infrastructure --workflow update-image-digest.yml --limit 3
```
Expected: `repository_dispatch` を起因とする新しい `Update Image Digest` run が存在し、
success で終了していること。digest が既存と同一の場合は PR が作られないため、
PR の有無は成否の判定に使わない。

- [ ] **Step 7: 証跡を PR に添付する**

呼び出し元 run と受信側 run の URL、および Step 5 の抜粋ログを PR 本文の
「検証」セクションに追記する。スクリーンショットを添付する場合は
`github-image-upload` スキル（`gh image upload`）を使う。

```bash
gh pr edit <pr-number> --body-file -
```

- [ ] **Step 8: PR を Ready にする**

```bash
gh pr ready <pr-number>
```

---

### Task 3: ロールアウトする

> **注意:** このタスクはユーザー主導の手順書である。エージェントは自動実行しない。
> PR のマージ、タグの作成、他リポジトリへの push、secret の設定・削除は
> いずれもユーザーの承認を個別に得たうえで行う。

**Files:**
- Modify（`SlashNephy/infrastructure`）: `deploy-curl-jq.yml`, `deploy-wait-for.yml`, `deploy-headlessx.yml`, `deploy-actions-runner.yml`, `deploy-watch-k8s-events.yml`, `deploy-restart-epgstation-deployment.yml`
- Modify（`SlashNephy/kuroda-bot`）: `.github/workflows/build-image.yml`
- Modify（`SlashNephy/mackerel-plugin-switchbot`）: `.github/workflows/build-image.yml`

**Interfaces:**
- Consumes: Task 1 の secret 名と、マージ後に打つタグ `v1.0.0`
- Produces: 全呼び出し元が App トークン方式に移行した状態

- [ ] **Step 1: PR をマージしてタグを打つ**

secret 名の変更は呼び出し側にとって破壊的変更である。`v0.0.7` からの
`v0.0.8` は Renovate に patch と判定され、呼び出し側が自動更新されて壊れるため、
メジャーバージョンを上げる。

```bash
gh pr merge <pr-number> --squash --delete-branch
git checkout main && git pull
git tag v1.0.0 && git push origin v1.0.0
git rev-parse v1.0.0
```
Expected: `v1.0.0` が push され、SHA が得られる（以下 `<TAG_SHA>`）

- [ ] **Step 2: 残り 2 リポジトリに private key を設定する**

```bash
gh secret set DISPATCH_APP_PRIVATE_KEY -R SlashNephy/kuroda-bot < /path/to/private-key.pem
gh secret set DISPATCH_APP_PRIVATE_KEY -R SlashNephy/mackerel-plugin-switchbot < /path/to/private-key.pem
```
Expected: 両方で `✓ Set Actions secret ...`

- [ ] **Step 3: 8 ワークフローの参照を更新する**

各ファイルで次の 2 箇所を変更する（`<TAG_SHA>` は Step 1 の値）。

```yaml
    uses: SlashNephy/actions/.github/workflows/build-docker-image.yml@<TAG_SHA> # v1.0.0
```
```yaml
    secrets:
      dispatch-app-private-key: ${{ secrets.DISPATCH_APP_PRIVATE_KEY }}
```

Task 2 で一時的に編集した `deploy-curl-jq.yml` も、ここでタグの SHA に戻す。

- [ ] **Step 4: 旧 secret 名の残骸が無いことを確認する**

```bash
gh api "search/code?q=org:SlashNephy+dispatch-github-token" --jq '.items[]|.repository.full_name+" "+.path'
```
Expected: `.github/workflows/` 配下のヒットなし（計画・設計ドキュメントのヒットは無視してよい）

- [ ] **Step 5: `BOT_GITHUB_TOKEN` の削除可否を報告する**

`kuroda-bot` と `mackerel-plugin-switchbot` では `BOT_GITHUB_TOKEN` が
dispatch でしか使われていないため削除できる。secret の削除は破壊的操作のため
エージェントは実行せず、ユーザーに次のコマンドを提示して判断を仰ぐ。

```bash
gh secret delete BOT_GITHUB_TOKEN -R SlashNephy/kuroda-bot
gh secret delete BOT_GITHUB_TOKEN -R SlashNephy/mackerel-plugin-switchbot
```

`SlashNephy/infrastructure` では受信側の `update-image-digest.yml` が
引き続き使用するため削除しない。
