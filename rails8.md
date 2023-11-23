# 実装の流れ

1. UserモデルにBoardモデルとのアソシエーションを追加。
2. ルーティングの設定。
3. コントローラーの設定。
4. ビューファイルの作成。

# ****UserモデルにBoardモデルとのアソシエーションを追加****

- user.rbにアソシエーションの定義を設定する。

```
#user.rb

has_many :boards, dependent: :destroy
```

# ルーティングの設定

- 掲示板作成のためのアクションを追加する。

```
#routes.rb

resources :boards, only %i[index new create]
```

# コントローラーに追記

```
#controllers/boards_controller.rb

class BoardsController < ApplicaationController

	def index
		#Board.allのallはRuby on RailsのActiveRecordで使われるメソッド。
		#これはBoardモデルに関連するすべてのレコードをデータベースから取得するためもの。
		@boards = Board.all.includes(:user).order(created_at: :desc)
	end

	def new
		@board = Board.new
	end

	def create
		#buildメソッドはその関連付けられたオブジェクトの集合に新しいインスタンスを追加するために使われる。
		#ログインしているユーザーに関連づけられた新しいBoardインスタンスを作成し、フォームデータから安全にパラメータを受け取り、それをその新しいボードインスタンスの属性に設定するという流れになる。
		@board = current_user.boards.build(board_params)
		if @board.save
			redirect_to boards_path, success: t('defaults.message.created', item: Board.model_name.human)
		else
			flash.now['danger'] = t('defaults.message.not_created', item: Board.model_name.human)
			rendr :new
		end
	end

	def show
		@board = Board.find(params[:id])
	end

	private

	def board_params
		params.require(:board).permit(:title, :body)
	end
end
```

# ビューファイルの作成

### 掲示板新規作成画面

```
#views/boards/new.html.erb

<div class="container">
	<div class="row">
		<div class="col-lg-8 offset-lg-2">
			<h1><%= t('.title') %></h1>
			<%= render 'from', { board: @board } %>
		</div>
	</div>
</div>
```

### 掲示板フォーム画面のパーシャル

```
#views/boards/_from.html.erb

<%= form_with model: board, local: true do |f| %>
	<div class="form-group">
		<%= f.label :title %>
		<%= f.text_field :title, class: 'form-control' %>
	</div>
	<div class="form-group">
		<%= f.label :body %>
		<%= f.text_area :body, class: 'form-control', rows: 10%>
	</div>

	<%= f.submit class: 'btn btn-primary' %>
<% end %>
```

- フォームのテキストエリアの入力幅はstyleで指定するのではなく、`rows:`で設定する。
    
    `rows:`入力欄の高さを行数で指定する。
    
    入力内容がこの高さを超えた場合は、入力欄にスクロールバーが表示される。
    

### ヘッダーの掲示板作成画面へのリンクを編集

```
#views/shared/_header.html.erb

<%= link_to t('boards.new.title'), new_board_path, class: 'dropdown-item' %>
```
