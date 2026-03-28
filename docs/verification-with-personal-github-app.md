# GitHub Projects ステータス更新ワークフロー: 検証用の個人所有 GitHub App 作成手順

## 概要

この文書は、`github-projects-status-sandbox` ディレクトリで動作確認するために、**個人所有の GitHub App** を使って検証する手順をまとめたもの。

- 目的は検証と作業メモ
- 本番運用は [migration-to-github-app.md](/Users/ryotaroinagaki/my-app/github-projects-status-sandbox/docs/migration-to-github-app.md) の **Organization 所有 GitHub App** を使う
- 個人所有の GitHub App は検証用途に限定し、本番では使わない

## この手順の位置づけ

本番手順との違いは **GitHub App の所有者が個人アカウント** である点だけで、ProjectV2 を更新するための考え方自体は同じ。

- 対象 Project が Organization 配下なら、**個人所有の GitHub App を Organization にインストール**して検証する
- 対象 Project が個人アカウント配下なら、このリポジトリの本来の運用条件とは異なるため、検証結果をそのまま本番判断に使わない
- GitHub App の webhook は使わない
- GitHub Actions から `installation access token` を生成して ProjectV2 API を呼び出す

## 前提条件

- 検証対象の Project が **Organization の ProjectV2** であること
- 個人アカウントで GitHub App を作成できること
- 対象 Organization に GitHub App をインストールできること
  - 必要なら Organization owner に承認してもらう
- 対象 repository で Actions を実行できること

## 1. 個人アカウントで GitHub App を作成

1. GitHub の **Settings** > **Developer settings** > **GitHub Apps** > **New GitHub App** を開く
2. 以下を設定する:
   - **GitHub App name**: 任意の名前
     - 例: `project-status-sandbox-personal`
   - **Homepage URL**: このリポジトリの URL 等
   - **Webhook**: Active の**チェックを外す**
3. **Permissions** を以下の通り設定する

### Repository permissions

| 権限 | レベル |
|---|---|
| Metadata | Read-only（自動付与） |

### Organization permissions

| 権限 | レベル |
|---|---|
| Projects | Read and write |

4. **Where can this GitHub App be installed?** では、Organization に入れて検証できるように **Any account** を選ぶ
5. **Create GitHub App** をクリック

### 補足

- この検証では `Projects` 権限が重要
- PR assignee の更新は `GITHUB_TOKEN` を使う想定なので、GitHub App に `Issues` や `Pull requests` の repository permission は付けない
- `workflow_dispatch` や `repository_dispatch` を GitHub App から呼ぶわけではないため、`Actions (write)` や `Contents (write)` は不要

## 2. 秘密鍵を生成

1. 作成した GitHub App の設定ページを開く
2. **General** > **Private keys** で **Generate a private key** をクリック
3. ダウンロードされた `.pem` を安全に保管する
4. 画面上の **App ID** を控える

## 3. GitHub App を検証先にインストール

### Organization Project を使って検証する場合

1. GitHub App の設定ページで **Install App** を開く
2. 対象 Organization を選択する
3. 対象 repository を含めてインストールする
4. 権限確認が表示されたら承認する

このパターンが、このリポジトリでの検証では基本となる。

### 個人アカウントにだけインストールする場合

- 個人所有の ProjectV2 を触る検証はできる
- ただし Organization ProjectV2 を前提にした本番運用の代替検証にはならない

## 4. 検証用 Secrets を設定

検証用と本番用を混同しないよう、Secrets 名は分ける。

対象リポジトリの **Settings** > **Secrets and variables** > **Actions** で以下を登録する:

| Secret 名 | 値 |
|---|---|
| `SANDBOX_APP_ID` | 検証用 GitHub App の App ID |
| `SANDBOX_APP_PRIVATE_KEY` | `.pem` ファイルの内容（全文） |

本番用の `APP_ID` / `APP_PRIVATE_KEY` を使い回さない。

## 5. ワークフローに検証用 Secrets を渡す

検証時は、`actions/create-github-app-token` に渡す値を検証用 Secrets に切り替える。

```yaml
      - name: Generate GitHub App token
        id: generate-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.SANDBOX_APP_ID }}
          private-key: ${{ secrets.SANDBOX_APP_PRIVATE_KEY }}
          owner: ${{ github.repository_owner }}
```

注記:

- `owner` は、検証対象の Project が属している Organization 名になる前提
- 個人所有の GitHub App でも、Organization にインストールされていれば installation token で Organization ProjectV2 を更新できる
- 対象 repository に GitHub App が入っていないと失敗する

## 6. 動作確認

以下を順に確認する:

- [ ] `actions/create-github-app-token` で token 生成に成功する
- [ ] ProjectV2 API 呼び出しが 403 にならない
- [ ] PR 新規作成時に assignee 設定が `GITHUB_TOKEN` で成功する
- [ ] Project のステータス更新が期待通りに行われる
- [ ] 対象 GitHub App の owner が個人アカウントであることを認識したうえで、Organization にインストールされている

## 7. よくある失敗要因

### 1. `Only on this account` を選んでしまい、Organization にインストールできない

- 検証対象が Organization ProjectV2 なら、Organization にインストールできる設定が必要
- 個人アカウントにしか入れられない状態だと、今回の検証用途に合わない

### 2. GitHub App は作れたが、Organization へのインストール承認が取れていない

- 個人所有の GitHub App を Organization に入れる場合、Organization owner 側の承認が必要になることがある

### 3. `Projects` 権限を付けた後に installation 側で再承認していない

- 後から権限を増やした場合は、installation 側で **Approve new permissions** が必要

### 4. GitHub App を Organization ではなく個人アカウントにだけインストールしている

- 個人インストールだけでは、Organization ProjectV2 の更新に必要なアクセスが満たせない

### 5. 本番用 Secrets と検証用 Secrets が混ざっている

- `APP_ID` と `SANDBOX_APP_ID` を混同すると、どの App で動いているか分からなくなる
- 検証中は Secrets 名を明確に分ける

## 8. 検証が終わったら

- 検証結果を踏まえて、本番は [migration-to-github-app.md](/Users/ryotaroinagaki/my-app/github-projects-status-sandbox/docs/migration-to-github-app.md) の Organization 所有 GitHub App に寄せる
- 不要になった個人所有 GitHub App は削除するか、少なくとも利用停止状態にする
- 検証用 Secrets は削除または無効化する
