# 事前準備

* CircleCI Server に初ログイン.
* Add project.
* `config.yml` がないことによるエラー発生を確認.

# Hello world (config.0.yml)

CircleCI を使い慣れていても, ファイル名の間違いや構文エラーによって初手で躓く. 初心忘るべからずで, 常に Hello world から始めてもよいだろう.

* `git checkout -b hello-circleci`
* `mkdir -p .circleci`
* `touch .circleci/config.yml`
* `config.yml` で Hello world を実装
* `circleci config validate --host=https://circleci.com` コマンドで `config.yml` の構文チェック
* `git add .circleci/config.yml`
* `git commit -a -m 'Hello CircleCI'` && `git push -u origin hello-circleci`

# 意味のあることをする (config.1.yml)

無事 Hello world を終えられたら, 意味のあることをする. ここでは, 依存関係にあるモジュールのインストールとシングル ファイル スクリプト (webpack) のビルドを実行する.

`config.yml` には以下のギミックが仕込まれている:

* VCS からのチェックアウト. `git clone` を独自実装する必要はない. `checkout` しないと何も始まらないことに注意.
* バージョン番号の計算. コンテナのタグに CircleCI のビルド ID と Git のコミット ダイジェストを挿入する. CircleCI に関係なく便利なテクニック.
* 依存関係キャッシュ. 後々のジョブでこのキャッシュを活用し, 依存関係にあるモジュールのインストールを高速化できる.
* Artifacts へのスクリプトの保存.

# Rebuild with SSH (config.2.yml)

問題が起きたら "Rebuild with SSH" で容易に問題の箇所を特定できる.

今回は Yarn コマンドが Docker イメージ `alpine` にインストールされていないことが原因.
`apk add yarn` するようジョブを構成しなおしてもよいが, 今後毎回これを実行するのは時間がかかって無駄だし苦痛なので, すでに Yarn がインストールされている `node:current-alpine` を使用する.

# Workflow と Parallelism (config.3.yml)

次の段階ではテストを実装したい.
ただ単純に `build` ジョブでテストを実行させるだけでもよいが, そうすると拡張性が損なわれてワークフローの最適度が下がり, オペレーションが非効率化していくので, もう少し慎重にアプローチを検討する.

今回のプロジェクトで実装する単体テストを分析すると, 次のような特徴がある:

* 依存関係にあるモジュールがあらかじめインストールされている必要がある. 特に, ビルドに使用するディレクトリと同じディレクトリでテストする必要がある. (でないと, テストの対象とビルドの対象が異なってしまう可能性がある.)
* ビルドと単体テストは並行してテストを実行できる.
* 実行したい単体テストは複数あるが, それらは同時並行的に実行できる.

そこで, 次のようなアプローチをとる.

1. 依存関係の解決を独立したジョブとして定義し, **ワークスペース (workspace)** の機能を使用してビルドと単体テストを後続のジョブとしてつなげる. (ジョブの直列化)
2. ビルドと単体テストを並行して実行する. (ジョブの並列化)
3. 単体テストのジョブ内では複数のテストを並列に実行する. (テスト分割と parallelism)

1, 2 のようなジョブ単位の構成は**ワークフロー (workflow)** で実装する. 3 のようなジョブ内の構成はジョブ定義で実装する.

# デプロイ (config.4.yml)

ビルドしたものは, 少なくともステージング環境に自動デプロイするのが望ましい. (デプロイした先を使ってスモーク テストや E2E テスト, ドッグ フーディングができるので.) もちろん, 本番環境へのデプロイも可能.

まず, デプロイに必要な手続きを記述したジョブを定義する. その次にワークフローを構成する.

ワークフローの構成では, 本番環境への自動デプロイも想定して以下の 2 点を組み込んである:

1. `main` ブランチの時のみデプロイを実行する. (`filter`)
2. デプロイ前に人手による最終決定のチャンスを設定しておく. (承認ジョブ)
3. デプロイに必要な認証情報を安全な場所に保管する. (Context)
