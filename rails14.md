# 実装の流れ

1. Bookmarkモデルを作成する
2. バリデーションの追加
3. アソシエーションの設定
4. ルーティングの設定
5. コントローラの設定
6. ビューファイルの作成

# ****Bookmarkモデルを作成する****

- 「どのユーザー」が「どの投稿」をブックマークしたかを記録する為に、データベースに「user_id」と「post_id」の2つのカラムを持つ`bookmarkテーブル` を用意する。

```
$ rails g model Bookmark user:references board:references
```

- 生成されたマイグレーションファイル

```
#2023XXXXXXXXXX_create_bookmarks.rb

class CreateBookmarks < ActiveRecord ::migration[7.0]
	def change
		create_table :bookmarks do |t|
		t.references :user, forign_key: true
		t.references :board, forign_key: true

		t.timestamps
		end
		add_index :bookmarks, [:user_id, :board_id], unique: :true
	end
end
```

- もし`user`と`board`の組み合わせのレコードが重複すると、ブックマークを外しても関係性のレコードが残り、ブックマークが解除できなくなってしまう。
    
    そのため、ユーザーと掲示板の組み合わせのレコードを一意にするための**unique制約**が必要
    

# バリデーションの追加

```
#bookmark.rb

class Bookmark < ApplicationRecord
	belongs_to :user
	belongs_to :board

	validates :user_id, presence: true, uniqueness: { scope: :board_id }
end
```

- `uniqueness: { scope: :board_id }` とする。
    
    こうすることで、**ブックマークは掲示板に対してユーザ一人までですよ。**という意味合いを持たせることができる。
    
    `scope`をつけないと、テーブル全体で一意な数字という意味になってしまう。
    

```
$ rails db:migrate
```

- マイグレーションして、設定を反映させる。

# アソシエーションの設定

### Boardモデルの設定

```
#board.rb

has_many :bookmarks, dependent: :destroy
```

- 掲示場とブックマークは1対多の関係になる。
    
    `has_many:bookmark` で設定し、掲示板が削除されたらブックマークも削除されるように`dependent: :destroy` を設定する。
    

### Userモデルの設定

```
#user.rb

has_many :bookmarks, dependent: :destroy
has_many :bookmarks_boards, through: :bookmarks, source: :board
```

- ユーザーとブックマークも1対多の関係なので、`has_many`を設定。
- `:source`オプションは、`has_many :through`関連付けにおける「ソースの」関連付け名、つまり関連付け元の名前を指定する。
    
    このオプションは、関連付け名から関連付け元の名前が自動的に推論できない場合以外には使う必要はない。
    
- `bookmark_boards`は`source:` オプションを使って架空の`bookmark_boards` を定義している。
    
    **つまり、Userモデルから`bookmark_boards`というメソッドを使用すると、`through:`によって定義された`bookmarks`モデルを介して、`board` にアクセスできるということ。**
    
    この設定によって`user.bookmark_boards` のような形でブックマークしている`Board`インスタンスを配列として取得できるようになる。
    
    具体的に内部で行われていることは、Userモデルのインスタンスに`bookmarks`メソッドを実行し、得られたbookmarksレコードひとつひとつに対して`board`メソッドを実行している。
    

# ルーティングの設定

```
#config/routes.rb

Rails.application.routes.draw do
	root 'static_pages#top'
	
	get 'login', to: 'user_sessions#new'
	post 'login', to: 'user_sessions#create'
	delete 'logout', to: 'user_sessions#destroy'

	resources :users, only: %i[new create]
	resources :boards do
		resources :comments, only: %i[create], shallow: true
			#ここ↓が大事
			collection do
				get 'bookmarks'
			end
	end
		resources :bookmarks, only: %i[create destroy]
end
```

- index/new/createのような、idを持たないようなアクションのことを`コレクション` と呼び、show/edit/update/destroyのような、idを必要とするアクションのことを`メンバー` と呼ぶ。
- boards_controllerに新しく`bookmarks` というアクションを追加したく、prefixパスも`bookmarks_boards_path` のようにidを伴わないパスにしたい場合、collectionを使用しようする。
- • `/boards/bookmarks`とすることでURLから何を表示しているか推測しやすくしたいため。

# コントローラーの設定

### ****BookmarksControllerを作成****

- ビューファイルは必要ないので、手動で作成した方が良い。

### ****Userモデルにブックマーク処理のロジックを定義する****

```
#user.rb

def bookmark(board)
	bookmark_boards << board
end

def unbookmark(book)
	bookmark_boards.destroy(board)
end

def bookmark?(board)
	bookmark_boards.include?(board)
end
```

- bookmarkメソッド
    
    ユーザの`bookmark_boards` （ブックマークした掲示板）から渡された引数の`board` の中にある`board_id` を参照して探す。
    
    無ければ`<<` メソッドによって末尾に格納され、`bookmarks` テーブルに保存される。`save` メソッドは必要ない。
    
- unbookmarkメソッド
    
    ユーザの `bookmark_boards`（ブックマークした掲示板）から渡された引数の`board` の中にある`board_id` を参照して探す。存在すれば削除するメソッド 。
    
- bookmark?メソッド
    
    ユーザの`bookmark_boards` （ブックマークした掲示板）から渡された引数の`board` の中にある`board_id` を参照して探す。
    
    `include?` メソッドは、引数(`board`)が存在すれば`true` を返し、無ければ`false` を返す。
    

### ****bookmarks_controllerに追記****

```
#controllers/bookmarks_controller.rb

class BookmarkController < ApplicationController
	def create
		board = Board.find(params[:board_id])
		current_user.bookmark(board)
		redirect_back fallback_location: root_path, success: t('.success')
	end

	def destroy
		board = current_user.bookmarks.find(params[:id]).board
		current_user.unbookmark(board)
		redirect_back fallback_location: root_path, success: t('defaults.message.unbookmark')
	end
end
```

