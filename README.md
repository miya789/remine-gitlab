# 要件

- [ ] Redmine 上で、Git リポジトリの差分を確認できる
  - `redmine:4.1.1` ならどのプラグインも動いた
- [x] GitLab 上で、コミットメッセージに記載の ID が Redmine へリンクされる

# 方法

## 0. 初期構築

以下は、構築した Redmine でチケット作成を可能にするための、初期構築手順

### プラグインのインストール

```
cd redmine/plugins/
git clone https://github.com/alphanodes/redmine_messenger.git
```

### 選択肢の値 > チケットの優先度

- 「新しい値」から、適当に「test-priority」というチケットの優先度を作成

### チケットのステータス

- 「新しいステータス」から、適当に「test-status」というステータスを作成

### トラッカー

- 「新しいトラッカー」から、適当に「test-tracker」というトラッカーを作成

### プロジ適当に

- 「新しいプロジェクト」から、適当に「test-pj」というプロジェクトを作成

## 1. GitLab リポジトリの閲覧

基本的に、Redmine のローカルリポジトリが必要になる。
従って、論外だがマウントを共有する方法[^redmine-gitlab-mount]もある。

[^redmine-gitlab-mount]: [Redmine と Gitlab を連携する方法 | 3 月 | 2018 年 | Corporate Blog | 三栄ハイテックス株式会社・LSI 設計や人工知能、組み込みソフト、ビジネス IT 支援](https://www.sanei-hy.co.jp/blog/2018/03/00133/)

Redmine と GitLab を連携するプラグイン[^redmine-gitlab-plugin]があるらしい。
従来は、fetch して、Redmine のローカルリポジトリが必要だったが、これを解消可能になるらしい。

[^redmine-gitlab-plugin]: [Redmine と GitLab の連携プラグインを開発しました！ | フューチャー技術ブログ](https://future-architect.github.io/articles/20210908a/)

いきなりこれは難しそうだが、GitHub[^redmine_github_hook], [^redmine-github-webhooks] なら他にも需要がありそう。
これはきちんと Redmine のプラグインページ[^github-hook]にあった。

[^redmine_github_hook]: https://github.com/koppen/redmine_github_hook
[^redmine-github-webhooks]: [Linking GitHub and Redmine - Webhooks - - DX Business -Macnica,Inc.](https://www.macnica.co.jp/en/business/dx/manufacturers/github/blog_20190109.html)
[^github-hook]: [Github Hook - Plugins - Redmine](https://www.redmine.org/plugins/redmine_github_hook)

## 2. Redmine チケットへの連携

Issues を無効化して、統合の設定をしておけば、リンクが勝手に Redmine になる。
そんなに有用でもない。

## 3. Close 検知による連携

Close 検知で連携する方法[^redmine_github_hook-note]もあるらしい。

[^redmine_github_hook-note]: [GitHub と Redmine を連携させてチケット管理を楽にする方法 | OC テックノート](https://oc-technote.com/github/github-redmine/)
