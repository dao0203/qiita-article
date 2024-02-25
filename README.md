# qiita-article
Qiitaの記事リポジトリ

## よく使うコマンド
### プレビューの表示
記事の執筆は、ブラウザでプレビューしながら確認できる。またlコマンド実行時にQiitaに投稿している記事がダウンロードされる。
```bash
npx qiita preview
```

### 記事の管理
#### 記事の作成
以下のコマンドで新規記事を作成できる。
```bash
npx qiita new <記事のファイルのベース名>
```

#### 記事の投稿・更新
以下のコマンドで記事を投稿・更新できる。
```bash
npx qiita publish <記事のファイルのベース名>
```

以下のコマンドで全ての記事を更新できる
```bash
npx qiita publish --all
```
また、以下のコマンドで記事ファイルをQiitaと同期できる。
```bash
npx qiita pull
```
