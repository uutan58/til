# 掲示板詳細画面の追加、コメント機能について

- 掲示板一覧から掲示板詳細画面へ遷移できるようにする。
- 掲示板詳細画面から、コメントを書き込めるようにする。
- コメントした本人だけ削除・編集ボタンを表示するよう制限する。

# ****作業の流れ****

1. Commentモデルを作成
2. コメントのルーティングを設定
3. commentsコントローラを作成
4. boards_controllerへの追記
5. 掲示板の編集と削除のボタンは部分パーシャル化する
6. 個々の掲示板表示パーシャル内で編集と削除ボタンのパーシャルを組み込む
7. 掲示板詳細画面の作成
8. コメントフォームのパーシャルを作成
9. 各コメント部分のパーシャルを作成

# ****Commentモデルを作成****

- UserとBoardモデルに関連付けさせるには、`references`で指定する。
    
    なぜなら、普通の`$ bundle exec rails generate model Comment body:text user_id:integer board_id:integer` という書き方では、
    
    - インデックスが貼られない。
    - 外部キー制約が貼られない。
    
    などの問題が生じるから。
    

```
$ bundle exec rails g model Comment body:text user:refetences board:references
```

### 作成されたマイグレーション

```
#db/migrate/YYYYMMDD_create_comments.rb

class CreateComments < ActiveRecord::Migration[7.0]
  def change
    create_table :comments do |t|
      t.references :user
      t.references :board
      t.text :body

      t.timestamps
    end
  end
end
```

- rails7.0では、マイグレーションの際にオプションを記述しなくても、自動的に適切な設定が行われるようになった。

### ****commentモデルに、bodyのバリデーションを追加****

```
#app/models/comment.rb

class Comment < ApplicationRecord
  belongs_to :user
  belongs_to :board

	#これ↓を追加
  validates :body, presence: true, length: { maximum: 65_535 }
end
```

### ****BoardモデルとUserモデルにコメントとの関連を追加****

```
#app/models/board.rb

class Board < ApplicationRecord
  mount_uploader :board_image, BoardImageUploader
  
	belongs_to :user
	#これ↓を追加
  has_many :comments, dependent: :destroy

  validates :title, presence: true, length: { maximum: 255 }
  validates :body, presence: true, length: { maximum: 65_535 }
end
```

- `has_many`はテーブル同士を関連づけるもの
- 【*has_many*(Board)は*belongs_to*(Comment)を何個も投稿できるよ！】みたいなイメージ。
- `dependent: :destroy`オプションをつけることによって「commentに紐づいたboardが消されたら該当commentも削除」という作業を行なってくれる。

```
#app/models/user.rb

class User < ApplicationRecord
  authenticates_with_sorcery!

  has_many :boards, dependent: :destroy
	#これ↓を追加
  has_many :comments, dependent: :destroy
```

- Boardモデルと同様。

```
$ bundle exec rails db:migrate
```

- マイグレートを実施し、データベースに反省させる。

※モデルにルールを追加したら、それをデータベースにも伝えるために`bundle exec rails db:migrate`という魔法の言葉を使うのです！

# コメントのルーティングを設定

- ルーティングをネストして、`boards`と`comments`との親子関係を作る。
    
    今回必要なのはcreateアクションだけなので、`%i[create]` とする。
    
- ルーティングをネストすることで親のidを子のパスに含めることができる。
- idをパスに含めることでアソシエーション先のレコードのid(ここではbord_id)を`params`に追加してコントローラーに送り、コントローラーで取得することができるようになる。
- 掲示板詳細画面を作成する際に『show.html.erb』のファイルを作成する。
    
    そのため、boardsに`show`を追加する。ファイルを作成したタイミングでも良い。
    

```
Rails.application.routes.draw do
  root 'static_pages#top'

  get 'login', to: 'user_sessions#new'
  post 'login', to: 'user_sessions#create'
  delete 'logout', to: 'user_sessions#destroy'

  resources :users, only: %i[new create]
																						#これ↓を追加
  resources :boards, only: %i[index new create show] do
		#これ↓を追加
    resources :comments, only: %i[create]
  end
end
```

- ルーティングをネストする際は、`do`〜`end`を付けないといけない。

```
#記入例
Rails.application.routes.draw do											↓これ															↓これ
	resources :boards, only: %i[index new create show] do
		resources :comments, only: %i[create]
  end
	 ↑これ
end
```

# ****commentsコントローラを作成する****

- 以下のコマンドを入力することで`comments_controller` が生成する

```
$ bundle exec rails g controller comments
```

### ****comments_controllerを設定****

```
#app/controllers/comments_controller.rb

class CommentsController < ApplicationController
	def create
		comment = current_user.comments.build(comment_params)
		if comment.save
			#①解説あり
			redirect_to board_path(comment.board), success: t('defaults.message.create', item: Comment.model_name.human)
		else
			redirect_to board_path(comment.board), danger: t('defaults.message.create', item: Comment.model_name.human)
		end
	end

	private

	#②解説あり
	def comment_params
		params.require(:comment).permit(:body).merge(board_id: params[:board_id])
	end
end
```

