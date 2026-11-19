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
| [`format`](../format/action.yml) | `pnpm format` を実行して差分を自動コミット。push したかどうかを出力 `pushed` で返す |
| [`bump-and-pr`](../bump-and-pr) | バージョン bump → ブランチ作成 → PR 作成を 1 ステップで行う |
| [`resolve-target-sha`](../resolve-target-sha/action.yml) | publish 対象の SHA を `pull_request` / `workflow_dispatch` の両イベントから解決 |

`@main` を SHA pin に置き換える運用は、利用側のリポジトリで `actions: write` の編集権限を持つメンバーが行ってよい (renovate がアップデート提案を出す)。

### `$default-branch` プレースホルダ

テンプレート内の `$default-branch` は GitHub がテンプレート生成時に利用側リポジトリのデフォルトブランチ名 (通常は `main`) に自動置換する。よってデフォルトブランチが `main` でないリポジトリでもそのまま動く。

### 自動コミット後の必須チェック (`workflow_dispatch`)

`format` アクションと `bump-and-pr` アクションは `GITHUB_TOKEN` でコミット・push・PR 作成を行う。`GITHUB_TOKEN` によるイベントでは他のワークフローが起動しない (無限ループ防止のための GitHub の仕様) ため、自動コミットには CI が走らない。ruleset の必須チェックを設定しているリポジトリでは、自動コミットが HEAD のブランチや PR がマージできなくなる。

そこで各テンプレートは、push した直後に必須チェックを担う CI を `workflow_dispatch` で明示的に起動する。`workflow_dispatch` は `GITHUB_TOKEN` から呼んでも起動する例外として扱われる。GitHub App や PAT、追加の secret は不要。

| テンプレート | dispatch するジョブ | 起動するタイミング | 起動するワークフロー |
| ------------ | ------------------- | ------------------ | -------------------- |
| `ci.yml` | `dispatch` | `ci` ジョブの `format` がフォーマット差分をコミットして push したとき | 実行中のワークフロー自身 |
| `release.yml` | `dispatch-ci` | `release` ジョブが `release` ブランチの PR を作成したあと | `ci.yml` |
| `publish.yml` | `dispatch-ci` | `publish` ジョブが `alpha` ブランチの PR を作成したあと | `ci.yml` |

#### dispatch を別ジョブに分けている理由

dispatch には `actions: write` が必要だが、この権限は `pnpm install` / `lint` / `test` / `format` など依存パッケージのコードを実行するジョブには渡さない。依存パッケージが侵害された場合に、`contents: write` で用意したブランチに対して任意のワークフロー (例: `id-token: write` を持つ `publish.yml`) を起動できてしまうため。

各テンプレートでは、checkout も install もせず `gh workflow run` だけを実行するジョブに `actions: write` だけを渡す。起動先とブランチはワークフロー内で固定しているので、前段のジョブから操作できるのは「dispatch するかどうか」だけになる。`dispatch` / `dispatch-ci` ジョブに処理を追加するときも、この性質を崩さないこと。

#### 利用側のリポジトリでの注意

- **必須チェック名**: ruleset の必須チェックにはジョブ名を指定する。CI テンプレートではジョブ名 `ci` がチェック名になる。`workflow_dispatch` で起動した実行のチェックも、push で起動した実行と同じ名前でコミットに付く。`dispatch` ジョブは必須チェックに含めない (dispatch しない実行ではスキップされる)。
- **`workflow_dispatch` トリガー**: dispatch 先のワークフローの `on:` に `workflow_dispatch:` が必要。CI テンプレートには追加済み。CI のワークフローファイル名を `ci.yml` から変えた場合は、`release.yml` / `publish.yml` の `dispatch-ci` ジョブの `CI_WORKFLOW` も合わせて書き換える。
- **`actions: write` 権限**: dispatch するジョブだけに `actions: write` を渡す。各テンプレートの `dispatch` / `dispatch-ci` ジョブに設定済み。
- **`format` の出力 `pushed`**: `format` アクション自体は dispatch しない。CI テンプレートでは `ci` ジョブの出力として `pushed` を受け渡し、`dispatch` ジョブがそれを見て起動する。既存のワークフローで `format` の SHA pin を更新するだけでは dispatch されないので、CI テンプレートに合わせて `dispatch` ジョブなどを追加する。
- **無限ループ防止**: `dispatch` ジョブは `workflow_dispatch` で起動された実行では動かない。dispatch された CI の実行で `format` がさらに差分をコミットした場合、そのコミットには CI が走らない。`release` / `alpha` ブランチで起きないよう、bump 後のファイルがフォーマット済みになるようにしておく (必要なら `bump-and-pr` の `pre-commit-command` で `pnpm format` を実行する)。
- **フォークからの PR は対象外**: CI テンプレートは `push` で起動するため、フォークからの PR では本体リポジトリの CI が走らず、自動コミットも dispatch も行われない。フォークのブランチには `GITHUB_TOKEN` で push できず、`workflow_dispatch` で起動することもできない。
- **コミットメッセージの `[skip ci]`**: `format` の自動コミットメッセージ `format [skip ci]` は push / pull_request イベントの起動を抑止するだけで、`workflow_dispatch` には影響しない。

### lerna monorepo 前提 (Release / Publish)

`release.yml` と `publish.yml` は lerna monorepo を前提に設計されている。具体的には:

- バージョン管理: `lerna.json` の `version` を `jq -r .version lerna.json` で取得
- バージョン bump: `pnpm packages:bump-version <semver>` (利用側で `lerna version` をラップするスクリプトを `packages:bump-version` として定義しておく)
- 公開: `pnpm packages:publish` (利用側で `lerna publish from-package` 等をラップする)