- `redirect_back` は、直前のページにリダイレクトする。`fallback_location` は直前のページがない場合のリダイレクト先を指定する。
- なぜ `params[:id]` を使った取得の仕方をしているかというと、destroyアクションへのルーティングが、`bookmarks/:id` となっているから。
- `current_user`に紐ついたブックマークレコードをパラメータ(bookmarks/:id)から取り出して、後ろの`.board` によって、`board_id` を参照し、そのブックマークレコードに紐ついた掲示板レコードを取得している。

### ****Board_controllerに追記****

```
def bookmarks
 @bookmark_boards = current_user.bookmark_boards.includes(:user).order(created_at: :desc)
end
```

- コレクションルーティングを追加した`bookmarks`アクションをboards_controllerに追加する。
- `includes(:user)` を使って関連するuserの情報も取得する＝キャッシュを取得するという。

# ビューファイルの作成

### ビューファイルの作業手順

- 掲示板一覧画面の中でブックマーク追加, ブックマーク削除を切り替えるパーシャルを用意する(`_bookmark_button.html.erb`)
- ブックマーク追加/解除ボタンをそれぞれパーシャルで用意する(`_bookmark.html.erb` / `_unbookmark.html.erb`)
- 自分の掲示板にはブックマークボタンが表示されないように実装する
- ブックマークした掲示板を一覧表示する画面を用意する(`bookmarks.html.erb`)

### ****ブックマーク追加, ブックマーク削除を切り替えるパーシャルを作成****

```
#boards/_bookmark_button.html.erb

<div class='ms-auto'>
  <% if current_user.bookmark?(board) %>
    <%= render 'unbookmark', { board: board } %>
  <% else %>
    <%= render 'bookmark', { board: board } %>
  <% end %>
</div>
```

- user.rbで定義した`bookmark?` メソッドがここで使われている。
    
    もし掲示板がブックマークされていたらブックマーク解除ボタンを表示し、ブックマークされていなければブックマーク登録ボタンを表示するように切り分けている。
    
- これが、ブックマークを切り替えるスイッチになる。

### ****ブックマーク登録ボタンのパーシャルを作成****

```
#boards/_bookmark.html.rb

<%= link_to bookmarks_path(board_id: board.id), id: "bookmark-button-for-board-#{board.id}", data: { turbo_method: :post } do %>
  <i class="bi bi-star"></i>
<% end %>
```

- Rails7.0仕様なので、`, data: { turbo_method: :post }` を付けないとエラーが出る。
- id内で”js-bookmark〜”と記載した例が多く出てくるが、これがあるとルーティングがうまくいかない？

### ****ブックマーク解除ボタンのパーシャルを作成****

```
#boards/_unbookmark.html.erb

<%= link_to bookmark_path(current_user.bookmarks.find_by(board_id: board.id)), id: "unbookmark-button-for-board-#{board.id}", data: { turbo_method: :delete } do %>
  <i class="bi bi-star-fill"></i>
<% end %>
```

- Rails7.0仕様なので、`data: { turbo_method: :delete }` を付けないとエラーが出る。
- `current_user`に紐ついているbookmarksレコードの中から、渡された`board` を使って掲示板の`id(board_id)`を探し出し、**bookmarkレコードを**取得。（掲示板レコードではない。なぜならこのあと生成されているdestroyアクションのURLパスの/bookmarks/`:id` から、bookmarkコントローラのdestroyアクションでbookmarkレコードのidを取得したいから、bookmarkレコードを渡しておく必要がある。）
    
    そして、それを`bookmark_path` へのリクエストにパラメータとして渡す。
    

### ****自分の掲示板にはブックマークボタンが表示されないように実装する****

```
#boards/_board.html.erb

<% if current_user.own?(board) %>
	<%= render 'crud_menus', board: board %>
<% else %>
	<%= render 'bookmark_button', board: board %>
<% end %>
```

- 現在ログインしているユーザの掲示板かどうか判定しつつ、一致している場合は編集ボタンと削除ボタンのパーシャルを表示し、一致していない場合はブックマークボタンのっパーシャルを表示するように、_boardパーシャルに組み込む。
- 『_crud_menus.html.erb』は事前に作成しておく。
    
    中身は、編集・削除ボタンのパーシャル。
    
    これがないと、ブックマークボタンの実装でハマる。
    

※カリキュラムでは『_crud_menus.html.erb』は作成せず、直接『boards/_board.html.erb』に『bookmark_button』を記載している。

```
#boards/_crud_menus.html.erb

<div class='ms-auto'>
    <%= link_to edit_board_path(board), id: "button-edit-#{board.id}" do %>
        <i class="bi bi-pencil-fill"></i>
    <% end %>
    <%= link_to board_path(board), id: "button-delete-#{board.id}", data: { turbo_method: :delete, turbo_confirm: t('defaults.delete_confirm') } do %>
        <i class="bi bi-trash-fill"></i>
    <% end %>
</div>
```

### ****ブックマークした掲示板の一覧表示画面を作成****

```
#boards/bookmarks.html.erb

<% content_for(:title, t('.title')) %>
<div class="container pt-3">
  <div class="row">
    <div class="col-lg-10 offset-lg-1">
      <!-- 検索フォーム -->
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

  <!-- 掲示板一覧 -->
  <div class="row">
    <div class="col-12">
      <div class="row">
				#ここでブックマークした掲示板を一覧画面に表示している。
        <% if @bookmark_boards.present? %>
          <%= render @bookmark_boards %>
        <% else %>
          <p><%= t('.no_result') %></p>
        <% end %>
      </div>
    </div>
  </div>
</div>
```
