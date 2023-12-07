# kaminariを使用する

```
#Gemfile

gem 'kaminari'
```

```
$ bundle install
```

# configファイルを生成する

```
$ rails g kaminari:config
```

- 生成したファイルに1ページあたりに表示するレコード数を設定する。

```
＃kaminari_config.rb
Kaminari.configure do |config|
	config.default_per_page = 20
end
```

# indexアクションでpageスコープを使用する

```
#boards_controller.rb

def index　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　#↓ここを追加
	@boards = Board.all.includes(:user).order(created_at: :desc).page(params[:page]
end
```

- 他のページに表示させる場合も、上記のようにアクション内にある「@〇〇」の欄に追記する。

# paginateヘルパーを使用して画面に表示させる

```
#views/boards/index.html.erb

<%= paginate @boards %>
```

```
#記入例
#views/boards/index.html.erb

<% content_for(:title, t('.title')) %>
<div class="container pt-3">
  <div class="row">
    <div class="col-lg-10 offset-lg-1">
      <!-- 検索フォーム -->
        <%= render 'search_form', q: @q, url: boards_path %>
        </div>
      </form>
    </div>
  </div>
  <!-- 掲示板一覧 -->
  <div class="row">
    <div class="col-12">
      <div class="row">
        <% if @boards.present? %>
          <%= render @boards %>
        <% else %>
          <div class="mb-3">掲示板がありません</div>
        <% end %>
        <%= paginate @boards, theme: 'bootstrap-5' %>
      </div>
      </div>
    </div>
  </div>
</div>
```

- 表示スタイルを変更する場合は`theme: 'bootstrap-5’`のように任意のものを記載する。

```
#Gemfile
gem'bootstrap5-kaminari-views'
```

```
$ bundle install
```

- 上記のようにGemを追加してから使用する。

※gemをインストールしてから、ヘルパー以下に追記しないと反映されない…？
