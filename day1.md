#Day1

##はじめに　
***この勉強会は自主ゼミであり、参加する皆さんの自主性を期待します。***  
本勉強会はRailsのソースコードリーディングを進めていく自主ゼミです。　　
扱う内容はActive ModelとActive Supportです(全部は多分無理です)。　　

***本ゼミの目的はRailsの全てを理解するのではなく、優れたオープンソースのコードを読むことでプログラミング能力を向上しようということです***

##自己紹介
氏名　渡邉知樹
大学 東京大学理科I類1年生
Engineer(機会学習、Web、DBなどなど)
趣味は数学

facebook
<https://www.facebook.com/profile.php?id=100011591890838&lst=100011591890838%3A100011591890838%3A1485578448>

##団体説明
***uTech(ゆーてっく)***

東京大学を活動拠点とする、IT系サークル。

この春から活動を始めました。

扱う領域は
* Web(インターネット)
* 機械学習
* スマホアプリ
* 競技プログラミング
* 情報
* 数学　　

などなど幅広く活動していきます。

##参考書籍
メタプログラミングRuby</br>
ぐうの音もでない名著です。
##アイスブレイク  
今後一緒に話し合いながら、ソースコードリーディングを進めていくので、仲良くなりましょう。　　

##ソースコードをクローン
https://github.com/rails/rails

##Active Model
Active Recordが扱うのにふさわしくないModelを生成するためのモジュールの集まり。
そもそもModelとはビジネスロジックを担当している層。(詳しくはMVCを確認してみてください。)
基本的にはRailsはMVCアーキテクチャを採用している。</br>
しかし、MVCのデメリットが目立ってきた。</br>
Railsは基本的にModelとしてActive Recordを採用していますが、Model自体が肥大化してしまう（FatModel）ことを防ぐために、テーブルと結びついているModelをActive Record、それ以外のModelをActive Modelとするアーキテクチャを現在採用しています。</br>

##Active Support  
これはRailsのためというより、Rubyの拡張としての意味合いが強いです。
Active SupportはActive Modelを読むために必要なときに必要な箇所のみ読むことにします。

##内容
###activemodel/lib/active_model.rb
Active Modelの大元。</br>
ここで大量にモジュールをauto_loadしている。</br>
auto_loadとは?</br>
ActiveSupportのRuby拡張</br>
auto_loadは遅延ファイル読み込み。(必要になったときにファイルを読み込む)</br>

###activesupport/lib/active_support/dependencies/autoload.rb
module Autoloadがextendされたときにクラスインスタンス変数を初期化している。</br>
eager_autoloadでまとめて必要なものを先にrequireしている。</br>

***autoloadされているModuleを頭から見ていきます。***

###activemodel/lib/active_model/attribute_assignment.rb
assign_attributes関数のrespond_to?で例外処理しているのはダックタイピングの考え方だと思われる。</br>

###activemodel/lib/active_model/forbidden_attributes_protection.rb
例外処理してるだけ。

###再びactivemodel/lib/active_model/attribute_assignment.rb
sendやevalは賛否両論。しかし、適切にprivateやprotectして機能を制限していけばうまく使える。
ゆえにただのsendではなくpublic_sendを使っている。

###acutivesupport/lib/active_support/concern.rb
すごい簡単にいえば、めんどくさいし難しいextend周りをうまいことやってくれるmodule
具体的に何をやってくれるかは英語のコメントでわかるので割愛
base.instance_variable_setはクラスインスタンス変数を設定している。
###append_features
append_featuresメソッドは、モジュールが他のクラスやモジュールにインクルードされる前に呼び出される</br>
この具体例を見ながら読むと分かりやすい。
http://ref.xaio.jp/ruby/classes/module/append_features</br>
要は、includeとextendが連鎖していくと期待どうりの継承関係にならなくなる。</br>
そこで、include呼び出しを行っているbaseがconcernかどうかで場合分けしている。</br>
concernだった場合、includeを行わず、配列に追加。</br></br>
concernではなかった場合、今まで配列に溜まっていたものも含めてまとめてinclude。</br>
最後までconcernかもしれなくない?</br>
一般的にはありえるが、Railsはauto_loadやincludeをするためだけのようなmoduleがあるため、そのようなことはおこらない。</br>
###activemodel/lib/active_model/attribute_methods.rb
具体例</br>
http://qiita.com/pekepek/items/8eead2021024f70f08f8</br>





