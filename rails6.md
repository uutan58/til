# デコレーターとは

- Decoratorとは、ソフトウェアデザインパターンの一種。

> デザインパターンとは
> 
> 
> Railsにおけるデザインパターンとは、**モデルやコントローラ、ビューに頻出する実装パターンを、オブジェクト設計の原則にもとづいて抽象化したパターン**のことです。 デザインパターンにしたがうことで、設計上の問題を防ぐことができます。
> 
> 参照：[Railsのデザインパターンまとめ](https://applis.io/posts/rails-design-patterns#rails%E3%81%AB%E3%81%8A%E3%81%91%E3%82%8B%E3%83%87%E3%82%B6%E3%82%A4%E3%83%B3%E3%83%91%E3%82%BF%E3%83%BC%E3%83%B3%E3%81%A8%E3%81%AF)
> 

# デコレーターを導入する流れ

1. 「draper」をインストールする。
2. デコレーターを作成する。
3. デコレーターにメソッドを記述する。
4. ビューファイルでデコレーターのメソッドを呼び出す。

# draperをインストールする

```
#Gemfileに記載する

gem 'draper'
```

```
#ターミナルでコマンドを実行

$ bundle install
```

- Gemをインストールしたあとは、生成された全てのデコレーターが継承するA`pplicationDecorator`を作成するコマンドを入力する。

```
#ターミナルでコマンドを実行

$ rails generate decorator User
```

- これにより、「add/decorators/user_decorator.rb」が生成される。

# デコレーターにメソッドを記述する

- 例題として、フルネームを呼び出すメソッドを作成していく。
    
    『last_name』と『first_name』は、事前にUserモデルのカラムに追加してある。
    

```
#app/decorators/user_decorator.rb

class UserDecorator < Draper::Decorator
	delegate_all  # UserモデルのメソッドをUserDecoratorクラスでも使用できるようにするための記述
	
	def full_name
		#これ
		"#{object.last_name} #{object.first_name}"
	end
end
```

- 『delegate_all』は、UserモデルのメソッドをUserDecoratorクラスでも呼び出せるようにするための記述。
    
    この記述をすることで、『user.name』など、UserモデルのインスタンスメソッドがUserDecoratorクラスでも使用可能になる。
    
    そのため、上記ではfull_nameメソッドで、『last_name』や『first_name』といったメソッドが使えている。
    

# ビューファイルでデコレーターのメソッドを呼び出す

- 下記のように記述し、現在ログインしているユーザー（current_user）をデコレーターメソッドを呼び出してフルネームで表示する。

```
- #メソッドを使いたいViewファイルに記述する

<%= current_user.decorate.full_name %>
```
