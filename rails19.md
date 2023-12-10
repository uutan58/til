# 実装の流れ

1. ルーティングの設定
2. CarrierWaveを使ってアップローダークラスを作成
3. アバター画像のカラムを追加
4. アップローダークラスとカラムの紐づけ
5. コントローラーの設定
6. ビューファイルの作成

# ルーティングの設定

```
#routes.rb

resource :profile, only: %i[show edit update]
```

- 

今回は一覧画面(`index.html.erb`)が必要なく、詳細画面(`show.html.erb`)へのURLがidを含む形(`profiles/:id`)だと自分が何番目に作成されたユーザーか外部から分かってしまう。

詳細画面へのURLのidの部分を他のユーザーのidに書き換えられる危険性もあるため、**単数形リソース**でルーティングを設定すること。

# ****CarrierWaveを使ってアップローダークラスを作成****

```
$ bundle exec rails g uploader Avatar
```

- コマンドを実行すると以下の`app/uploaders/board_image_uploader.rb`が作成される。

```
#uploaders/board_image_uploader.rb

class AvatarUploader < CarrierWave::Uploader::Base
  storage :file

  def store_dir
    "uploads/#{model.class.to_s.underscore}/#{mounted_as}/#{model.id}"
  end

  def default_url
    'sample.jpg'
  end

  def extension_whitelist # 拡張子の制限
    %w[jpg jpeg gif png]
  end
end
```

- 生成されたアップローダにデフォルト画像ファイルとアップロード可能な拡張子の設定を追記する。
- 上記のコードより、デフォルトで`storage :file` が指定されているので、アップロードしたファイルは`public/` 配下に保存される。
    
    `default_url` は画像を添付しなかった場合に、自動で登録されるわけではない。`@user.avatar.nil`などで呼び出して`nil`だった場合に、代わりに呼び出されるURLとなる。
    
    保存をしていないことで`default_url`の値を書き換えるだけで画像が更新される。
    

# ****アバター画像のカラムを追加****

```
$ rails g migrate add_avatar_to_users avatar:string
```

- usersテーブルに`avatar`カラムを追加する。

```
#2023XXXXXXXXXX_add_avatar_to_users.rb

class AddAvatarToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :avatar, :string
  end
end
```

```
$ rails db:migrate
```

- マイグレーションする。

# ****アップローダークラスとカラムの紐付け****

```
#app/models/user.rb

mount_uploader :avatar, AvatarUploader
```

- 「`avatar` カラム」と「`AvatarUploader` クラス」を紐付ける。

# ****コントローラの設定****

```
$ bundle exec rails g controller profiles
```

- 新たに`profiles_controller`を作成する。

```
#profiles_controller.rb

class ProfilesController < ApplicationController
  before_action :set_user, only: %i[edit update]

  def show;end

  def edit;end

  def update
    if @user.update(user_params)
      redirect_to profile_path, success: t('defaults.message.updated', item: User.model_name.human)
    else
      flash.now['danger'] = t('defaults.message.not_updated', item: User.model_name.human)
      render :edit
    end
  end

  private

  def set_user
    @user = User.find(current_user.id)
  end

  def user_params
    params.require(:user).permit(:email, :first_name, :last_name, :avatar, :avatar_cache)
  end
end
```

- 今回作成するprofilesコントローラはモデルと紐付かない。
    
    何故わざわざprofilesコントローラを作成して、ユーザの編集画面へ遷移するような設計にするのでしょうか？Usersコントローラから`edit`
    
    アクションを用意して`/users/:id/edit`のように遷移することも考えられる。
    
    しかし、これだとURLに不要な`id`が含まれてしまう。
    
    ユーザの編集画面は自分自身のプロフィールにだけ存在すれば事足りる。
    
    「はじめに」でも言及したが、他のユーザのプロフィールを編集することは想定されていないので、`profile`という`resource`(単数形リソース)でルーティング設定をし、`:id`の含まないURLを生成している。
    
- 画像ファイルをコントローラで受けとるように忘れずストロングパラメータに追記する。

# ****ビューファイルの作成****

### ****プロフィール詳細画面****

```
#profile/show.html.erb

<% content_for(:title, t('.title')) %>
<div class="container pt-3">
  <div class="row">
    <div class="col-md-10 offset-md-1">
      <h1 class="float-left mb-5">プロフィール</h1>
      <%= link_to '編集', edit_profile_path, class: 'btn btn-success float-right' %>
      <table class="table">
        <tr>
          <th scope="row">メールアドレス</th>
          <td><%= current_user.email %></td>
        </tr>
        <tr>
          <th scope="row">氏名</th>
          <td><%= current_user.decorate.full_name %></td>
        </tr>
        <tr>
          <th scope="row">アバター</th>
          <td><%= image_tag current_user.avatar_url, class: 'rounded-circle', size: '50x50' %></td>
        </tr>
      </table>
    </div>
  </div>
</div>
```

```
#profile/edit.html.erb

<% content_for(:title, t('.title')) %>
<div class="container">
    <div class="row">
        <div class="col-md-10 offset-md-1">
            <h1><%= t('.title') %></h1>
            <%= form_with model: @user, url: profile_path, local: true do |f| %>
                <%= render 'shared/error_messages', object: f.object %>
                <%= f.label :email %>
                <%= f.email_field :email, class: 'form-control mb-3' %>

                <%= f.label :last_name %>
                <%= f.text_field :last_name, class: 'form-control mb-3' %>

                <%= f.label :first_name %>
                <%= f.text_field :first_name, class: 'form-control mb-3' %>

                <%= f.label :avatar %>
                <%= f.file_field :avatar, class: 'form-control mb-3', accept: 'image/*', onchange: 'previewFileWithId(preview)' %>
                <%= f.hidden_field :avatar_cache %>

                <div class="mt-3 mb-3">
                    <%= image_tag @user.avatar.url, class: 'rounded-circle mr15', size: '100x100', id: 'preview' %>
                </div>
                <%= f.submit class: 'btn btn-primary' %>
            <% end %>
        </div>
    </div>
</div>
```
