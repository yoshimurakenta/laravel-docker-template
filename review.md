# Laravel Lesson レビュー①

## Todo一覧機能

### Todoモデルのallメソッドで実行しているSQLは何か
SELECT * FROM todos;が実行されている。
 todosテーブルの全カラムのレコードを取得するSQL

### Todoモデルのallメソッドの返り値は何か
配列に似た　Illuminate\Database\Eloquent\Collectionクラスのインスタンスでtodosテーブルのレコードを全件返す。


### 配列の代わりにCollectionクラスを使用するメリットは
データの操作を効率よく行えるようになるため
具体的に
1.簡潔なデータ操作が可能になる。
　collectionクラスにはデータ操作を行うためのメソッドが用意されている。
・filter:コレクション内の要素を条件で絞り込む
・map:各要素を加工して新しいコレクションを作成
・reduce:各要素をまとめて単一の値を生成
・pluck:特定のフィールドのみを取得
・sortBy:特定のキーでソート
2.メソッドチェーンができ、直感的操作とコードの可読性が上がる
メソッドが常にCollectionインスタンスを返すため、メソッドチェーンを可能にしている。

### view関数の第1・第2引数の指定と何をしているか
view関数はLaravelでビューをレンダリング（HTMLなどを生成すること）するために使用される。この関数を使用することでBladeテンプレートにデータを渡し、それをHTMLとして返すことができる。

view関数を使うことでControllerからbladeファイルへ値を渡すことができる。
view関数は画面に表示したいbladeファイルを第１引数に、第２引数に渡したいデータを連想配列の形で渡すことができる。
第１引数
・型：string
・意味:表示するビューファイルのパスを指定。
（例）
```php
return view('todo.index'); // resources/views/todo/index.blade.php をレンダリング
```

第２引数
・型:array
・意味:ビューに渡すデータをキーと値の連想配列として指定する
　キーがビュー内で利用する変数名で値が変数に代入されるデータ
view関数の第２引数は[blade内での変数名 =>　代入したい値]
```php
return view('todo.index', ['todos' => $todos]);
```
todosという変数名でコントローラの$todos変数をビュー内で利用可能になる。
ビュー内では次のように使用

@foreach ($todos as $todo)
  <div class="d-flex align-items-center p-2">
    <span class="col-9">{{ $todo->content }}</span>
  </div>
@endforeach


### index.blade.phpの$todos・$todoに代入されているものは何か
$todosにはデータベースのtodosテーブルから取得した全レコードのコレクション（Todo:all()の結果が代入されている）
$todoは$todosの中の１つのレコード（Todoモデルのインスタンス）が代入されている。

[$todos as $todo]で$todosの中に含まれる各レコードが１つずつ$todoに代入される。

## Todo作成機能

### Requestクラスのallメソッドは何をしているか
HTTPリクエストで送信された全ての入力されたデータを取得するためのメソッド。
フォームデータや、
クエリパラメータなどリクエストで送られた全ての入力値を連想配列として返す。
注意点としては
不要なデータも含まれる可能性がある。
allメソッドはCSRFトークンやその他隠しフィールドも含めて返すため必要なでーたを取り出すにはonlyやexceptメソッドを使う方がいい場合もある。
$data = $request->only(['content']); // 必要なフィールドのみ取得
$data = $request->except(['_token']); // 特定のフィールドを除外して取得

### fillメソッドは何をしているか
連想配列で取得した値をTodoインスタンスの各プロパティに一括で代入する。
->fill()は$todo->{連想配列のkey} = {連想配列のvalue}を配列の全ての要素に対して行う。
```php
  $todo->fill($inputs);
```
  inputsの中身は$request->all()で取得したフォームやリクエストデータが配列として格納されている。
例えば$inputsに以下の配列が含まれていると
```php
[
    'content' => 'Learn Laravel',
    'status' => 'pending'
]
```
fillメソッドは$inputsのキーを元に対応するモデルの属性に値を割り当てる。
```php
$todo->content = 'Learn Laravel';
$todo->status = 'pending';
```
となる。

