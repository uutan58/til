# 実装の流れ

1. Boardモデルの作成
2. Fakerを使ってダミーデータを作成する
3. 掲示板一覧のルーティングを設定
4. 掲示板一覧のコントローラーを作成
5. 掲示板一覧画面のビューファイルを作成
6. ログインしていなければ掲示板一覧を表示できなくする
7. ログイン時の処理を修正する
8. タイムゾーンを日本時刻に設定する

# Boardモデルの作成

- 掲示板のモデルを生成する。
    
    「title」「body」「userと紐付けるための外部キー」の3つのカラムを生成する。
    

### references型

- 『user has_many borads』の関係を実現するためには『boradsテーブル』に『user_id』が必要になる。

```
$ bundle exec rails g model board titel:string body:text user:references
```

- 外部キーを作る時は、基本的には『references型』を使用する。
    
    なぜなら、`rails g model board title:string body:text user_id:integer`という書き方では、インデックスが貼られない、外部キー制約が貼られない、などの問題が生じるため。
    

### マイグレーションファイルを確認し、NOT NULL制約を追記

- マイグレーションファイルと、『modelファイル』が作成される。
- NOT NULL制約を追加しておく。

```
#2023XXXXXXXXXXX_create_boards.rb

class CreateBoards < ActiveRecord::Migration[5.2]
  def change
    create_table :boards do |t|
      t.references :user, foreign_key: true
      t.string :title, null: false
      t.text :body, null: false

      t.timestamps
    end
  end
end
```

- references型を指定して、『rails g model』を実行するとモデルの他に、マイグレーションファイルにも下記のような記述が入る。

```
t.references :テーブル名, foreign_key: true
```

- 『t.referencesカラム』が他のテーブルを参照していることを示す。
    
    カラム名は、指定したテーブルの前に『id_』が付く。
    
- `foreign_key: true`は外部キーであることを示している。
    
    親テーブルに対象が存在しない場合、子がテーブルへの保存ができないようにする。
    

※references型で作成すると、インデックスを自動で貼ってくれる。

### BoardモデルにUserモデルとのアソシエーションと、バリデーションを設定

- 掲示板を作成したユーザーとの関連を`belongs_to`で設定する。
- 「タイトルと本文の必須入力」と「文字数制限」のバリデーションを設定する。

```
#app/models/board.rb

class Board < ApplocationRecord
	belongs_to :user

	validates :title, presence: true, length: { maximum: 255 }
	validates :body, presence: true, length: { maximum: 65_535 }
end
```

```
$ bundle exec rails db:migrate
```

- マイグレーションする。

### アソシエーションの使い方

- モデルを関連付けしたことで、必要なオブジェクトを呼び出すメソッドが使えるようになる。
    
    具体的には、あるユーザーの投稿一覧が欲しい時には、`user.boards`というメソッドを使えば掲示板オブジェクトを取得できる。
    
- この投稿に紐付いているユーザーをオブジェクトとして欲しい時には、`boards.user`というメソッドで取得することができるようになる。

### Fakerを使ってダミーデータを作成する

