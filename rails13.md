# 実装にあたって必要な作業

1. ルーティングの設定
2. コントローラーの設定
3. ビューファイルの設定

# ルーティングの設定

- `edit`、`update`、`destroy`アクションを追加する

```
#config/routes.rb

Rails.application.routes.draw do
	resources :boards, only: %i[index new create show edit update destroy] do
		resources :comments, only: %i[create], shallow: true
end
```

- resorcesすべてのアクションを使うことになったので、`resources :boards do`で記載しても良い。

```
#config/routes.rb

Rails.application.routes.draw do
	resources :boards do
		resources :comments, only: %i[create], shallow: true
end
```

# コントローラーの設定

```
#記入例
#app/controllers/boards_controller.rb

class BoardsController < ApplicationController
	#「edit,pdate,destroy」については『set_boardアクション』を介して掲示板を取得する設定。
　before_action :set_board, only: %i[edit update destroy]

~~~~~~~~~~~~~~~~~一部省略~~~~~~~~~~~~~~~~~~~~

	
  def edit; end

  def update
    if @board.update(board_params)
      redirect_to boards_path(@board), success: t('defaults.message.updated', item: Board.model_name.human)
    else
      flash.now[:danger] = t('defaults.message.not_updated', item: Board.model_name.human)
      render :edit
    end
  end

  def destroy
		#!を付けることで例外を発生させる。万が一処理に失敗した場合は、システムエラーとして処理を中断させておかしな挙動のまま機能を使わせないようにするため。
    @board.destroy!
    redirect_to boards_path, success: t('defaults.message.deleted', item: Board.model_name.human)
  end

  private

  def set_board
    @board = current_user.boards.find(params[:id])
  end

  def board_params
    params.require(:board).permit(:title, :body, :board_image, :bord_image_cache)
  end
```

- `redirect_to @board` は`redirect_to board_path(@board)` の省略形。

# ビューファイルの設定

### ****編集ボタン、削除ボタンの実装／表示****

- 編集のリンクを、掲示板編集画面へ遷移するようにパスを変更する。
- 削除リンクを、パスを変更、確認アラートが表示されるように記述を追加する。

```
#app/views/boards/_board.rb

<div class='d-flex justify-content-end'>
 <%= link_to edit_board_path(board), id: "button-edit-#{board.id}" do %>
  <i class="bi bi-pencil-fill"></i>
   <% end %>
 <%= link_to board_path(board), id: "button-delete-#{board.id}", data: { turbo_method: :delete, turbo_confirm: t('defaults.delete_confirm') } do %>
  <i class="bi bi-trash-fill"></i>
   <% end %>
</div>
```

```
#app/views/boards/show.html.erb

<div class='d-flex justify-content-end'>
 <%= link_to edit_board_path(@board), id: "button-edit-#{@board.id}" do %>
  <i class="bi bi-pencil-fill"></i>
 <% end %>
 <%= link_to board_path(@board), id: "button-delete-#{@board.id}", data: { turbo_method: :delete, turbo_confirm: t('defaults.delete_confirm') } do %>
  <i class="bi bi-trash-fill"></i>
 <% end %>
</div>
```

### 掲示板編集画面の作成

```
#app/views/boareds/edit.html.erb

<% content_for(:title, @board.title) %>
<div class="container">
  <div class="row">
    <div class="col-lg-8 offset-lg-2">
      <h1><%= t('.title') %></h1>
      <%= render 'form', { board: @board } %>
    </div>
  </div>
</div>
```

- `<%= render 'form', { board: @board } %>` は`app/views/boards/_form.html.erb` の情報を引っ張ってきて表示している。
