# 実装の流れ

- エラーメッセージを表示するパーシャルを作成する。
- ビューファイルでエラーメッセージを表示させる。

# エラーメッセージを表示するパーシャルを作成する

```
#shared/_error_messages.html.erb

<% if object.errors.any? %>
	<div class="alert alert-danger">
		<ul class="mb-0">
			<% object.errors.full_messages.each do |msg| %>
				<li><%= msg %></li>
			<% end %>
		</ul>
	</div>
<% end %>
```

- 再利用性を高めるため、汎用的なパーシャルとして作成する。

# ****ビューファイルでエラーメッセージを表示させる****

```
#views/boards/new.html.erb

<%= form_with model: board, local: true do |f| %>
	#この↓一行を任意のビューファイルに追記するだけ
	<%= render 'shared/error_messages', object: f.object %>
	<div class="form-group">
		<%= f.label :title %>
```

- 掲示板入力フォーム、ユーザー新規登録からパーシャルを呼び出す。

### <%= render 'shared/error_messages', object: f.object %>について

- このコードはRailsのパーシャルをレンダリングするもので、`shared/error_messages`というパーシャルに`f.object`（フォームオブジェクト）を`object`という名前で渡している。
    
    パーシャル内で`object`が持つ`errors`メソッドを使って、そのオブジェクトのエラーメッセージを取得し、表示している。
    
- エラーメッセージが「〇〇を入力してください」と日本語で表示されるのは、Railsが標準でI18n（国際化）ライブラリをサポートしているから。
    
    Railsでは`config/locales`ディレクトリにロケールファイルを置くことで、エラーメッセージを含む多くの文字列を翻訳できるようになっている。
    
    もしロケールファイルが設定されておらず、特に日本語の設定がなければ、デフォルトでは英語のエラーメッセージが表示される。
    
    日本語のエラーメッセージが表示される場合は、どこかで日本語のロケールファイルが設定されているはず。
