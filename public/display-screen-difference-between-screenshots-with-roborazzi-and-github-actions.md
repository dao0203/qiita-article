---
title: roborazziで取得したスクリーンショットをPRにて表示する
tags:
  - Android
  - GitHub
  - GitHubActions
  - Roborazzi
private: false
updated_at: '2024-02-24T02:28:42+09:00'
id: afebaeb10af0219e29a0
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
こんにちは、こんばんは、佐藤佑哉です。
本日は、Androidアプリ開発で使用されるRoborazziを用いて、画面の差分をPRのコメントに表示するをGitHubActionsで実現していきたいと思います。
## 本記事の動機
前回投稿したroborazziでのスクリーンショットの方なのですが、現状、スクリーンショットと差分レポートの生成のみになっており、実用性に欠けています。これをGitHub Actionsを使用して、実際に開発でも実用できるようにするのが本記事の動機です。

https://qiita.com/dao0203/items/f6f3633b8e8a0ce6c49d

## 使用する技術
本記事で使用するGitHub ActionsとRoborazziのみを使用します。

https://github.co.jp/features/actions

https://github.com/takahirom/roborazzi

## 前回の記事のまとめ
前回まで, roborazziを使用してVRTを行い、生成されたスクリーンショットや差分レポートを表示するところまで行いました。今回はそのスクリーンショットを用いて、PRに表示していきたいと思います。


https://github.com/dao0203/todo-sample-app

