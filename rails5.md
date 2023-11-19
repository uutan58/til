# gem rails-i18nについて

- Railsでは、様々な言語に対応するために【i18n（アイ・エイティーン・エヌ）】というGemを使用する。
- rails-i18nは、Ruby on Railsアプリケーションで国際化（i18n）と地域化（l10n）を簡単に実装するためのGem。
- 多言語対応を行う際に役立つプリセットの翻訳ファイルやロケールデータを提供しており、Railsのデフォルトのi18n機能と組み合わせて使用することで、アプリケーションの国際化・地域化が容易になる。

# gem rails-i18nのインストール方法

```
#Gemlile

gem 'rails-i18n', '~> バージョンを記載'
```

- 上記のGemを導入することによって、Railsを日本語で使う場合のデフォルトのロケールファイルを「[svenfuchs/rails-i18n](https://github.com/svenfuchs/rails-i18n/blob/master/rails/locale/ja.yml)」をダウンロードしなくても使えるようになる。

```
$ bundle install
```

- 上記を打ち込み実行することでインストールが完了する。
- Gemを追加したり、設定ファイルを追加・変更した時はサーバーの再起動を行わないと反映されないことがあるため要注意！

# デフォルトの言語を日本語に設定する

```
#config/application.rb
#記入例
equire_relative 'boot'
require 'rails/all'

Bundler.require(*Rails.groups)

module BoardApp
  class Application < Rails::Application
    config.time_zone = 'Tokyo'
    config.active_record.default_timezone = :local

    #以下の記述を追記する(設定必須)
    #デフォルトのlocaleを日本語(:ja)にする
    config.i18n.default_locale = :ja

  end
end
```

- ここで『config.i18n.default_locale = :ja』を設定しないとi18nが反映されないので要注意！

# ****i18nの複数ロケールファイルが読み込まれるようpathを通す****

```
#config/application.rb
#記入例
equire_relative 'boot'
require 'rails/all'

Bundler.require(*Rails.groups)

module BoardApp
  class Application < Rails::Application
    config.time_zone = 'Tokyo'
    config.active_record.default_timezone = :local

    #先程追加したコード
    config.i18n.default_locale = :ja
		#以下の記述を追記する(設定必須)
    config.i18n.load_path += Dir[Rails.root.join('config', 'locales', '**', '*.{rb,yml}').to_s]

  end
end
```

- 上記を記述することによって、複数のローケルファイルが読み込まれるようになる。
- ただし、上記コードがなくても『config.i18n.default_locale = :ja』だけでも良いみたい？なぜなら、Gemが自身にその機能が備わっているから、らしい。

> [rails-i18nの公式GitHubからの引用](https://github.com/svenfuchs/rails-i18n)
> 
> 
> ```
> Configuration
> 	Enabled modules
> 
> By default, all rails-i18n modules (locales, pluralization, transliteration, ordinals) are enabled.
> 
> If you would like to only enable specific modules, you can do so in your Rails configuration:
> ```
> 

# ja.ymlファイル内の記載例

viewsの場合

```
# ビューはビューを格納しているフォルダ名を起点にし、ビュー名毎に記述する。
# インデント(2space)でpathを制御している

ja:
  users:
    index:
      title: ユーザ一覧
	helpers:
    submit:
      create: 登録
      submit: 保存
      update: 更新
    show:
      # 引数の指定もできる。
      title: %{user_name}さんのユーザ情報
    edit:
      # view側で t(.titile), user_name: @user.name みたいな感じで設定できる
      title: %{user_name}さんのユーザ情報を編集
```

- フォルダ名とファイル名の位置をインデントで合わせること。

modelの場合

```
# モデルは全て activerecord 以下に記述する。
# これにより、User.model_name.human / User.human_attribute_name({attr_name})で使用可能。

ja:
  activerecord:
    models:
      # view側： User.model_name.human => "ユーザ" / t("activerecord.models.user")と同じ
      user: ユーザー 
      board: 掲示板
    # model毎に定義したいattributesを記述
    attributes:
        user:
          id: ID
          # view側： User.human_attribute_name :name => "名前" /　t("activerecord.attributes.user.name")と同じ
          first_name: 名前
          last_name: 姓
          email: メールアドレス
          file: プロフィール画像
          crypted_password: パスワード
  # 全てのmodelで共通して使用するattributesを定義
  attributes:
    created_at: 作成日
    updated_at: 更新日
```

controllersの場合

```
ja:
  messages:
    update:
      notice: "メッセージの更新に成功しました。"
      alert: "メッセージの更新ができません。"
```

# 設定した日本語を表示する記載方法

viewsの場合

```
#上記の記載例を元に
#「ユーザー一覧」を日本語表示にする
<%= t ('users.index.title') %>

#「登録」を日本語表示にする
<%= t ('helpers.submit.create') %>
```

- <%= t (’フォルダ名.ファイル名.任意のワード’) %>のように記載する。
- 「t」と「(」の間は半角スペースがいる。