### $fillableは何のために設定しているか
 $fillableはfillメソッドによってModelに代入可能なプロパティを記述する。
 一括代入は便利ではあるが、脆弱性があり対策をせずに使用することで悪意のあるユーザーに攻撃されてしまう可能性があるため$fillableを設定して許可されていないものを無視する。
こうすることで
1.セキュリティの向上:
 許可された属性にのみ値を割り当てることで、予期せぬデータの挿入を防ぐ。
2.コードの可読性と保守性:
 許可されている属性がモデル内で明確になる。
3.効率的なデータ操作:
 短いコードで属性の一括設定が可能。

 例として
 ```php
 // 悪意のあるリクエストデータ例
$request->all() = [
    'content' => 'Learn Laravel',
    'status' => 'pending',
    'is_admin' => true // 悪意のあるデータ
];

// fillメソッドが直接割り当て可能な場合
$todo->fill($request->all());
```
このように意図しないものが設定されてしまう可能性がある。

```php
protected $fillable = ['content', 'status'];
```
このように設定をすることで
```php
// 許可された属性のみ設定される
$todo->fill([
    'content' => 'Learn Laravel',
    'status' => 'pending',
    'user_id' => 1 // $fillableに含まれないため無視される
]);

$todo->save();
```
許可をされていないuser_idは無視をされる

### saveメソッドで実行しているSQLは何か
新規作成をするためのINSERT文が実行をされている。

仮に$inputsに以下のデータが含まれていると
```php
$inputs = [
    'content'  => 'Learn Laravel',
    'title'    => 'Laravel Basics',
    'location' => 'Online',
    'deadline' => '2024-12-29',
    'category' => 'Programming'
];
```
このデータを元にして実行されるSQLクエリは次のようになる。
```sql
INSERT INTO todos (content, title, location, deadline, category, created_at, updated_at)
VALUES ('Learn Laravel', 'Laravel Basics', 'Online', '2024-12-01', 'Programming', '2024-11-11 10:30:00', '2024-11-11 10:30:00');
```
これによってデータベースに保存される。

### redirect()->route()は何をしているか
指定した名前付きルートにリダイレクトをしている。
redirect()はユーザーを別のURLにリダイレクトするためのメソッドでリダイレクト先にはURLや名前付きルートを使う。
route()メソッドは名前付きルートに基づいてリダイレクト先のURLを生成する。
これによりURLを直接記述することなくリダイレクト先を動的に決定できる。
```php
return redirect()->route('todo.index');
```
このコードはtodo.indexという名前付きルートにリダイレクトする。
名前付きルートを使用することでURLが変更された場合でも名前付きルートを使ってリダイレクトすることでコードの保守性が高まる。

メソッドチェーンで切る理由は
redirect() は、Illuminate\Routing\Redirector クラスのインスタンスを返します。
そメソッドであるroute()メソッドだから

## その他

### テーブル構成をマイグレーションファイルで管理するメリット
マイグレーションファイルでデータベース構成を管理することで、効率性・保守性・統一性が大幅に向上する。
具体的に
1.バージョンの管理が可能
マイグレーションファイルはGitなどのバージョン管理システムと統合して運用できる。
またデータベースの変更履歴を記録できるため誰がいつどのような変更を行なったかが明確になる。
2.環境ごとに同じ構成を再現できる。

### マイグレーションファイルのup()、down()は何のコマンドを実行した時に呼び出されるのか
up()メソッドはマイグレーションを適用する際、[php artisan migrate]、[php artisan migrate:fresh]を実行し呼び出される。
[php artisan migrate]は全てのマイグレーションファイルのup()を順番に実行する。
[php artisan migrate:fresh]は一度全てのテーブルを削除し、全てのup()を再実行する。（この際down()は呼び出されない）