[Fakerを使ってダミーデータを作成する方法](https://www.notion.so/Faker-6403f832b93b43dfbc5fdbf7dccefd3b?pvs=21)

# 掲示板一覧画面のルーティングを設定

```
#config/routes.rb

Rails.application.routes.draw do
	root 'static_pages#top'

	get 'login', to: 'user_sessions#new'
	post 'login', to: 'user_sessions#create'
	delete 'logout', to: 'user_sessions#destroy'

	resources :users, only: %i[new create]
	#これ↓
	resources :boards, only: %i[index]
end
```

# 掲示板一覧のコントローラーを作成

```
$ bundle exec rails g controller boards
```

- 生成されたBoardsControllerに、掲示板の一覧をデータベースから取得する処理を追加する。

```
#app/controllers/boaeds_controller.rb

class BoardController < ApplicationController
	def index
		@boards = Board.all.includes(:user).order(created_at: :desc)
	end
end
```

- 『includesメソッド』は、関連付いたモデルのデータを先に取得するメソッド。
    
    このメソッドを使うことで、Boardモデルからデータを取得する際に、関連するUserモデルのデータもまとめて取得してくれる。
    
    なぜこのような書き方をするのかというと、N+1問題を起こさないため。
    

# 掲示板一覧画面のビューファイルを作成

- 掲示板一覧画面の表示方法は、大本のindex.html.erbを作成して、単一の掲示板表示部分は_board.html.erbというパーシャルファイルを作成し、index.html.erbから繰り返し呼び出して掲示板一覧を表示する設計する。
    
    このときN +1問題が起きないように注意。
    

### 大本の掲示板一覧画面を作成

```
#app/controllers/index.html.erb

<div class="container pt-3">
	<div class="row">
		<div class="col-lg-10 offset-lg-1">
			<!-- 検索フォーム --!>
			<form>
				<div class="input-group mb-3">
				<input class="form-control" placeholder="検索ワード" type="search"/>
				<div class="input-group-append">
					<input type="submit" value="検索" class="btn btn-primary"/>
				</div>
			</div>
		</form>
	</div>
</div>

<!-- 掲示板一覧 --!>
<div class="row">
	<div class="col-12">
		<div class="row">
			#ここ↓は、掲示板が存在しない場合「掲示板がありません」と表示させている。
			<% if @boards.present? %>
				<%= render @boards %>
			<% else %>
				<p><% t('.no_result') %></p>
			<% end %>
			#ここ↑まで
		</div>
	</div>
</div>
</div>
```

### 単一の掲示板を表示するパーシャルを作成

```
#app/views/boards/_board.html.erb

<div class="col-sm-12 col-lg-4 mb-3">
  <div id="board-id-<%= board.id %>">
    <div class="card">
      <%= image_tag 'board_placeholder.png', class: 'card-img-top', size: '300x200' %>
      <div class="card-body">
        <h4 class="card-title">
          <a href="#">
            <%= board.title %>
          </a>
        </h4>
        <div class='mr10 float-right'>
          <a href="#"><%= icon 'fas', 'trash', class: 'pr-1' %></a>
          <a href="#"><%= icon 'fa', 'pen' %></a>
        </div>
        <ul class="list-inline">
          <li class="list-inline-item">
            <%= icon 'far', 'user' %>
            <%= board.user.decorate.full_name %>
          </li>
          <li class="list-inline-item">
            <%= icon 'far', 'calendar' %>
            <%= l board.created_at, format: :long %>
          </li>
        </ul>
        <p class="card-text"><%= board.body %></p>
      </div>
    </div>
  </div>
</div>
```

### **ヘッダーに掲示板一覧画面へのリンクを追記**

```
#shared/_header.html.erb

<div class="dropdown-menu dropdown-menu-right">
	#boards_pathを追加
  <%= link_to t('boards.index.title'), boards_path, class: 'dropdown-item' %>
  <%= link_to '掲示板作成', '#', class: 'dropdown-item' %>
</div>
```

### **ロケールファイルに追記**

```
#locales/activerecord/ja.yml

ja:
  activerecord:
    attributes:
      board:
        title: 'タイトル'
        body: '本文'
```

# ****ログインしていなければ掲示板一覧を表示できなくする****

- ユーザーがログインしていなければ、掲示板一覧を表示できないようにして、ログイン画面へリダイレクトさせる。
- ここでは、`require_login`というメソッドをアクションの前に呼ばれるフィルタとして登録する。
    
    アクションの前に呼ばれるフィルタを登録するには、`before_action`メソッドを使う。
    
    ```
    #app/controllers/appllication_controller.rb
    
    class ApplicationController < ActionController::Base
      add_flash_types :success, :info, :warning, :danger
      before_action :require_login
    
      private
    
      def not_authenticated
        redirect_to login_path, warning: t('defaults.message.require_login')
      end
    end
    ```
    
- `require_login`　メソッド
    
    sorceryで定義されている。
    
    ログイン状態を判定し、ログインしていなければ結果的に`not_authenticated` メソッドを実行する。
    
- `not_authenticated` メソッド
    
    このメソッドもsorceryで定義されているメソッド。
    
    デフォルトでは`root_path`へリダイレクトさせているので、`login_path`へと設定し直している。
    
    また、`warning` を使って「ログインしてください」というフラッシュメッセージが出るようにする。
    

# ****未ログイン状態でもトップページ、ユーザー新規登録画面、ログイン画面を表示できるようにする****

- `ApplicationController` の`before_action` で、`require_login` のフィルタを登録したため、未ログインでもアクセス可能なアクションでは、`require_action` をskipするように設定する。

### **トップページのフィルタをスキップする**

```
#controllers/static_pages_controller.rb

class StaticPagesController < ApplicationController
  skip_before_action :require_login, only: %i[top]

  def top; end
end
```

### **ログイン画面のフィルタをスキップする**

- ログイン後の遷移先を掲示板一覧に変更しておく。

```
#controllers/user_sessions_controller.rb

class UserSessionsController < ApplicationController
   skip_before_action :require_login, only: %i[new create]
 
   def new; end
 
   def create
     @user = login(params[:email], params[:password])
     if @user
       redirect_back_or_to boards_path, success: t('.success')
     else
```

### **新規登録画面のフィルタをスキップする**

```
#controllers/user_sessions_controller.rb

class UsersController < ApplicationController
  skip_before_action :require_login, only: %i[new create]
```

- これでログイン成功後は掲示板一覧ページに遷移し、非ログイン時はログイン画面にダイレクトされるようになった。

# ****タイムゾーンを日本時刻に設定する****

```
#config/application.rb

config.time_zone = "Tokyo"
config.active_record.default_timezone = :local
```

- `config.time_zone` Rails自体のアプリケーションの時刻の設定
    
    `config.active_record.default_timezone` データベースを読み書きする際に、データベースに記録されている時間をどのタイムゾーンで読み込むかの設定
