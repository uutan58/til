# 実装の流れ

1. carrierwaveのインストール
2. アップローダークラスの作成
3. アップロード画像のカラムを作成
4. ストロングパラメーターへの追記
5. 画像ファイルの入力フィールドを用意

# carrierwaveのインストール

```
#Gemfile

gem 'carrierwave', '~> 2.0'←任意のバージョンを記載
```

```
$ bundle install
```

# ****アップローダークラスの作成****

- Carrierwaveをインストールすることで、下記コマンドを実行することができるようになる。

```
bundle exec rails g uploader BoardImage
```

- コマンドを実行すると、以下の`app/uploaders/board_image_uploader.rb` が生成される。

```
#app/uploaders/board_image_uploader.rb

class BoardImageUploader < CarrierWave::Uploader::Base
	storage :file #デフォルトで指定されているので、アップロードしたファイルは『public/』配下に保存される。

	def store_dir　#生成時にはこのアクションのみ事前に記載されている
		"uploads/#{model.class.to_s.underscore}/#{mounted_as}/#{model.id}"
	end

	def default_url #デフォルトの画像ファイルの指定
		'board_placeholder'
	end

	def extension_whitelist #拡張子の制限
		%w[jpg jpeg gif png]
	end
end
```

### .gitignoreを編集

- public/uploads/配下に保存される画像は、Githubなどにアップロードする必要がないので、以下のように.gitignoreファイルに`/public/uploads`を指定してGit管理下から除外。

```
#.gitignore

/public/uploads
```

# ****アップロード画像のカラムを追加****

- アップロード画像の情報を保存する`board_image` カラムを追加するために、マイグレーションファイルを作成する。

```
$ bundle exec rails g migration AddBoardImageToBoards board_image:string
```

- 以下、作成されたマイグレーションファイル。

```
#2023XXXXXXXXXX_add_board_image_to_boards.rb

class AddBoardImageToBoards < ActiveRecord::Miggration[7.0]
	def change
		add_column :boards, :board_image, :string
	end
end
```

- マイグレーションを実行して、boardsテーブルに`board_image` カラムを追加する。

```
$ bundle exec rails db:migrate
```

- データベースに保存されるのは「画像データ」ではなく「画像のファイル名」。
    
    画像データを保存するとデータベースサーバーの容量を圧迫するから。
    

# ****アップローダークラスとカラムの紐付け****

- 投稿の際に画像も投稿する機能を追加したいので、boardモデルに先ほど作成した投稿画像用の「`board_image` カラム」と「`BoardImageUploader` クラス」を紐付ける。

```
#アップローダークラスとカラムの紐付けテンプレート

class モデル名 < ActiveRecord::Base
  mount_uploader [:カラム名], [アップローダークラス]
end
```

```
#記入例
#app/uploaders/board_image_uploader.rb

class Board < ApplicationRecord
	#この↓コード一行を追加する
	mount_uploader :board_image, BoardImageUploader
	belongs_to :user

	validates :title, presence: true, length: { maximum: 255 }
	validaes :body, presence: true, length: { maximum: 65_535 }
end
```

- boardモデルで画像を投稿する際の「board_imageカラム」の設定を「BoardImageUploaderクラス」で行えるようになる。

# ****ストロングパラメータに追記する****

- 画像ファイルをコントローラーで受けとれるようにストロングパラメータに追記する。

```
#app/controllers/boards_controller.rb

def board_params #『:board_image, :board_image_cache』を追加
	params.require(:board).permit(:title, :body, :board_image, :board_image_cache)
end
```

# ****画像ファイルの入力フィールドを用意する****

```
#app/views/boards/new.html.erb

<%= form_with model:board, local: true do |f| %>
  <%= render 'shared/error_messages', object: f.object %>
    <div class="form-group">
      <%= f.label :title %>
      <%= f.text_field :title, class: 'form-control' %>
    </div>
    <div class="form-group">
      <%= f.label :body %>
      <%= f.text_area :body, class: 'form-control', rows: 10 %>
    </div>
    <div class="form-group">
			#「board_image」カラムの入力項目を表示。
      <%= f.label :board_image %>
			#「**f.htmlタグ名 :カラム名**」と指定する。
			#**カラム名は保存される先のテーブルのカラム名を指定する**ので、「*board_image*」を指定。
			#これで、boardテーブルのboard_imageカラムに投稿した画像のデータが送られる。
			#*accept*で**ファイルの種類を指定**。image/は画像ファイル全般**。**
      <%= f.file_field :board_image, class: 'form-control mb-3', accept: 'image/*' %>
      #バリデーションに引っかかり、掲示板が登録できなかった時（エラーが出て入力画面に戻された時）に選んだ画像を保持しておく機能。
			#入力画面に戻された時に再度画像を選び直さなくていいように。
			<%= f.hidden_field :board_image_cache %>
    </div>
    <%= f.submit class: 'btn btn-primary' %>
<% end %>
```
# ****アップロードした画像を表示****

```
#app/views/boards/_board.html.erb

#after
<%= image_tag board.board_image.url, class: 'card-img-top', width: "300", height:"200" %>

#before
<%= image_tag "board_placeholder.png", class: "card-img-top", width: "300", height:"200" %>
```

- 保存した画像（board_image）を表示させるには、画像が保存されている場所(パス)を取得する必要がある。
- Boardモデルに、BoardImageUploaderクラスとboard_imageカラムを紐づけたことで「urlメソッド」が使えるようになっている。
- 「urlメソッド」　→ ファイルのURLを取得　「使い方の例：boardモデル.board_image.url」
- `board.board_image.url`でBoardモデルのboard_imageカラムからURLを取得している。

※アップローダーでデフォルトで表示される画像を設定しているので、画像が設定されていないとういう事が起きないため、`if`、`else`での画像が設定されていなかったらなどの分岐は必要ない。
