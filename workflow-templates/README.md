# Workflow templates

tuqulore 組織で共通利用するための GitHub Actions ワークフローテンプレート集。

| テンプレート | 用途 |
| ------------ | ---- |
| [`ci.yml`](./ci.yml) | push 起動で lint / test を実行し、フォーマット差分を自動コミット |
| [`release.yml`](./release.yml) | `workflow_dispatch` で起動し、release ブランチでリリース PR を作成 (lerna monorepo) |
| [`publish.yml`](./publish.yml) | リリース PR のマージで起動し、npm 公開 → タグ作成 → alpha バンプ PR (lerna monorepo) |

利用側のリポジトリでは GitHub UI の「New workflow」画面から該当テンプレートを選んで生成する。生成された yaml は通常のワークフロー扱いなので、リポジトリに合わせて自由に編集してよい。

## 共通前提

### Composite action への依存

すべてのテンプレートは `tuqulore/.github` リポジトリ内の以下の composite action を `@main` で参照する。テンプレートを使うには利用側のリポジトリから本リポジトリにアクセスできる必要がある (public リポジトリのため通常は問題なし)。

| Composite action | 役割 |
| ---------------- | ---- |
| [`setup`](../setup/action.yml) | `actions/checkout` + pnpm + Node.js のセットアップと `pnpm install` |
| [`configure-git`](../configure-git/action.yml) | CI 内でのコミット用に git の user.name / user.email を設定 |
| [`format`](../format/action.yml) | `pnpm format` を実行して差分を自動コミット |
| [`bump-and-pr`](../bump-and-pr) | バージョン bump → ブランチ作成 → PR 作成を 1 ステップで行う |
| [`resolve-target-sha`](../resolve-target-sha/action.yml) | publish 対象の SHA を `pull_request` / `workflow_dispatch` の両イベントから解決 |

`@main` を SHA pin に置き換える運用は、利用側のリポジトリで `actions: write` の編集権限を持つメンバーが行ってよい (renovate がアップデート提案を出す)。

### `$default-branch` プレースホルダ

テンプレート内の `$default-branch` は GitHub がテンプレート生成時に利用側リポジトリのデフォルトブランチ名 (通常は `main`) に自動置換する。よってデフォルトブランチが `main` でないリポジトリでもそのまま動く。

### lerna monorepo 前提 (Release / Publish)

`release.yml` と `publish.yml` は lerna monorepo を前提に設計されている。具体的には:

- バージョン管理: `lerna.json` の `version` を `jq -r .version lerna.json` で取得
- バージョン bump: `pnpm packages:bump-version <semver>` (利用側で `lerna version` をラップするスクリプトを `packages:bump-version` として定義しておく)
- 公開: `pnpm packages:publish` (利用側で `lerna publish from-package` 等をラップする)

単一パッケージ (`package.json` の `version` を使う) の場合は、生成後のワークフロー内で `version-command` と `bump-command` を書き換える必要がある。

### npm Trusted Publishing 前提 (Publish)

`publish.yml` の npm 公開ステップは npm の Trusted Publishing (OIDC) を使う前提で書かれている。利用側のパッケージごとに、npm 側で「Trusted Publisher」として **このリポジトリの `publish` ワークフロー** を登録しておく必要がある。詳しくは [npm Trusted Publishing](https://docs.npmjs.com/trusted-publishers) を参照。

従来のトークン方式 (`NODE_AUTH_TOKEN` + `.npmrc`) に切り替える場合は、生成されたワークフローに `setup-node` の `registry-url` 設定と `NODE_AUTH_TOKEN` の受け渡しを追加する。

## テンプレート個別の補足

### `ci.yml`

- 起動: 全ブランチ push (タグ push は `tags-ignore: '**'` で除外)
- 編集ポイント: `pnpm lint` / `pnpm test` 行はリポジトリの構成に合わせて編集する (例: `pnpm -r lint` / `pnpm -r test`、または不要な行を削除)。
- 権限: `contents: write` (format アクションが差分を自動コミットして push するため)。

### `release.yml`

- 起動: `workflow_dispatch` のみ。実行時に `patch` / `minor` / `major` から選ぶ。
- 出力: `release` ブランチに bump コミットを作成し、デフォルトブランチに向けた PR を開く。PR タイトルは `chore(release): vX.Y.Z`、本文には前タグからの compare URL を含む。
- 編集ポイント: ブランチ名・コミットプレフィックスを変えたい場合は `env.RELEASE_BRANCH` / `env.RELEASE_COMMIT_PREFIX` を書き換える。
- 失敗時クリーンアップ: 既存の release PR をクローズし、`release` ブランチを削除する (ベストエフォート)。

### `publish.yml`

- 起動: リリース PR (head=`release`、タイトル=`chore(release): ...`、fork 由来でない) のマージ、または手動 (`workflow_dispatch`)。
- 流れ: check ジョブで対象 SHA を解決してタグ存在を確認 → 既存タグなら以降スキップ → publish ジョブで該当 SHA を checkout して npm 公開 → タグ作成 → alpha バンプ PR を作成。
- 編集ポイント: ブランチ名は `env.RELEASE_BRANCH` / `env.PRERELEASE_BRANCH`、コミットプレフィックスは `env.RELEASE_COMMIT_PREFIX` で一括管理。pre-release identifier (現状 `alpha`) を beta / rc に変える場合は `bump-command` の `--preid` と `commit-prefix`、`pr-body`、DIST_TAG 判定の `*-alpha.*` を併せて書き換える。
- 権限: `check` は `contents: read` のみ、`publish` は `contents: write` / `id-token: write` (Trusted Publishing) / `pull-requests: write` (alpha PR 作成)。
