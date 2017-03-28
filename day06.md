# Day5

## 今回の内容

+ 1. active_model / gem_version.rb
+ 2. active_model / lint.rb
+ 3. active_model / model.rb
+ 9. active_model / naming.rb
  - 4. active_support / core_ext / hash / except.rb
  - 6. active_support / core_ext / module / introspection.rb
    + 5.active_support / inflector
  - 7. active_support / core_ext / module / remove_method.rb
  - 8. active_support / core_ext / module / delegation.rb

## 1. active_model / gem_version.rb

Gemのバージョンを返す関数があるだけ．
特に書くこともない．

## 2. active_model / lint.rb

Object が to_model という関数に対応してるかテストするためのコード群．
ときに書くこともない．

## 3. active_model / model.rb

+ これを読み込むことで，クラスを Model としてあつあうことができる．
+ newするときに Person.new(name: 'bob', age: '18') こういう書き方ができるようにする
+ persisted? はオブジェクトが永続性を持っているかという意味．要するにデーターベースに保存されているか？ということらしい．
+ その他，Validation などもここで読み込まれ，使えるようになる.
+ 読み込まれている他のファイルはまた別の機会に．(Validations,Translation)

## 4. active_support / core_ext / hash / except.rb

+ 指定したキーを除いた hash を返す .except, .except! を追加する．
+ ただ，指定されたキーをdeleteしてるだけ．
+ 今更気づいたけど， core_ext って Ruby のクラスの拡張をする(Core(Ruby) extened)物が入ってるフォルダなのね．

5. active_support / core_ext / module / introspection.rb