## スクリーンショットを比較してPRに表示するまでの流れ
今回は、[takahirom](https://github.com/takahirom)さんが作成した[サンプルプロジェクト](https://github.com/takahirom/roborazzi-compare-on-github-comment-sample)workflowを参考にして、作成していきたいと思います。

下記のイメージが今回の実装の構成図になります。

![githubactions構成図.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2989029/1e8fb531-cc89-02ef-a8e8-ae54399eb179.png)

### 変更前のスクリーンショットを成果物として保存する
まずはじめに、mainブランチから変更する前のスクリーンショットを作成し、成果物として保存します。

```yml:store-screenshot.yml
name: StoreScreenshot

on:
  push:
    branches:
      - main

jobs:
  store-screenshot-test:
    runs-on: ubuntu-latest
    timeout-minutes: 20

    permissions:
      contents: read # for checkout
      actions: write # for upload-artifact

    steps:
      ## Android開発の環境をセットアップは省略
      ## ...

      - name: record screenshot
        run: |
          ./gradlew recordRoborazziDebug --stacktrace --rerun-tasks

      - uses: actions/upload-artifact@v4
        if: ${{ always() }}
        with:
          name: screenshots
          path: |
            actual_images/ # 変更前のスクリーンショットを保存するディレクトリ
          retention-days: 30
```

### StoreScreenshotワークフローでアップロードしたスクリーンショットと変更後のスクリーンショットを比較し、生成された差分を成果物として保存する
次にmeinブランチから変更前のスクリーンショットを取得し、変更後のスクリーンショットと比較して、差分を生成します。


```yml:compare-screenshot.yml
name: CompareScreenshot

on:
  pull_request:

jobs:
  compare-screenshot-test:
    runs-on: ubuntu-latest
    timeout-minutes: 20

    permissions:
      contents: read
      actions: write

    steps:
      ## Android開発の環境をセットアップは省略
      ## ...

      ## mainブランチから変更前のスクリーンショットを取得
      - uses: actions/download-artifact@v2
        continue-on-error: true
        with:
          name: screenshots
          workflow: StoreScreenshot
          branch: main

      ## 変更前のスクリーンショットと比較し、差分を生成
      - name: compare screenshot test
        id: compare-screenshot-test
        run: |
          ./gradlew compareRoborazziDebug --stacktrace

      ## 差分を成果物として保存
      - uses: actions/upload-artifact@v4
        if: ${{ always() }}
        with:
          name: screenshots-diff
          path: |
            **/build/outputs/roborazzi/
          retention-days: 30

      ## PR番号を保存
      - name: Save PR number
        run: |
          mkdir -p ./pr
            echo "${{ github.event.number }}" > ./pr/NR

      ## PR番号を成果物として保存
      - name: upload PR number
        uses: actions/upload-artifact@v4
        with:
          name: pr
          path: pr/
```


### 成果物を使用して、PRにコメントとして表示する
最後に、生成された差分をPRに表示するためのコメントを作成します。

```yml:compare-screenshot-comment.yml
name: Screenshot compare comment

on:
  ## CompareScreenshotワークフローが完了したとき
  workflow_run:
    workflows:
      - CompareScreenshot
    types: [ completed ]

## 並列処理を行う
concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref }}-${{ github.event.workflow_run.id }}
  cancel-in-progress: true

jobs:
  comment-compare-screenshot:
    if: >
      github.event.workflow_run.event == 'pull_request' &&
      github.event.workflow_run.conclusion == 'success'

    timeout-minutes: 2

    permissions:
      actions: read
      contents: write
      pull-requests: write

    runs-on: ubuntu-latest

    steps:
      ## PR番号をダウンロード
      - uses: dawidd6/action-download-artifact@v2
        with:
          name: pr
          run_id: ${{ github.event.workflow_run.id }}

      # PR番号を取得
      - id: get-pull-request-number
        name: Get pull request number
        shell: bash
        run: |
          echo "pull_request_number=$(cat NR)" >> "$GITHUB_OUTPUT"

      ## mainブランチに戻る
      - name: main checkout
        uses: actions/checkout@v4
        with:
          ref: main

      ## コミット履歴を持たないブランチを作成
      - name: switch companion branch
        env:
          BRANCH_NAME: companion_${{ github.event.workflow_run.head_branch }}
        run: |
          git branch -D $BRANCH_NAME || true
          git checkout --orphan $BRANCH_NAME
          git rm -rf .

      ## CompareScreenshotワークフローで生成された差分を取得
      - uses: dawidd6/action-download-artifact@v3
        with:
          name: screenshots-diff
          path: screenshots-diff
          run_id: ${{ github.event.workflow_run.id }}

      ## 差分の画像が命名規則に従っているか確認
      - id: check-if-there-are-valid-files
        name: Check if there are valid files
        shell: bash
        run: |
          mapfile -t files_to_add < <(find . -type f -name "*_compare.png")
          
          exist_valid_files="false"
          for file in "${files_to_add[@]}"; do
            if [[ $file =~ ^[a-zA-Z0-9_./-]+$ ]]; then
              exist_valid_files="true"
              break
            fi
          done
          echo "exist_valid_files=$exist_valid_files" >> "$GITHUB_OUTPUT"

      ## 差分の画像をcompanionブランチにpush
      - id: push-screenshot-diff
        shell: bash
        if: steps.check-if-there-are-valid-files.outputs.exist_valid_files == 'true'
        env:
          BRANCH_NAME: companion_${{ github.event.workflow_run.head_branch }}
        run: |
          files_to_add=$(find . -type f -name "*_compare.png")
          
          
          for file in $files_to_add; do
            if [[ $file =~ ^[a-zA-Z0-9_./-]+$ ]]; then
              git add $file
            fi
          done
          git config --global user.name ScreenshotBot
          git config --global user.email 41898282+github-actions[bot]@users.noreply.github.com
          git commit -m "Add screenshot diff"
          git push origin HEAD:$BRANCH_NAME -f

      ## 差分の画像をPRに表示するためのコメントを生成
      - id: generate-diff-reports
        name: Generate diff reports
        if: steps.check-if-there-are-valid-files.outputs.exist_valid_files == 'true'
        env:
          BRANCH_NAME: companion_${{ github.event.workflow_run.head_branch }}
        shell: bash
        run: |
          files=$(find . -type f -name "*_compare.png" | grep "roborazzi/" | grep -E "^[a-zA-Z0-9_./-]+$")
          delimiter="$(openssl rand -hex 8)"
          {
            echo "reports<<${delimiter}"
          
            echo "Snapshot diff report"
            echo "| File name | Image |"
            echo "|-------|-------|"
          } >> "$GITHUB_OUTPUT"
          
          for file in $files; do
            fileName=$(basename "$file" | sed -r 's/(.{20})/\1<br>/g')
            echo "| [$fileName](https://github.com/${{ github.repository }}/blob/$BRANCH_NAME/$file) | ![](https://github.com/${{ github.repository }}/blob/$BRANCH_NAME/$file?raw=true) |" >> "$GITHUB_OUTPUT"
          done
          echo "${delimiter}" >> "$GITHUB_OUTPUT"

      ## コメントを取得
      - name: Find Comment
        uses: peter-evans/find-comment@v2
        id: fc
        if: steps.generate-diff-reports.outputs.reports != ''
        with:
          issue-number: ${{ steps.get-pull-request-number.outputs.pull_request_number }}
          comment-author: "github-actions[bot]"
          body-includes: "Snapshot diff report"

      ## コメントが存在しない場合、新規作成
      - name: Add or update comment on PR
        uses: peter-evans/create-or-update-comment@v3
        if: steps.generate-diff-reports.outputs.reports != ''
        with:
          comment-id: ${{ steps.fc.outputs.comment-id }}
          issue-number: ${{ steps.get-pull-request-number.outputs.pull_request_number }}
          body: ${{ steps.generate-diff-reports.outputs.reports }}
          edit-mode: replace

      ## 期限切れになったcompanionブランチを削除
      - name: Clean up outdated companion branches
        run: |
          git branch -r --format="%(refname:lstrip=3)" | grep "companion_" | while read -r branch; do
            last_commit_date_timestamp=$(git log -1 --format="%ct" "origin/$branch")
            now_timestamp=$(date +%s)
          
            echo "branch: $branch now_timestamp: $now_timestamp last_commit_date_timestamp: $last_commit_date_timestamp"
            if [ $((now_timestamp - last_commit_date_timestamp)) -gt 2592000 ]; then
              echo "Deleting branch $branch"
              git push origin --delete $branch
            fi
          done
```

セットアップが完了したので、画面に少しだけ変更を加えてみましょう。

```kotlin:HomeScreen.kt
/*...*/
Spacer(modifier = Modifier.height(6.dp))
LazyColumn { /*...*/ }
/*...*/
```

これように実装をすることで、以下の写真のようにPRに差分を表示することができます。

![スクリーンショット 2024-02-23 21.42.41.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2989029/b6dd84e7-d71d-0c87-46af-a1130a68b55a.png)





## まとめ
今回は、RoborazziとGitHub Actionsを使用して画面の差分をPRに表示する実装をいたしました。

今回は、GitHub Actionsをメインに触り、初めての技術ばかりでした。
特にartifactの理解に苦しみ、「変更前のスクリーンショットどうやるんだ・・・」「変更後の画面がpushされたら変更後のスクリーンショットはどうするんだ・・・」みたいになっていました。
しかし、今回の実装をきっかけに、開発環境を効率化するという点で技術領域を広げることはできたのは良い経験になりました。

また、今回は写真の保存方法として、GitHubのstorageとcopnaionブランチを使用しましたが、[Nowinandroid](https://github.com/android/nowinandroid)のように、リポジトリに直接保存する方法もあり、それぞれのプロジェクトに合わせて選択することができそうです。

最後にはなりますが、本記事が誰かの力になれば幸いです😆