down()メソッドはマイグレーションを取り消す際、[php artisan migrate:rollback]、[php artisan migrate:reset]を実行し呼び出される。
down()メソッドはup()メソッドで行なった変更を元に戻す。（テーブル削除やカラム削除など）
[php artisan migrate:rollback]は最後に実行したマイグレーションのdown()を実行する（１ステップ分）
[php artisan migrate:reset]は全てのマイグレーションのdown()を順番に実行する。

### Seederクラスの役割は何か
データベースにテストデータや初期データを挿入すること。
具体的に
1.初期データの登録
アプリケーションで必要な初期データ（ユーザーの役割、設定データ）をデータベースに挿入する。テーブル作成後に手動でデータを入力する手間を省ける。
2.テストデータの作成
開発中に機能やレイアウトを確認する為サンプルデータ簡単に生成できる。テスト用のレコードを何度でも再生成可能。
3.一貫性のあるデータ生成
コードを通じてデータを管理するため異なる環境（開発、ステージング、本番）で同じ初期データを用意できる。
4.データの管理と再利用
シーダーファイルを使ってデータの内容を記述するためデータ修正や再生成が簡単に行える。

### route関数の引数・返り値・使用するメリット
route関数の引数：ルートの名前を指定する。この名前は routes/web.php に定義したルートの name メソッドで指定されたもの。
route関数の返り値はredairectorインスタンス
使用するメリット
・URLを直接記述をすることないためアプリケーション内でURLが変更された場合に一箇所の変更で済む。
・リダイレクト先にURLを指定する代わりに使うことができURLが変更されてもリダイレクトの記述を変更する必要がない。

### @extends・@section・@yieldの関係性とbladeを分割するメリット
それぞれの意味としては
@extends
@extends('Bladeファイルのパス')を使用することで他のBladeファイルを継承することができる。
・@section
親Blade内の @yield に挿入するコンテンツ（セクション）を定義する。
@section内に記述したコンテンツは親Bladeの指定された場所（@yield）に挿入される。
・@yield
親Blade内で、子Bladeから挿入されるコンテンツが入る場所を指定する。
@yieldで指定したセクション名に対応する内容が子Bladeから挿入される。
まず、@extendsで継承する親Bladeを指定し、
そして、継承先の子Bladeの@section('content') ~ @endsectionで囲われた部分を、親Bladeの@yield('content')の部分に挿入している
引数に同じ文字列'content'を指定することで、@section()と@yield()を紐づけている。

Bladeを分割することで
・再利用性の向上
共通部分（ヘッダー、フッター、ナビゲーションバーなど）を１つにまとめて複数ページで使いまわせる。
。可読性の向上
コードが整理されることで見やすくなる。
・保守性の向上
分割することで修正箇所が特定しやすくなる。
・開発チームでの分担が容易になる。
Bladeファイルが独立しているため複数の開発者が同時に作業しやすくなる。
などのメリットがある。

### @csrfは何のための記述か
CSRFを防ぐためのLaravelで提供されているBladeテンプレートのディレクティブ。
フォームにCSRFトークンを埋め込んでいる。

Blade
<form method="POST" action="">
    @csrf
    <input type="text" name="name">
    <button type="submit">送信</button>
</form>


html
<form method="POST" action="">
    <input type="hidden" name="_token" value="ランダムなCSRFトークン">
    <input type="text" name="name">
    <button type="submit">送信</button>
</form>
このように変換をされ、CSRFトークンをhiddenタグでPOSTメソッドで送信をする。
Laravelはリクエストを受け取った際に、自動的にCSRFトークンを検証します。この処理はミドルウェア VerifyCsrfToken によって行われます。


### {{ }}とは何の省略系か
PHPのecho文の省略系で
{{ $Hello }}は<?php echo $Hello; ?>と同じ意味になる。

BladeテンプレートでHTMLにPHPコードを埋め込む際に使用するもので{{}}内に記述した内容はサーバー側でPHPとしてその結果を出力する。
また、HTMLエスケープが自動で適用されるため、XSS対策がされている。
エスケープを無効にしたい場合は{!! $Hello !!}と記述をすることでエスケープを無効にできる。