---
title: reg-suitとroborazziを使用してVRTを行い、差分をPRに表示する
tags:
  - Android
  - GitHubActions
  - reg-suit
  - VRT
  - Roborazzi
private: false
updated_at: '2024-02-26T01:22:10+09:00'
id: 6709667949d97abdf39f
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
こんにちは、こんばんは、佐藤佑哉です。
本日は、Androidアプリ開発で使用されるRoborazziとReg-suitを用いて、VRTを行い、その結果をPRに表示する方法をGitHubActionsで実現していきたいと思います。

## 本記事の動機
これは、[前回の記事の続き](https://qiita.com/dao0203/items/afebaeb10af0219e29a0)になります。
1番初めのVRTの記事でreg-suitを使用してVRTを行い、差分レポートを表示するところまでやりました。そして、前回はroborazziのVRTの結果をPRに表示するところまでやりました。
今回は、roborazziとreg-suitを組み合わせて、GitHub Actionsを使用して、VRTを行い、その結果をPRに表示する方法を紹介していきたいと思います。
前回のroborazziでは差分の写真やコメントをPRに表示することができましたが、reg-suitのサイトには動的に差分を表示する機能があり、サイトの方が見やすいと感じたので、それをPRに表示するようにしていきたいと思います。

## 使用する技術
今回は以下の技術を使用していきます。

reg-suit
VRTを行い、差分レポートを表示するためのツール

https://reg-viz.github.io/reg-suit/

Roborazzi
スクリーンショットを撮影するためのライブラリ

https://github.com/takahirom/roborazzi

GitHub Actions
CI/CDを行うためのツール

https://github.co.jp/features/actions

GitHub Pages
GitHubの静的ページホスティングサービス

https://docs.github.com/ja/pages/getting-started-with-github-pages/about-github-pages


## 前回の記事のまとめ
前回の記事では以下のようなことを行いました。

https://qiita.com/dao0203/items/afebaeb10af0219e29a0

![roborazziのworkflow構成図.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2989029/940f3f7e-b138-be0a-877d-36287a946021.png)

前回はGitHubのStorageを使用して、変更前のスクリーンショットを保存し、それを比較して差分を表示するという流れでした。
今回は、reg-suitによるVRTがメインなので、roborazziの役割であるスクリーンショットの撮影の部分は変わらずに使用していきます。

## PRに表示するまでの流れ
今回は、以下のような流れで実装していきます。

前回と違い、今回はサイトを表示するために、実装コストもかからず、無料で使用できるGitHub Pagesを使用していきます。

![roborazziのworkflow構成図その2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2989029/d542747f-0ff4-4baa-abe2-474555e30c0c.png)


### reg-suitを使用してVRTを行う
以下のように`compare-screenshot.yml`のjobを変更して、VRTを行うようにします。

まず初めに、github pagesに結果を表示するために、actionsをwriteに変更します。

```yml
permissions:
  contents: write
  actions: write
```

次に、reg-suitが正しく動作するために、現在のチェックアウトされているGitブランチの状態を取得するために以下のコマンドを実行します。
reg-suitのgit-hash-pluginは、ベースコミットのハッシュ値を取得するために、現在のチェックアウトされているGitブランチの状態を取得する必要があります。
しかし、GitHub Actionsで実行されると、デフォルトでdetached HEAD状態になってしまうため、以下のコマンドを実行して、detached HEAD状態を解消します。
これをしないと、`reg-suit run`時に`Error: Fail to detect the current branch.
`が発生してしまい、正しくVRTが行えません。

```yml
 ## #regs/head/を書くことで、ヘッドブランチ名を取得できる
- name: workaround for detached HEAD
  run: |
    git checkout ${GITHUB_HEAD_REF#refs/heads/} || git checkout -b ${GITHUB_HEAD_REF#refs/heads/} && git pull origin ${GITHUB_HEAD_REF#refs/heads/}
```

次に、スクリーンショットを`.reg/expected`に移動します。
reg-suitは、スクリーンショットを確認するために、変更前のスクリーンショットは`.reg/expected`に保存する必要があります。

```yml
## 前回と同じく、store-screenshot.ymlで保存したスクリーンショットをダウンロード
- uses: dawidd6/action-download-artifact@v2
  continue-on-error: true
  with:
    name: screenshots
    workflow: store-screenshot.yml
    branch: ${{ github.event.pull_request.base.ref || github.event.repository.default_branch }}

## 先ほどダウンロードしたスクリーンショットを/.reg/expectedに移動
- name: Move screenshots
  run: |
    mkdir -p .reg/expected
    mv actual_images/* .reg/expected ## actual_imagesはroborazziで保存したスクリーンショットのディレクトリ名
```

次に、変更後のスクリーンショットを記録します。
roborazziのスクリーンショットの撮影の部分は変わらずに使用していくので、そのまま記録します。

```yml
- name: record screenshots
  id: record-screenshot-test
  run: |
    ./gradlew recordRoborazziDebug --stacktrace
```

次に、reg-suitの実行を行います。
以下のコマンドを実行することで、VRTを行い、差分レポートを表示することができます。

```yml
- name: run reg-suit
  id: compare-screenshot-test
  run: |
    yarn ci:reg-suit ## package.jsonに記述されているコマンド reg-suit run を実行
```

次に、結果をgithub pagesにデプロイします。
以下のコマンドを実行することで、github pagesに結果をデプロイすることができます。

```yml
- name: deploy test results to github pages
  uses: peaceiris/actions-gh-pages@v3
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    publish_dir: .reg
    destination_dir: ${{ github.head_ref }}
```

最後に、結果をPRに表示します。

```yml
- name: find comment
  uses: peter-evans/find-comment@v3
  id: fc
  with:
    issue-number: ${{ github.event.pull_request.number }}
    comment-author: github-actions[bot]
    body-includes: '## Screenshot Test Results'

- name: upsert comment
  uses: peter-evans/create-or-update-comment@v4
  with:
    comment-id: ${{ steps.fc.outputs.comment-id }}
    issue-number: ${{ github.event.pull_request.number }}
    body: |
      ## Screenshot Test Results
      https://dao0203.github.io/todo-sample-app/${{ github.head_ref }}
    edit-mode: replace
```

このように実装し、以下のようなymlファイルになると思います。

<details><summary>compare-screenshot.yml</summary><div>

```yml
name: CompareScreenshot

on:
  pull_request:

jobs:
  compare-screenshot-test:
    runs-on: ubuntu-latest
    timeout-minutes: 20

    permissions:
      contents: write
      actions: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: 17
          distribution: adopt

      - name: Set up Gradle
        uses: gradle/gradle-build-action@v3
        with:
          gradle-version: wrapper

      - name: workaround for detached HEAD
        run: |
          git checkout ${GITHUB_HEAD_REF#refs/heads/} || git checkout -b ${GITHUB_HEAD_REF#refs/heads/} && git pull origin ${GITHUB_HEAD_REF#refs/heads/}

      - uses: dawidd6/action-download-artifact@v2
        continue-on-error: true
        with:
          name: screenshots
          workflow: store-screenshot.yml
          branch: ${{ github.event.pull_request.base.ref || github.event.repository.default_branch }}

      ## 先ほどダウンロードしたスクリーンショットを/.reg/expectedに移動
      - name: Move screenshots
        run: |
          mkdir -p .reg/expected
          mv actual_images/* .reg/expected

      ## 変更後のスクリーンショットを記録する
      - name: record screenshots
        id: record-screenshot-test
        run: |
          ./gradlew recordRoborazziDebug --stacktrace

      ## reg-suitを実行してスクリーンショットの差分を確認
      - name: run reg-suit
        id: compare-screenshot-test
        run: |
          yarn ci:reg-suit

      - name: deploy test results to github pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: .reg
          destination_dir: ${{ github.head_ref }}

      - name: find comment
        uses: peter-evans/find-comment@v3
        id: fc
        with:
          issue-number: ${{ github.event.pull_request.number }}
          comment-author: github-actions[bot]
          body-includes: '## Screenshot Test Results'

      - name: upsert comment
        uses: peter-evans/create-or-update-comment@v4
        with:
          comment-id: ${{ steps.fc.outputs.comment-id }}
          issue-number: ${{ github.event.pull_request.number }}
          body: |
            ## Screenshot Test Results
            https://dao0203.github.io/todo-sample-app/${{ github.head_ref }}
          edit-mode: replace
```
</div></details>

これで、VRTを行い、その結果をPRに表示することができるようになりました。

![ScreenshotTestResult.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2989029/3c32215b-e3ee-98bd-8243-6e3aa2b845fc.png)


しかし、このままだと、サイトを開いた時に、`404 Not Found`が表示されてしまいます。
そのため、GitHubPagesでサイトが表示されるようにするために、`Settings -> Pages`に移動し、以下のような設定を行います。

![GitHubPagesConfig.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2989029/9c008534-821d-b0f6-f093-056dc32d4ca8.png)

これによって、GitHubPagesでサイトが表示されるようになりました。

![スクリーンショット 2024-02-26 0.26.09.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2989029/0f788493-0799-358d-cc98-effef1f3541a.png)

## まとめ
今回は、reg-suitとroborazziとGitHubActionsを用いて、VRTを行い、その結果をGitHubPagesに表示し、PRに表示する方法を紹介しました。
GitHubPagesを使用することで、サイトの表示が簡単にできるため、そこまで実装コストと変更コストがかからず、無料で使用できるため、とても便利だと感じました。
また、開発を通じて、reg-suitの高機能なツールを活かしたまま、Roborazziの高速なスクリーンショットの撮影を活かすことができるため、移行作業もスムーズに行えると感じました。

今回の記事が、VRTを行う際の参考になれば幸いです😆
