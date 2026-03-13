# GitHub Projects Status Sandbox

GitHub Projects v2 の `Status` 自動更新を、自分の検証用 Project と自分用 repo で試すための sandbox です。

このディレクトリは、本番向けテンプレートの [github-projects-status-updater](/Users/ryotaroinagaki/my-app/github-projects-status-updater) とは分けて使います。  
目的は、チームの repo に影響を出さずに、`pull_request` / `pull_request_review` のイベントで自動遷移が動くかを確認することです。

## 構成

- [README.md](/Users/ryotaroinagaki/my-app/github-projects-status-sandbox/README.md)
- [.github/workflows/project-status-sandbox.yml](/Users/ryotaroinagaki/my-app/github-projects-status-sandbox/.github/workflows/project-status-sandbox.yml)
- [demo-change.txt](/Users/ryotaroinagaki/my-app/github-projects-status-sandbox/demo-change.txt)

## 使い方

1. 自分用の GitHub Project を作る
2. Project の `Status` に次を作る
3. 検証用 repo に workflow を置く
4. `PROJECTS_TOKEN` を secret に追加する
5. PR を Project に入れた状態でイベントを発火させて確認する

必要な `Status` option:

- `作業中`
- `1次レビュー中`
- `最終レビュー中`
- `最終レビュー対応中`
- `マージ待ち`
- `完了`

PR 作成時には、PR 作成者が自動で assignee に追加されます。

## workflow の設定値

[.github/workflows/project-status-sandbox.yml](/Users/ryotaroinagaki/my-app/github-projects-status-sandbox/.github/workflows/project-status-sandbox.yml) の `env` を書き換えます。

- `PROJECT_OWNER_LOGIN`
- `PROJECT_NUMBER`
- `FINAL_REVIEWER_LOGIN`
- `STATUS_FIELD_NAME`
- 各ステータス名

`PROJECT_OWNER_LOGIN` は organization / user のどちらでも使えます。workflow 側で両方を見に行きます。

## secret

- `PROJECTS_TOKEN`

必要条件:

- GitHub Projects v2 を更新できる token であること
- `Status` field が single select field であること

## 2つの GitHub でできるか

できます。  
ただし、何を 2 つ持っているかで確認できる範囲が変わります。

- 2つの repo がある: 十分です。1つを sandbox repo にすればよく、もう 1 つは必須ではありません
- 2つの GitHub アカウントがある: 最終レビューループの検証には十分です

おすすめの役割:

- Account A: repo owner / PR 作成者
- Account B: 最終レビュワー

この 2 アカウント構成で確認しやすいもの:

- `作業中`
- `最終レビュー中`
- `最終レビュー対応中`
- `マージ待ち`
- `完了`

`1次レビュー中` も厳密に試したい場合は、最終レビュワーとは別の一般レビュワーが 1 人必要です。  
つまり、その状態まで含めて全部確認するなら 3 人目がいる方が楽です。

## 検証シナリオ

1. PR を作成して `作業中` になる
2. PR 作成者が assignee に入る
3. 一般レビュワーを request して `1次レビュー中` になる
4. 最終レビュワーを request して `最終レビュー中` になる
5. 最終レビュワーが `changes_requested` を返して `最終レビュー対応中` になる
6. 修正後に最終レビュワーへ再 request して `最終レビュー中` に戻る
7. 最終レビュワーが `approved` を返して `マージ待ち` になる
8. merge して `完了` になる

補足:

- push だけではステータスを戻しません
- team reviewer の request は対象外です
- PR が Project に入っていない場合、workflow は no-op で終了します

## 実際のテスト手順

### 1. 準備

- sandbox 用の repo を GitHub 上に作る
- sandbox 用の Project を GitHub 上に作る
- Project の `Status` に必要な option を追加する
- sandbox repo を Project に追加対象にする
- 必要なら reviewer に使うアカウントを sandbox repo の collaborator に入れる

### 2. sandbox を置く

- このディレクトリの中身を sandbox repo に置く
- [.github/workflows/project-status-sandbox.yml](/Users/ryotaroinagaki/my-app/github-projects-status-sandbox/.github/workflows/project-status-sandbox.yml) の `env` を自分用に書き換える
- repo secret に `PROJECTS_TOKEN` を追加する

書き換える値:

- `PROJECT_OWNER_LOGIN`: 検証用 Project の owner login
- `PROJECT_NUMBER`: 検証用 Project の number
- `FINAL_REVIEWER_LOGIN`: 最終レビュワーにする login。sandbox の既定値は `Ryotaro-Inagaki`

### 3. PR を Project に入れる

- Project 側の自動追加を使うか、手動で PR を Project に入れる
- この workflow は PR が Project に入っていないと更新しません

### 4. 最小テスト

- `demo-change.txt` を編集してブランチを切る
- PR を作成する
- PR の assignee に作成者自身が自動追加されることを確認する
- Actions が走った後、Project の Status が `作業中` になることを確認する

### 5. 最終レビューのループを試す

- 最終レビュワーを request して `最終レビュー中` になることを確認する
- 最終レビュワーが `changes_requested` を返して `最終レビュー対応中` になることを確認する
- PR 作成者が修正して push する
- その時点ではステータスが変わらないことを確認する
- 最終レビュワーに再度 request して `最終レビュー中` に戻ることを確認する
- 最終レビュワーが `approved` を返して `マージ待ち` になることを確認する
- merge して `完了` になることを確認する

### 6. １次レビューも試す

- 最終レビュワーではない一般レビュワーを request する
- Project の Status が `1次レビュー中` になることを確認する

## 検証のコツ

- まずは 2 アカウントで最終レビューループを通す
- `1次レビュー中` は一般レビュワーが確保できた時に追加で確認する
- 最初の 1 回は `demo-change.txt` だけを変更する小さい PR にする
- workflow が動かない時は、PR が Project に入っているかを最初に確認する
