# GitHub Projects ステータス更新ワークフロー: PAT から Organization 所有の GitHub App への移行手順

## 概要

GitHub Actions ワークフロー `project-status-sandbox.yml` の認証方式を、個人アクセストークン (PAT) から GitHub App に変更する。

検証用に個人所有の GitHub App を使う場合は、[verification-with-personal-github-app.md](/Users/ryotaroinagaki/my-app/github-projects-status-sandbox/docs/verification-with-personal-github-app.md) を参照すること。この文書は本番運用を前提に、Organization 所有の GitHub App を作成する手順をまとめている。

## 変更の目的

- PAT は個人アカウントに紐づくため、そのユーザーが組織を離れるとトークンが無効になるリスクがある
- GitHub App を Organization 所有にすると、特定個人への依存をさらに減らせる
- 権限スコープを最小限に絞れる
- `installation access token` を使うことで、個人の PAT に依存せずに ProjectV2 API を利用できる

## 前提条件

- 対象プロジェクトが **Organization のプロジェクト**であること
  - 個人アカウントの ProjectV2 は GitHub App の権限スコープに含まれないため、PAT が必要
- Organization の管理者権限を持つメンバーがいること
- GitHub App は **Organization 所有**で作成すること
  - 個人所有の GitHub App でも技術的には動作するが、退職やアカウント整理の影響を受けやすいため非推奨
  - すでに個人所有の GitHub App を使っている場合は、先に Organization へ ownership transfer する
- この手順では **GitHub App の webhook は使わない**
  - GitHub Actions から GitHub App トークンを生成して ProjectV2 API を呼び出す構成
  - `workflow_dispatch` や `repository_dispatch` を GitHub App から実行する用途とは必要権限が異なる

## 0. GitHub App の所有者を確認する

新規作成する場合は、Organization の **Settings** > **Developer settings** > **GitHub Apps** から登録する。

既存の GitHub App を流用する場合は、以下を確認する:

1. GitHub App の所有者が Organization になっているか
2. 管理者が 1 人に偏っていないか
3. 必要なら GitHub App managers を追加する

個人所有の GitHub App を使っている場合は、移行前に ownership transfer を実施する。

## 1. GitHub App の作成

1. GitHub の **Organization Settings** > **Developer settings** > **GitHub Apps** > **New GitHub App** を開く
2. 以下を設定する:
   - **GitHub App name**: 任意の名前（例: `project-status-updater`）
   - **Homepage URL**: リポジトリの URL 等
   - **Webhook**: Active の**チェックを外す**（Webhook は不要）
3. **Permissions** を以下の通り設定する:

### Repository permissions

| 権限 | レベル |
|---|---|
| Metadata | Read-only（自動付与） |

### Organization permissions

| 権限 | レベル |
|---|---|
| Projects | Read and write |

4. **Where can this GitHub App be installed?** で「Only on this account」を選択
5. **Create GitHub App** をクリック

### 権限に関する補足

- この移行で GitHub App に必要なのは、ProjectV2 を更新するための `Organization permissions: Projects (Read and write)` が中心
- PR の assignee 変更は GitHub App ではなく `GITHUB_TOKEN` で実行するため、GitHub App に `Issues` や `Pull requests` の repository permission は不要
- `workflow_dispatch` や `repository_dispatch` を GitHub App から実行したい場合は別途権限が必要
  - `workflow_dispatch`: `Actions (write)`
  - `repository_dispatch`: `Contents (write)`

## 2. 秘密鍵の生成

1. 作成した GitHub App の設定ページを開く
2. **General** > **Private keys** セクションで **Generate a private key** をクリック
3. `.pem` ファイルがダウンロードされるので安全に保管する
4. App の設定ページに表示されている **App ID** を控える

## 3. GitHub App を Organization にインストール

1. GitHub App の設定ページで **Install App** を開く
2. 対象の Organization を選択
3. リポジトリのアクセス範囲を選択（対象リポジトリのみ、または全リポジトリ）
4. **Install** をクリック
5. 権限の確認画面が表示された場合は承認する

### 重要: 権限変更後は再承認が必要

既存の GitHub App に後から permission を追加した場合、GitHub App の設定を保存しただけでは新権限は有効にならない。各 installation 側で **Approve new permissions** が必要。

権限を修正したのに 403 や想定外の動作が続く場合は、まず installation 側で新権限が承認済みか確認する。

## 4. リポジトリの Secrets を設定

対象リポジトリの **Settings** > **Secrets and variables** > **Actions** で以下を登録する:

| Secret 名 | 値 |
|---|---|
| `APP_ID` | GitHub App の App ID |
| `APP_PRIVATE_KEY` | `.pem` ファイルの内容（全文） |

既存の `PROJECTS_TOKEN`（PAT）は移行完了後に削除する。

## 5. ワークフローの変更

### 変更点の概要

| 項目 | 変更前 (PAT) | 変更後 (GitHub App) |
|---|---|---|
| PR assignee 設定 | PAT (`secrets.PROJECTS_TOKEN`) | デフォルトの `GITHUB_TOKEN` |
| Projects ステータス更新 | PAT (`secrets.PROJECTS_TOKEN`) | GitHub App トークン |
| ステップ数 | 1 ステップ | 3 ステップ（assignee / トークン生成 / ステータス更新） |