単一パッケージ (`package.json` の `version` を使う) の場合は、生成後のワークフロー内で `version-command` と `bump-command` を書き換える必要がある。

### npm Trusted Publishing 前提 (Publish)

`publish.yml` の npm 公開ステップは npm の Trusted Publishing (OIDC) を使う前提で書かれている。利用側のパッケージごとに、npm 側で「Trusted Publisher」として **このリポジトリの `publish` ワークフロー** を登録しておく必要がある。詳しくは [npm Trusted Publishing](https://docs.npmjs.com/trusted-publishers) を参照。

`publish.yml` を `workflow_dispatch` で起動できるのはデフォルトブランチからだけに制限している (`check` ジョブの `if:`)。任意のブランチで起動されると、`publish` ジョブがそのブランチの内容で `pnpm install` を実行してから公開に進むため、そのブランチに仕込まれた依存パッケージやスクリプトが `id-token: write` の権限で動いてしまう。

さらに固める場合は、GitHub の environment を作って `publish` ジョブに `environment:` で指定し、npm 側の Trusted Publisher 設定でその environment を必須にする。environment のデプロイ先ブランチを制限するときは、リリース PR のマージ (`pull_request` イベント) で起動したジョブは `refs/pull/<番号>/merge` に対して判定される点に注意する。デフォルトブランチだけに制限すると、通常の公開も止まる。

従来のトークン方式 (`NODE_AUTH_TOKEN` + `.npmrc`) に切り替える場合は、生成されたワークフローに `setup-node` の `registry-url` 設定と `NODE_AUTH_TOKEN` の受け渡しを追加する。

## テンプレート個別の補足

### `ci.yml`

- 起動: 全ブランチ push (タグ push は `tags-ignore: '**'` で除外)、および `workflow_dispatch` (format アクションの自動コミット後の再起動、`release.yml` / `publish.yml` からの起動、手動実行)。
- 編集ポイント: `pnpm lint` / `pnpm test` 行はリポジトリの構成に合わせて編集する (例: `pnpm -r lint` / `pnpm -r test`、または不要な行を削除)。
- 権限: `ci` ジョブは `contents: write` (format アクションが差分を自動コミットして push するため)。`dispatch` ジョブは `actions: write` のみ (自動コミット後にこのワークフロー自身を `workflow_dispatch` で起動するため)。
- 同時実行: 同じブランチの実行は 1 つにまとめる。push で起動した実行は、同じブランチで進行中の実行をキャンセルする。`workflow_dispatch` で起動した実行は進行中の実行をキャンセルせず、終了を待ってから始まる。`dispatch` ジョブが dispatch した直後に、dispatch 元の実行がキャンセル扱いになるのを避けるため。dispatch された実行は、その後の push による実行にキャンセルされる (古いコミットに対する実行のため問題ない)。

### `release.yml`

- 起動: `workflow_dispatch` のみ。実行時に `patch` / `minor` / `major` から選ぶ。
- 出力: `release` ブランチに bump コミットを作成し、デフォルトブランチに向けた PR を開く。PR タイトルは `chore(release): vX.Y.Z`、本文には前タグからの compare URL を含む。
- 編集ポイント: ブランチ名 (`release`) とコミットプレフィックス (`chore(release)`) を変えたい場合は yaml 内の該当箇所を一括で書き換える。job-level の `if:` では `env` context が使えないため、テンプレートでは `env` で集約せずハードコードしている (置換漏れに注意)。
- 失敗時クリーンアップ: 既存の release PR をクローズし、`release` ブランチを削除する (ベストエフォート)。
- 権限: `release` ジョブは `contents: write` (bump コミットの push) / `pull-requests: write` (リリース PR 作成)。`dispatch-ci` ジョブは `actions: write` のみ (PR 作成後に `ci.yml` を `workflow_dispatch` で起動)。
- 必須チェック: `release` ジョブが成功したら、`dispatch-ci` ジョブが `ci.yml` を `release` ブランチに対して起動する。CI のワークフローファイル名が異なる場合は `CI_WORKFLOW` を、ブランチ名を変える場合は `--ref` も書き換える。

### `publish.yml`

- 起動: リリース PR (head=`release`、タイトル=`chore(release): ...`、fork 由来でない) のマージ、または手動 (`workflow_dispatch`、デフォルトブランチからのみ)。
- 流れ: check ジョブで対象 SHA を解決してタグ存在を確認 → 既存タグなら以降スキップ → publish ジョブで該当 SHA を checkout して npm 公開 → タグ作成 → alpha バンプ PR を作成。
- 編集ポイント: ブランチ名 (`release` / `alpha`) とコミットプレフィックス (`chore(release)`) を変えたい場合は yaml 内の該当箇所を一括で書き換える。job-level の `if:` では `env` context が使えないため、テンプレートでは `env` で集約せずハードコードしている (置換漏れに注意)。pre-release identifier (現状 `alpha`) を beta / rc に変える場合は `bump-command` の `--preid` と `commit-prefix`、`pr-body`、DIST_TAG 判定の `*-alpha.*` も併せて書き換える。
- 権限: `check` は `contents: read` のみ、`publish` は `contents: write` / `id-token: write` (Trusted Publishing) / `pull-requests: write` (alpha PR 作成)、`dispatch-ci` は `actions: write` のみ (alpha PR 作成後に `ci.yml` を `workflow_dispatch` で起動)。
- 必須チェック: `publish` ジョブが成功したら、`dispatch-ci` ジョブが `ci.yml` を `alpha` ブランチに対して起動する。タグが既に存在して `publish` がスキップされた場合は起動しない。CI のワークフローファイル名が異なる場合は `CI_WORKFLOW` を、ブランチ名を変える場合は `--ref` も書き換える。
