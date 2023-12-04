# turbo-railsをインストールする

```
#Gemfile

gem 'turbo-rails'
```

```
$ bundle install
```

# アクション名に合わせて`アクション名.turbo_stream.erb`を作成する

- gem turbo-railsを使用するとturboを使っている時のリクエストが送られた時に、自動的にアクション名と同名のturbo_stream.erbファイルを探して返す。

```
#create.turbo_strem.erb

<%= turbo_stream.replace "bookmark-button-for-board-#{@board.id}" do %>
	<%= render 'boards/unbookmark', board: @board %>
<% end %>
```

```
#destroy.turbo_strem.erb

<%= turbo_stream.replace "unbookmark-button-for-board-#{@board.id}" do %>
  <%= render 'boards/bookmark', board: @board %>
<% end %>
```

- replaceメソッドを使っていること。
    
    replaceを使うと引数に渡された文字列と同名のIDを持つ要素を書き変える。
    

# コントローラーの内容を変更する

```
#bookmarks_controller.rb

def create
	#@を付ける
	@board = Board.find(params[:board_id])
												#@を付ける
  current_user.bookmark(@board)
end

def destroy
		#@を付ける
    @board = current_user.bookmarks.find(params[:id]).board
														#@を付ける
    current_user.unbookmark(@board)
  end
```
