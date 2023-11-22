# フラッシュメッセージとは

- 何らかの処理の後に表示させる簡易メッセージのこと。

例）ユーザーの新規登録時「登録が完了しました」やログイン時に「ログインしました」のようなやつ

- Railsでは、Viewで一時的なメッセージを表示するために、『flash』というハッシュ形式のオブジェクトが設定されている。
- 一度表示が完了すると、『flash』内のオブジェクトは自動で削除されるため、一時的に表示させたいお知らせ等がある場合に最適。

# フラッシュメッセージの使い方

- 『flash』はハッシュのような形式で記述する。
- 任意のキー名をつけることができる。

```
#コントローラ
flash[:キー名] = "表示したいメッセージ"
```

```
#ビューファイル
<%= flash[:キー名] %>
```

# flashのkeyについて

- flashオブジェクトは、ハッシュ形式で保存がなされており、デフォルトでは『notice』と『alert』がキーとして設定されている。
- bootstrapを使用している場合、より多くのキーで表示を出し分けることが可能。
- 主に「success（緑色）」「info（青色）」「warning（黄色）」「danger（赤色）を使用することが多い。公式では全8色ある。

# フラッシュメッセージの設定の流れ

1. Bootstrapに用意されているスタイルのフラッシュを定義する。
2. コントローラにフラッシュメッセージを追記する。
3. フラッシュメッセージの部分テンプレートを作成する。
4. フラッシュメッセージの表示をビューファイルに追記する。

# Bootstrapに用意されているスタイルのフラッシュを定義する

- 以下の設定を追加することで、デフォルトの『notice』と『alert』以外でBootstrapに用意されているスタイルのフラッシュを定義できる。

```
#application_controller.rb

#記入例1
add_flash_types :success, :info

#記入例2
add_flash_types :success, :danger
```

- 『add_flash_types』を定義しなくても毎回『flash』を記載すればフラッシュメッセージの表示は可能。

```
#『add_flash_types』を定義せず『flash』を使用する場合
redirect_to login_path, flash: {success: 'hoge' }
```

- ただし、この場合は記述量が増えてしまうため、『add_flash_types』を事前に定義した方が良い。
- 『redirect_back_or_to』では『add_flash_types』を定義していなくても『flash』の記載を省略できる。

```
#『redirect_back_or_to』で『flash』を省略した場合
redirect_back_or_to root_path, success: 'hoge'
```

# フラッシュメッセージの記述方法

```
#処理成功時
success: t('.success')

#処理失敗時
flash.now[:danger] = t('.fail')
```

# flashとflash.nowの違い

- 『flash』は」、次のアクションが動いた後のビューファイルにフラッシュメッセージを表示（リダイレクト）する時に使用する。
    
    リダイレクトされて、次のアクションが実行されてレンダリングされるビューファイルでメッセージを表示する。
    
- 『flash.now』は、現在のアクションで表示するビューファイルのみ有効なフラッシュメッセージを表示（リダイレクト）する時に使用する。
    
    処理失敗時などで、『render』を使っているアクションに書くことで通常の『flash』と同じ要領でメッセージを表示することができる。
    

# コントローラにフラッシュメッセージを追記

- 例）ログイン時の処理にフラッシュメッセージを追記する場合

```
#controllers/user_sessions_controller.rb

class UserSessionsController < ApplicationController
  def create
    @user = login(params[:email], params[:password])
    if @user
			#ログインに成功した場合
      redirect_to root_path, success: t('.success')
    else
			#ログインに失敗した場合
      flash.now[:danger] = t('.fail')
      render :new
    end
  end

  def destroy
    logout
		#ログアウトに成功した場合
    redirect_to root_path, success: t('.success')
  end
end
```

- 例）新規ユーザー登録時の処理にフラッシュメッセージを追記する場合

```
#controllers/users_controller.rb

class UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    if @user.save
			#新規ユーザーの登録に成功した場合
      redirect_to login_path, success: t('.success')
    else
			#新規ユーザーの登録に失敗した場合
      flash.now[:danger] = t('.fail')
      render :new
    end
  end
end
```

# フラッシュメッセージの部分テンプレートを作成する

- フラッシュメッセージは再利用する可能性を持たせたいため、パーシャル化しておく。

> 3.2 パーシャルとは
> 
> 
> 部分テンプレートまたはパーシャルは、出力を扱いやすく分割するための仕組みです。パーシャルを使用することで、ビュー内のコードをいくつものファイルに分割して書き出し、他のテンプレートでも使いまわすことができます。
> 
> 参照　[RAILS GUIDES](https://railsguides.jp/action_view_overview.html#%E3%83%91%E3%83%BC%E3%82%B7%E3%83%A3%E3%83%AB)
> 

```
#app/views/shared/_flash_message.html.erb

<% flash.each do |message_type, message| %>
  <div class="alert alert-<%= message_type %>"><%= message %></div>
<% end %>
```

# フラッシュメッセージの表示をビューファイルに追加する

- レイアウトファイルに追加する。

```
#layouts/applocation.html.erb

<body>
  <% if logged_in? %>
    <%= render 'shared/header' %>
  <% else %>
    <%= render 'shared/before_login_header' %>
  <% end %>
	#ここに記載する
  <%= render 'shared/flash_message' %>
  <%= yield %>
  <%= render 'shared/footer' %>
</body>
```