### 変更内容

#### 5-1. job に permissions を追加

assignee 設定にデフォルトの `GITHUB_TOKEN` を使うため、job レベルで権限を付与する。

```yaml
jobs:
  update-status:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      issues: write
```

#### 5-2. Assignee 設定ステップを分離

元のスクリプト内にあった `shouldAssignPullRequestAuthor` / `assignPullRequestAuthor` を独立したステップとして切り出す。このステップはデフォルトの `GITHUB_TOKEN` で実行する（`github-token` の指定不要）。

```yaml
    steps:
      - name: Assign pull request author
        if: github.event_name == 'pull_request' && (github.event.action == 'opened' || github.event.action == 'reopened')
        uses: actions/github-script@v7
        with:
          script: |
            const payload = context.payload;
            const owner = payload.repository?.owner?.login;
            const repo = payload.repository?.name;
            const issueNumber = payload.pull_request?.number;
            const authorLogin = payload.pull_request?.user?.login;
            const assignees = (payload.pull_request?.assignees ?? [])
              .map((assignee) => assignee?.login)
              .filter(Boolean);

            if (!owner || !repo || !issueNumber || !authorLogin) {
              core.setFailed("Missing repository or pull request data required to assign the author.");
              return;
            }

            if (assignees.includes(authorLogin)) {
              core.info(`Pull request author ${authorLogin} is already assigned.`);
              return;
            }

            await github.rest.issues.addAssignees({
              owner,
              repo,
              issue_number: issueNumber,
              assignees: [authorLogin],
            });

            core.info(`Assigned pull request author ${authorLogin} to PR #${issueNumber}.`);
```

#### 5-3. GitHub App トークン生成ステップを追加

`actions/create-github-app-token@v1` を使い、GitHub App のインストールトークンを生成する。`owner` を指定することで Organization スコープのトークンを取得し、ProjectV2 API にアクセスできるようにする。

```yaml
      - name: Generate GitHub App token
        id: generate-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          owner: ${{ github.repository_owner }}
```

注記:

- `owner` は Organization 名になることを前提とする
- このトークンは GitHub App の installation token であり、作成者個人の権限には依存しない
- 対象 repository に GitHub App がインストールされていないと、トークン生成や後続 API 呼び出しが失敗することがある

#### 5-4. ステータス更新ステップのトークンを変更

`github-token` を PAT から GitHub App トークンに変更する。スクリプト本体からは `shouldAssignPullRequestAuthor` / `assignPullRequestAuthor` 関連のコードを削除する。

```yaml
      - name: Update project status
        uses: actions/github-script@v7
        with:
          github-token: ${{ steps.generate-token.outputs.token }}
          script: |
            # (スクリプト本体 — assignee 関連コードを除いた既存ロジック)
```

## 6. 動作確認

以下のイベントで正しくステータスが更新されることを確認する:

- [ ] PR を新規作成 → assignee が設定される / ステータスが「作業中」になる
- [ ] PR を reopen → assignee が設定される / ステータスが「作業中」になる
- [ ] レビューをリクエスト → ステータスが「1次レビュー中」または「最終レビュー中」になる
- [ ] 最終レビュアーが changes_requested → ステータスが「最終レビュー対応中」になる
- [ ] 最終レビュアーが approve → ステータスが「マージ待ち」になる
- [ ] PR をマージ → ステータスが「完了」になる

あわせて以下も確認する:

- [ ] GitHub App の owner が Organization になっている
- [ ] GitHub App が対象 repository にインストールされている
- [ ] `Projects` permission が installation 側で承認済みになっている
- [ ] 対象 Project が Organization 配下の ProjectV2 である
- [ ] `actions/create-github-app-token` で生成したトークンを使って Project 更新が成功している

## 7. クリーンアップ

動作確認が完了したら:

1. リポジトリの Secrets から `PROJECTS_TOKEN`（PAT）を削除
2. GitHub の **Settings** > **Developer settings** > **Personal access tokens** から該当 PAT を revoke

## 8. よくある失敗要因

### 1. GitHub App が個人所有のままになっている

- そのままでも一見動くことはあるが、作成者の退職やアカウント整理時に運用リスクが残る
- 原則として Organization 所有に寄せる

### 2. 権限を追加したが installation 側で再承認していない

- GitHub App の設定画面で permission を追加しても、自動では有効にならない
- `Projects` 権限を追加した後は **Approve new permissions** を忘れない

### 3. GitHub App が対象 repository に入っていない

- `Only selected repositories` の場合、対象 repo が installation 対象外だと失敗する
- App は存在していても、その repo で token を使えない

### 4. 対象 Project が個人アカウント配下である

- 本手順の GitHub App 構成は Organization ProjectV2 前提
- 個人 ProjectV2 を更新する用途ではそのまま使えない

### 5. GitHub App に不要な権限を付けすぎている

- この手順では webhook 受信や workflow の dispatch 実行は不要
- `Actions (write)` や `Contents (write)` は、別用途で明示的に必要になった場合のみ追加する
