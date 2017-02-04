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
Active Modelの大元。
ここで大量にモジュールをauto_loadしている。
auto_loadとは?
ActiveSupportのRuby拡張
auto_loadは遅延ファイル読み込み。(必要になったときにファイルを読み込む)

###activesupport/lib/active_support/dependencies/autoload.rb
module Autoloadがextendされたときにクラスインスタンス変数を初期化している。
eager_autoloadでまとめて必要なものを先にrequireしている。

