# ****boards_controllerへの追記****

- コメントの新規作成は投稿の詳細で行うので、掲示板詳細画面（views/show.html.erb)でコメントの新規作成フォーム(`@comment`使う)とコメント一覧(`@comments`使う)を表示させる。

```
#app/controllers/borads_controller.rb

def show
	@board = Board.find(params[:id])
	@comment = Comment.new
	@comments = @board.comments.includes(:user).order(created_at: :desc)
end
```

# ****掲示板の編集と削除のボタンは部分パーシャル化する****

- 掲示板の編集と削除のボタンは、掲示板の一覧と詳細ページで同じものを表示するので、部分テンプレートとして作成する。

```
#app/views/boards/_board.html.erb

          <div class='ms-auto'>
            <%= link_to '#', id: "button-edit-#{board.id}" do %>
              <i class="bi bi-pencil-fill"></i>
            <% end %>
            <%= link_to '#', id: "button-delete-#{board.id}" do %>
              <i class="bi bi-trash-fill"></i>
            <% end %>
          </div>
```

# ****掲示板詳細画面の作成****

```
#app/views/boards/show.html.erb

<div class="container pt-5">
  <div class="row mb-3">
    <div class="col-lg-8 offset-lg-2">
      <h1><%= t('.title') %></h1>
      <article class="card">
        <div class="card-body">
          <div class="row">
            <div class="col-md-3">
              <%= image_tag @board.board_image_url, width: "300", height: "200", class: "card-img-top img-fluid" %>
            </div>
            <div class="col-md-9">
              <h3 style="display: inline;"><%= @board.title %></h3>
              <ul class="list-inline">
                <li class="list-inline-item"><%= "by #{@board.user.decorate.full_name}" %></li>
                <li class="list-inline-item"><%= l @board.created_at, format: :long %></li>
              </ul>
              <div class='d-flex justify-content-end'>
                <%= link_to "#", id: "button-edit-#{@board.id}" do %>
                  <i class='bi bi-pencil-fill'></i>
                <% end %>
                <%= link_to "#", id: "button-delete-#{@board.id}", class: "ms-2" do %>
                  <i class="bi bi-trash-fill"></i>
                <% end %>
              </div>
              </div>
            </div>
            <p><%= simple_format(@board.body) %></p>
          </div>
        </article>
      </div>
    </div>
    <%= render 'comments/form', comment: @comment, board: @board %>
    <div class="row">
      <div class="col-lg-8 offset-lg-2">
        <table class="table">
          <tbody id="table-comment">
            <%= render @comments %>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</div>
```

# コメントフォームのパーシャルを作成

- 原則としてパーシャルは再利用性を高めるために**ローカル変数を使用する**。
    
    インスタンス変数を使うとなると、コントローラと関連付いてしまうため、再利用性が低くなります。
    
    エラーメッセージを呼び込むパーシャルも実装する。
    

```
#app/views/comments/_form.html.erb

<div class="row mb-3" id="comment-form">
  <div class="col-lg-8 offset-lg-2">
    <%= form_with model: comment, url: board_comments_path(board) do |f| %>
      <%= f.label :body %>
      <%= f.text_area :body, class: "form-control mb-3", row: "4", placeholder: Comment.human_attribute_name(:body) %>
      <%= f.submit t('defaults.post'), class: "btn btn-primary" %>
    <% end %>
  </div>
</div>
```

# 各****コメント部分のパーシャルを作成****

```
#app/views/comments/_comment.html.erb

<tr id="comment-<%= comment.id %>">
  <td style="width: 60px">
    <%= image_tag "sample", width: "50", height: "50", class: "rounded-circle" %>
  </td>
  <td>
    <h3 class="small"><%= comment.user.decorate.full_name %></h3>
    <p><%= simple_format(comment.body) %></p>
  </td>
  <% if current_user.own?(comment) %>
    <td class="action">
      <ul class="list-inline justify-content-center" style="float: right;">
        <li class="list-inline-item">
          <%= link_to "#", class: "edit-comment-link" do %>
            <i class="bi bi-pencil-fill"></i>
          <% end %>
        </li>
        <li class="list-inline-item">
          <%= link_to "#", class: "delete-comment-link" do %>
            <i class="bi bi-trash-fill"></i>
          <% end %>
        </li>
      </ul>
    </td>
  <% end %>
</tr>
```

### ****コメントした本人だけにのみ編集・削除ボタンを表示する****

```
#app/models/user.rb

def own?(object)
    id == object&.user_id
end
```

- コメントの編集・削除ボタン表示の判定する時の条件分岐は、Userモデルにインスタンスメソッドとして記載する。
