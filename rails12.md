# タイトルを動的に出力する理由

- タイトルを固定にするのはユーザビリティやSEOの観点からあまりよくない、ページごとに出力を変えたい場合がある。
    
    タイトルとは、ブラウザのタブ部分に表記されている文字のこと。
    
    `content_for` というヘルパーを使って**「ページ名 | 固定タイトル」**と出力させる。
    
- 以下、実装例
    - トップページ・・・SAMPLE BOARD APP
    - ログインページ・・・ログイン | SAMPLE BOARD APP
    - ユーザー登録ページ・・・ユーザー登録 | SAMPLE BOARD APP
    - 掲示板作成ページ・・・掲示板作成 | SAMPLE BOARD APP
    - 掲示板一覧ページ・・・掲示板一覧 | SAMPLE BOARD APP
    - 掲示板詳細ページ・・・個別の掲示板のタイトル名 | SAMPLE BOARD APP
- 基本的に、「｜」の右側は固定タイトルとし、左側を各々のページごとに出力を変える。
    
    掲示板詳細ページのタイトルは個別の掲示板タイトルを表示させるようにし、トップページは「｜」は不要で固定タイトルのみ表示させる。
    

# 実装の流れ

1. 新しくカスタムヘルパーメソッドを作成する
2. ビュー側に設定を追記する
    1. レイアウトのタイトルを変更する
    2. 各ページのタイトルを変更する

# ****新しくカスタムヘルパーメソッドを作成する****

- タイトル部分を組み立てるカスタムヘルパーメソッドを`application_helper` に書く。

```
#app/aplication_helper.rb

module AplicationHelper
	def page_title(page_title = '')
		base_title = 'SAMPLE BOARD APP'
	
		page_title.empty? ? base_title : "#{page_title} | #{base_title}"
	end
end
```

- 検索すると`page_title.empty? ? base_tiitle : page_title + ' | ' + base_title`と出てくるが、RuboCop（Rubyの静的コード解析ツール）は文字列の連結の代わりに文字列の挿入（インターポレーション）を使用するよう警告してくるため、`"#{page_title} | #{base_title}"`を使用すること。
- 文字列の挿入はRubyで一般的に推奨されるスタイルで、パフォーマンスも良い。

# ****ビュー側に設定を追記する****

### ****レイアウトのタイトルを変更する****

- レイアウトのタイトルを動的に変更できるように修正する。

```
#app/views/layouts/application.html.erb

<title><%= page_title(yield(:title)) %></title>
```

### ****各ページのタイトルを変更する****

- ja.yml（ロケールファイル）を事前に設定しておく。
- `content_for` を使って、『ログイン、ユーザー作成、掲示板作成、掲示板一覧』の各ページの先頭にタイトルを設定する。
    
    ※トップページはそのままにすること
    

```
<% content_for(:title, t('.title')) %>
```

- 掲示板詳細ページは、個別の掲示板のタイトル名を表示したいので`@board.title`にする。

```
- #boards/show.html.erb

<% content_for(:title, @board.title) %>
```
