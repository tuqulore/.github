# Bump and PR

パッケージのバージョンを専用ブランチでバンプし、プルリクエストを作成する複合アクション。

lerna / pnpm / npm / yarn / changesets など、具体的なツールには依存しない。呼び出し側がバンプコマンドとバージョン取得コマンドを渡す。

## 使い方

### 前提

- リポジトリがチェックアウト済みであること
- git の user.name / user.email が設定済みであること（`tuqulore/.github/configure-git` を事前に呼ぶ）
- `gh` CLI が利用可能であること（GitHub-hosted runner には標準でインストール済み）
- workflow に `contents: write` と `pull-requests: write` 権限があること
- `branch` に指定するブランチがリモートに存在しないこと（同名ブランチが残っている場合は事前に削除）

### 最小例（npm の単一パッケージ）

```yaml
- uses: tuqulore/.github/setup@main
- uses: tuqulore/.github/configure-git@main
- uses: tuqulore/.github/bump-and-pr@main
  with:
    branch: release
    bump-command: npm version patch --no-git-tag-version
    commit-prefix: chore(release)
    github-token: ${{ github.token }}
```

### lerna monorepo の例

```yaml
- uses: tuqulore/.github/bump-and-pr@main
  with:
    branch: release
    bump-command: pnpm packages:bump-version patch --yes --no-git-tag-version
    version-command: jq -r .version lerna.json
    commit-prefix: chore(release)
    pr-body: Release v${VERSION}
    github-token: ${{ github.token }}
```

### alpha プレリリースの例

```yaml
- uses: tuqulore/.github/bump-and-pr@main
  with:
    branch: alpha
    bump-command: pnpm packages:bump-version prerelease --preid alpha --yes --no-git-tag-version
    version-command: jq -r .version lerna.json
    commit-prefix: chore(alpha)
    pr-body: Bump to alpha version v${VERSION}
    github-token: ${{ github.token }}
```

### pre-commit-command でテンプレートファイルを更新する例

```yaml
- uses: tuqulore/.github/bump-and-pr@main
  with:
    branch: release
    bump-command: pnpm packages:bump-version patch --yes --no-git-tag-version
    version-command: jq -r .version lerna.json
    commit-prefix: chore(release)
    pre-commit-command: |
      VERSION=$(jq -r .version lerna.json)
      jq --arg v "^$VERSION" '.devDependencies["@tuqulore-inc/eleventy-preset"] = $v' \
        packages/create-eleventy/templates/default/package.json > tmp.json
      mv tmp.json packages/create-eleventy/templates/default/package.json
    github-token: ${{ github.token }}
```

## 入力

| 名前                 | 必須 | デフォルト                                      | 説明                                                                             |
| -------------------- | ---- | ----------------------------------------------- | -------------------------------------------------------------------------------- |
| `branch`             | ✓    | -                                               | 作成して push するブランチ名（例: `release`, `alpha`）                           |
| `bump-command`       | ✓    | -                                               | バージョンをインプレースでバンプするシェルコマンド。タグやコミットは作らないこと |
| `version-command`    |      | `node -p "require('./package.json').version"`   | 現在のバージョンを標準出力に出すシェルコマンド                                   |
| `commit-prefix`      | ✓    | -                                               | コミットメッセージと PR タイトルのプレフィックス（例: `chore(release)`）         |
| `pre-commit-command` |      | `''`                                            | バンプ後・コミット前に実行する任意のシェルコマンド                               |
| `pr-base`            |      | `main`                                          | PR のベースブランチ                                                              |
| `pr-body`            |      | `Bump to v${VERSION}`                           | PR 本文。`${VERSION}` がバンプ後のバージョンに置換される                         |
| `github-token`       | ✓    | -                                               | `gh pr create` で使用するトークン                                                |

## 出力

| 名前      | 説明                                |
| --------- | ----------------------------------- |
| `version` | バンプ後のバージョン（例: `3.0.2`） |
| `pr-url`  | 作成された PR の URL                |

## 処理の流れ

1. `branch` で指定されたブランチを新規作成
2. `bump-command` を実行
3. `pre-commit-command` が指定されていれば実行
4. `version-command` でバージョンを取得
5. 全変更をコミット（メッセージ: `${commit-prefix}: v${VERSION}`）
6. リモートへ push
7. `gh pr create` で PR を作成（タイトル: `${commit-prefix}: v${VERSION}`）

## 注意事項

- `branch` と同名のブランチがリモートに既に存在する場合、push が失敗する。必要なら呼び出し側の workflow で事前にブランチを削除する
- 同じ base / head の組み合わせで open 中の PR が既にある場合、`gh pr create` はエラーになる
- 複合アクションは `post:` ステップを持てないため、失敗時のクリーンアップ（ブランチ削除・PR クローズ）は呼び出し側の workflow で行うこと
