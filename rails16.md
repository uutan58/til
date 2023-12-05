# turbo_streamを使用してコメント投稿を実装

```
#comments/create.turbo_stream.erb

<% if @comment.errors.present? %>
  <%= turbo_stream.replace "comment-form" do %>
    <%= render 'comments/form', comment: @comment, board: @comment.board %>
  <% end %>
<% else %>
  <%= turbo_stream.prepend "table-comment" do %>
    <%= render 'comments/comment', comment: @comment %>
  <% end %>
  <%= turbo_stream.replace "comment-form" do %>
    <%= render 'comments/form', comment: Comment.new, board: @comment.board %>
  <% end %>
<% end %>
```

- コメント投稿成功時にフォームの入力欄が空欄になっていること。
- 今回はコメントの空のインスタンスをパーシャルに渡す方法で行っている。

```
<%= turbo_stream.replace "comment-form" do %>
	<%= render 'comments/form', comment: Comment.new, board: @comment.board %>
<% end %>
```

# turbo_streamを使用してコメント削除を実装

```
#comments/destroy_turbo_stream.erb

<%= turbo_stream.remove "comment-#{@comment.id}" do %>
<% end %>
```

- removeメソッドを使って対象のIDを持つ要素を画面上から削除する。

# shalloルーティングの設定

```
#routes.rb

resources :boards, shallow: true do
```

- 全てのネストしたリソースが浅くなるため。
- コールバックメソッドの実行対象のアクションはexcept ではなくonlyで記載すること。
- アクションが増えた場合、そのアクションに対してメソッドが実行されることを防ぐため。

# コントローラーの設定を変更

```
#controllers/comments_controller.rb

class CommentsController < ApplicationController
  def create
	#↓変更点
    @comment = current_user.comments.build(comment_params)
    @comment.save
  end

  def destroy
	#↓変更点
    @comment = current_user.comments.find(params[:id])
    @comment.destroy!
  end
```

参照：**[猫でもわかるHotwire入門 Turbo編](https://zenn.dev/shita1112/books/cat-hotwire-turbo)**
