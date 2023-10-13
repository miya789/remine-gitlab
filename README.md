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

### プロジェクト

- 「新しいプロジェクト」から、適当に「test-pj」というプロジェクトを作成

### 管理 > 設定

- `Gitlab` にチェック

### リポジトリ

| バージョン管理システム | 識別子        | リポジトリのパス                                   |
| ---------------------- | ------------- | -------------------------------------------------- |
| Git                    | `local-repo`  | `/usr/src/redmine/repo/demo.git`                   |
| Gitlab                 | `remote-repo` | `https://gitlab/gitlab-instance-41707d42/demo.git` |

### Web Hooks

アクセストークンは Developer でないと使えなかった。
これを git config のリモートリポジトリとして設定しておけば使える。
![アクセストークンの例](doc/image/gitlab_accesstokens.png)

## 1. GitLab リポジトリの閲覧

基本的に、Redmine のローカルリポジトリが必要になる。
従って、論外だがマウントを共有する方法[^redmine-gitlab-mount]もある。

[^redmine-gitlab-mount]: [Redmine と Gitlab を連携する方法 | 3 月 | 2018 年 | Corporate Blog | 三栄ハイテックス株式会社・LSI 設計や人工知能、組み込みソフト、ビジネス IT 支援](https://www.sanei-hy.co.jp/blog/2018/03/00133/)

Redmine と GitLab を連携するプラグイン[^redmine-gitlab-plugin]があるらしい。
従来は、fetch して、Redmine のローカルリポジトリが必要だったが、これを解消可能になるらしい。
但し、これは自己署名証明書に対応していないので、これが使っているライブラリの作法[^NARKOZ-gitlab-ignore]に則って無効化する必要がある。
しかも、4.19 に対応していないので、自力で変更が必要。
こうすれば、Web Hooks 無しで、勝手に同期してくれる。(ローカルなものがあるかは不明)

```diff:ruby
## Set Gitlab endpoint and token
Gitlab.endpoint = root_url + '/api/v4'
Gitlab.private_token = password
+ Gitlab.httparty = {verify: false}
```

尚、`read_api`のスコープだけ付与でも、動いた。

[^redmine-gitlab-plugin]: [Redmine と GitLab の連携プラグインを開発しました！ | フューチャー技術ブログ](https://future-architect.github.io/articles/20210908a/)
[^NARKOZ-gitlab-ignore]: https://github.com/NARKOZ/gitlab/commit/40295b8889c0094babffc81a5d7749d32b0fbda6

連携に成功すると、以下の様にリポジトリが閲覧可能となる。
![リビジョンのページ](doc/image/redmine-gitlab-repo-1.png)
![リビジョン間の差分](doc/image/redmine-gitlab-repo-2.png)
更に、個別のファイルを開けば、ダウンロードボタンがある。
![リビジョン間の差分](doc/image/redmine-gitlab-repo-3.png)

いきなりこれは難しそうだが、GitHub[^redmine_github_hook], [^redmine-github-webhooks] なら他にも需要がありそう。
これはきちんと Redmine のプラグインページ[^github-hook]にあった。

[^redmine_github_hook]: https://github.com/koppen/redmine_github_hook
[^redmine-github-webhooks]: [GitHub と Redmine の連携 －Webhooks 編－ - DX 事業 - マクニカ](https://www.macnica.co.jp/business/dx/manufacturers/github/blog_20190109.html)
[^github-hook]: [Github Hook - Plugins - Redmine](https://www.redmine.org/plugins/redmine_github_hook)

GitLab 側の Web Hooks に於いて、Push Evnets でも発火するようにすれば通知してくれる。
![GitLabに於けるWeb Hooksの設定例](doc/image/gitlab_webhooks.png)

尚、`参照用キーワード`で指定したキーワードをコミットメッセージに入れないと、チケットのページにコミットが紐づかない。

## 2. Redmine チケットへの連携

Issues を無効化して、統合の設定をしておけば、リンクが勝手に Redmine になる。
そんなに有用でもない。

## 3. Close 検知による連携

Close 検知で連携する方法[^redmine_github_hook-note]もあるらしい。

[^redmine_github_hook-note]: [GitHub と Redmine を連携させてチケット管理を楽にする方法 | OC テックノート](https://oc-technote.com/github/github-redmine/)

## デバッグ

- リポジトリの権限が、`redmine`で触れない場合

```
["  GithubHook: Executing command: 'git fetch origin 'refs/heads/master:refs/heads/master''","  GithubHook: Command 'git fetch origin 'refs/heads/master:refs/heads/master'' didn't exit properly. Full output: [\"error: cannot open FETCH_HEAD: Permission denied\\n\"]","  GithubHook: Redmine repository updated: git (Git: 2.9ms, Redmine: 1.7ms)"]
```

- リポジトリにアクセスできる権限が無い場合

```
["  GithubHook: Executing command: 'git fetch origin 'refs/heads/master:refs/heads/master''","  GithubHook: Command 'git fetch origin 'refs/heads/master:refs/heads/master'' didn't exit properly. Full output: [\"fatal: could not read Username for 'https://gitlab': No such device or address\\n\"]","  GithubHook: Redmine repository updated: git (Git: 132.1ms, Redmine: 2.1ms)"]
```

# アクセスログ

## GitLab プラグイン

Redmine にてリポジトリタブをクリックすると、以下へのアクセスが確認された。

```
172.18.0.2 - - [21/Sep/2023:10:39:15 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/branches?page=1&per_page=50 HTTP/1.1" 200 401 "" "Gitlab Ruby Gem 4.19.0" 2.06
172.18.0.2 - - [21/Sep/2023:10:39:15 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/branches?page=2&per_page=50 HTTP/1.1" 200 2 "" "Gitlab Ruby Gem 4.19.0" -
172.18.0.2 - - [21/Sep/2023:10:39:16 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?all=true&since=2023-09-21T10%3A05%3A31Z&page=1&per_page=50 HTTP/1.1" 200 323 "" "Gitlab Ruby Gem 4.19.0" 1.86
172.18.0.2 - - [21/Sep/2023:10:39:16 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?all=true&since=2023-09-21T10%3A05%3A31Z&page=1&per_page=50 HTTP/1.1" 200 323 "" "Gitlab Ruby Gem 4.19.0" 1.86
172.18.0.2 - - [21/Sep/2023:10:39:16 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits/1cda7acbb2f9f6045d671f03c26e52ff6a7b29a1/diff HTTP/1.1" 200 268 "" "Gitlab Ruby Gem 4.19.0" 1.46
172.18.0.2 - - [21/Sep/2023:10:39:16 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?all=true&since=2023-09-21T10%3A05%3A31Z&page=2&per_page=50 HTTP/1.1" 200 2 "" "Gitlab Ruby Gem 4.19.0" -
172.18.0.2 - - [21/Sep/2023:10:39:16 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/tree?path=&ref=master&page=1&per_page=50 HTTP/1.1" 200 258 "" "Gitlab Ruby Gem 4.19.0" 1.97
172.18.0.2 - - [21/Sep/2023:10:39:16 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/files/.gitignore?ref=master HTTP/1.1" 200 345 "" "Gitlab Ruby Gem 4.19.0" 1.39
172.18.0.2 - - [21/Sep/2023:10:39:16 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?path=.gitignore&ref_name=master&per_page=1 HTTP/1.1" 200 332 "" "Gitlab Ruby Gem 4.19.0" 1.87
172.18.0.2 - - [21/Sep/2023:10:39:16 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/files/.gitlab-ci.yml?ref=master HTTP/1.1" 200 947 "" "Gitlab Ruby Gem 4.19.0" 1.62
172.18.0.2 - - [21/Sep/2023:10:39:17 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?path=.gitlab-ci.yml&ref_name=master&per_page=1 HTTP/1.1" 200 332 "" "Gitlab Ruby Gem 4.19.0" 1.86
172.18.0.2 - - [21/Sep/2023:10:39:17 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/files/README.md?ref=master HTTP/1.1" 200 2801 "" "Gitlab Ruby Gem 4.19.0" 1.86
172.18.0.2 - - [21/Sep/2023:10:39:17 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?path=README.md&ref_name=master&per_page=1 HTTP/1.1" 200 323 "" "Gitlab Ruby Gem 4.19.0" 1.86
172.18.0.2 - - [21/Sep/2023:10:39:17 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/files/SUMMARY.md?ref=master HTTP/1.1" 200 378 "" "Gitlab Ruby Gem 4.19.0" 1.36
172.18.0.2 - - [21/Sep/2023:10:39:17 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?path=SUMMARY.md&ref_name=master&per_page=1 HTTP/1.1" 200 326 "" "Gitlab Ruby Gem 4.19.0" 1.88
172.18.0.2 - - [21/Sep/2023:10:39:17 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/tree?path=&ref=master&page=2&per_page=50 HTTP/1.1" 200 2 "" "Gitlab Ruby Gem 4.19.0" -
172.18.0.2 - - [21/Sep/2023:10:39:17 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?path=&ref_name=master&per_page=10 HTTP/1.1" 200 942 "" "Gitlab Ruby Gem 4.19.0" 6.17
172.18.0.2 - - [21/Sep/2023:10:39:17 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/tags?page=1&per_page=50 HTTP/1.1" 200 2 "" "Gitlab Ruby Gem 4.19.0" -
172.18.0.2 - - [21/Sep/2023:10:39:17 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/branches?page=1&per_page=50 HTTP/1.1" 200 401 "" "Gitlab Ruby Gem 4.19.0" 2.06
172.18.0.2 - - [21/Sep/2023:10:39:17 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/branches?page=2&per_page=50 HTTP/1.1" 200 2 "" "Gitlab Ruby Gem 4.19.0" -
172.18.0.2 - - [21/Sep/2023:10:39:17 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?all=true&since=2023-09-21T10%3A05%3A31Z&page=1&per_page=50 HTTP/1.1" 200 323 "" "Gitlab Ruby Gem 4.19.0" 1.86
172.18.0.2 - - [21/Sep/2023:10:39:17 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?all=true&since=2023-09-21T10%3A05%3A31Z&page=1&per_page=50 HTTP/1.1" 200 323 "" "Gitlab Ruby Gem 4.19.0" 1.86
172.18.0.2 - - [21/Sep/2023:10:39:17 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits/1cda7acbb2f9f6045d671f03c26e52ff6a7b29a1/diff HTTP/1.1" 200 268 "" "Gitlab Ruby Gem 4.19.0" 1.46
172.18.0.2 - - [21/Sep/2023:10:39:17 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?all=true&since=2023-09-21T10%3A05%3A31Z&page=2&per_page=50 HTTP/1.1" 200 2 "" "Gitlab Ruby Gem 4.19.0" -
172.18.0.2 - - [21/Sep/2023:10:39:17 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/tree?path=&ref=master&page=1&per_page=50 HTTP/1.1" 200 258 "" "Gitlab Ruby Gem 4.19.0" 1.97
172.18.0.2 - - [21/Sep/2023:10:39:17 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/files/.gitignore?ref=master HTTP/1.1" 200 345 "" "Gitlab Ruby Gem 4.19.0" 1.39
172.18.0.2 - - [21/Sep/2023:10:39:18 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?path=.gitignore&ref_name=master&per_page=1 HTTP/1.1" 200 332 "" "Gitlab Ruby Gem 4.19.0" 1.87
172.18.0.2 - - [21/Sep/2023:10:39:18 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/files/.gitlab-ci.yml?ref=master HTTP/1.1" 200 947 "" "Gitlab Ruby Gem 4.19.0" 1.62
172.18.0.2 - - [21/Sep/2023:10:39:18 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?path=.gitlab-ci.yml&ref_name=master&per_page=1 HTTP/1.1" 200 332 "" "Gitlab Ruby Gem 4.19.0" 1.86
172.18.0.2 - - [21/Sep/2023:10:39:18 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/files/README.md?ref=master HTTP/1.1" 200 2801 "" "Gitlab Ruby Gem 4.19.0" 1.86
172.18.0.2 - - [21/Sep/2023:10:39:18 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?path=README.md&ref_name=master&per_page=1 HTTP/1.1" 200 323 "" "Gitlab Ruby Gem 4.19.0" 1.86
172.18.0.2 - - [21/Sep/2023:10:39:18 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/files/SUMMARY.md?ref=master HTTP/1.1" 200 378 "" "Gitlab Ruby Gem 4.19.0" 1.36
172.18.0.2 - - [21/Sep/2023:10:39:18 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?path=SUMMARY.md&ref_name=master&per_page=1 HTTP/1.1" 200 326 "" "Gitlab Ruby Gem 4.19.0" 1.88
172.18.0.2 - - [21/Sep/2023:10:39:18 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/tree?path=&ref=master&page=2&per_page=50 HTTP/1.1" 200 2 "" "Gitlab Ruby Gem 4.19.0" -
172.18.0.2 - - [21/Sep/2023:10:39:18 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?path=&ref_name=master&per_page=10 HTTP/1.1" 200 942 "" "Gitlab Ruby Gem 4.19.0" 6.17
172.18.0.2 - - [21/Sep/2023:10:39:18 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/tags?page=1&per_page=50 HTTP/1.1" 200 2 "" "Gitlab Ruby Gem 4.19.0" -
127.0.0.1 - - [21/Sep/2023:10:39:20 +0000] "GET /help HTTP/1.1" 200 71357 "" "curl/7.87.0-DEV" -
```

## GitHub プラグイン

GitLab にて master ブランチで更新を掛けると、以下へのアクセスが確認された。(Web IDE によるものも含む)

```
172.18.0.1 - - [21/Sep/2023:10:41:31 +0000] "GET /gitlab-instance-41707d42/demo/commit/1cda7acbb2f9f6045d671f03c26e52ff6a7b29a1/pipelines HTTP/2.0" 301 173 "https://localhost/-/ide/project/gitlab-instance-41707d42/demo/tree/master/-/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36 Edg/117.0.2045.31" -
172.18.0.1 - - [21/Sep/2023:10:41:31 +0000] "GET /gitlab-instance-41707d42/demo/-/commit/1cda7acbb2f9f6045d671f03c26e52ff6a7b29a1/pipelines HTTP/2.0" 200 1086 "https://localhost/-/ide/project/gitlab-instance-41707d42/demo/tree/master/-/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36 Edg/117.0.2045.31" 3.17
172.18.0.1 - - [21/Sep/2023:10:41:31 +0000] "POST /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits HTTP/2.0" 201 668 "https://localhost/-/ide/project/gitlab-instance-41707d42/demo/tree/master/-/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36 Edg/117.0.2045.31" -
172.18.0.1 - - [21/Sep/2023:10:41:31 +0000] "POST /gitlab-instance-41707d42/demo/ide_terminals/check_config HTTP/2.0" 422 0 "https://localhost/-/ide/project/gitlab-instance-41707d42/demo/tree/master/-/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36 Edg/117.0.2045.31" -
172.18.0.1 - - [21/Sep/2023:10:41:31 +0000] "GET /assets/webpack/yaml.fe43b05f.worker.js HTTP/2.0" 200 351681 "https://localhost/-/ide/project/gitlab-instance-41707d42/demo/tree/master/-/README.md/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36 Edg/117.0.2045.31" -
172.18.0.1 - - [21/Sep/2023:10:41:31 +0000] "GET /assets/webpack/editor.84a8ae44.worker.js HTTP/2.0" 200 44179 "https://localhost/-/ide/project/gitlab-instance-41707d42/demo/tree/master/-/README.md/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36 Edg/117.0.2045.31" -
172.18.0.1 - - [21/Sep/2023:10:41:31 +0000] "GET /api/v4/projects/34/runners?scope=active HTTP/2.0" 200 2 "https://localhost/-/ide/project/gitlab-instance-41707d42/demo/tree/master/-/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36 Edg/117.0.2045.31" -
172.18.0.1 - - [21/Sep/2023:10:41:31 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/branches/master HTTP/2.0" 200 381 "https://localhost/-/ide/project/gitlab-instance-41707d42/demo/tree/master/-/README.md/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/117.0.0.0 Safari/537.36 Edg/117.0.2045.31" 2.07
172.18.0.2 - - [21/Sep/2023:10:41:32 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/branches?page=1&per_page=50 HTTP/1.1" 200 393 "" "Gitlab Ruby Gem 4.19.0" 2.08
172.18.0.2 - - [21/Sep/2023:10:41:32 +0000] "GET /gitlab-instance-41707d42/demo.git/info/refs?service=git-upload-pack HTTP/2.0" 401 265 "" "git/2.20.1" -
172.18.0.2 - redmine [21/Sep/2023:10:41:32 +0000] "GET /gitlab-instance-41707d42/demo.git/info/refs?service=git-upload-pack HTTP/2.0" 200 295 "" "git/2.20.1" -
172.18.0.2 - - [21/Sep/2023:10:41:32 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/branches?page=2&per_page=50 HTTP/1.1" 200 2 "" "Gitlab Ruby Gem 4.19.0" -
172.18.0.2 - redmine [21/Sep/2023:10:41:32 +0000] "POST /gitlab-instance-41707d42/demo.git/git-upload-pack HTTP/2.0" 200 1315 "" "git/2.20.1" -
172.18.0.2 - - [21/Sep/2023:10:41:32 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?all=true&since=2023-09-21T10%3A05%3A31Z&page=1&per_page=50 HTTP/1.1" 200 385 "" "Gitlab Ruby Gem 4.19.0" 3.08
172.18.0.2 - - [21/Sep/2023:10:41:32 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?all=true&since=2023-09-21T10%3A05%3A31Z&page=1&per_page=50 HTTP/1.1" 200 385 "" "Gitlab Ruby Gem 4.19.0" 3.08
172.18.0.2 - - [21/Sep/2023:10:41:32 +0000] "GET /gitlab-instance-41707d42/demo.git/info/refs?service=git-upload-pack HTTP/2.0" 401 265 "" "git/2.20.1" -
172.18.0.2 - - [21/Sep/2023:10:41:32 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits/1cda7acbb2f9f6045d671f03c26e52ff6a7b29a1/diff HTTP/1.1" 200 268 "" "Gitlab Ruby Gem 4.19.0" 1.46
172.18.0.2 - redmine [21/Sep/2023:10:41:32 +0000] "GET /gitlab-instance-41707d42/demo.git/info/refs?service=git-upload-pack HTTP/2.0" 200 295 "" "git/2.20.1" -
172.18.0.2 - - [21/Sep/2023:10:41:32 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits/0c7bf8fd16b615b3e217a83ff44dde35e6a2f63a/diff HTTP/1.1" 200 251 "" "Gitlab Ruby Gem 4.19.0" 1.45
172.18.0.2 - - [21/Sep/2023:10:41:32 +0000] "GET /api/v4/projects/gitlab-instance-41707d42%2Fdemo/repository/commits?all=true&since=2023-09-21T10%3A05%3A31Z&page=2&per_page=50 HTTP/1.1" 200 2 "" "Gitlab Ruby Gem 4.19.0" -
```

# その他

## Docker による構築

- https://qiita.com/hadacchi/items/ca10939ca016147e225a

## 不明

- https://blog.devplatform.techmatrix.jp/blog/try-gitlab-redmine/
- https://qiita.com/mima_ita/items/82c5d5da30de3bc47e28
- https://qiita.com/n_slender/items/54cd282c140fadbbb322
