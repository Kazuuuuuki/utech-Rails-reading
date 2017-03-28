# Day5

## 今回の内容

+ 1.active_model / gem_version.rb
+ 2.active_model / lint.rb
+ 3.active_model / model.rb
+ 9.active_model / naming.rb
    - 4.active_support / core_ext / hash / except.rb
    - 6.active_support / core_ext / module / introspection.rb
        + 5.active_support / inflector
    - 7.active_support / core_ext / module / remove_method.rb
    - 8.active_support / core_ext / module / delegation.rb

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

## 5. active_support / core_ext / module / introspection.rb

+ Module の 上の階層を調べるための .parent_name, .parent, .parents, .local_constants メソッドを追加する

```Ruby
require 'active_support/inflector'

class Module
  # Returns the name of the module containing this one.
  #
  #   M::N.parent_name # => "M"
  ## 親Moduleの名前を返す
  def parent_name
    if defined? @parent_name
      @parent_name
    else
      ## name は "Animals::Cat.name" というクラスに対し "Animals::Cat" という文字列が返る Ruby のもとからあるメソッド *1
      ## この正規表現は，:: が2つ続き，その後 : 以外の文字が1つ以上続き終端に達する ときにその文字列を集合に含む．
      ## つまり，親がいない場合は (ex, "Cat") 集合に含まず，nilが返る
      ## $`は，正規表現評価後，マッチしたテキストの手前の文字列を返す
      ## "Creature::Animals::Cat" ならば，"::Cat"の部分がマッチし，その手前の"Creature::Animals" が$`に代入される
      @parent_name = name =~ /::[^:]+\Z/ ? $`.freeze : nil
    end
  end

  # Returns the module which contains this one according to its name.
  #
  #   module M
  #     module N
  #     end
  #   end
  #   X = M::N
  #
  #   M::N.parent # => M
  #   X.parent    # => M
  #
  # The parent of top-level and anonymous modules is Object.
  #
  #   M.parent          # => Object
  #   Module.new.parent # => Object
  ## 実際の親Moduleを返す 
  def parent
    parent_name ? ActiveSupport::Inflector.constantize(parent_name) : Object
  end

  # Returns all the parents of this module according to its name, ordered from
  # nested outwards. The receiver is not contained within the result.
  #
  #   module M
  #     module N
  #     end
  #   end
  #   X = M::N
  #
  #   M.parents    # => [Object]
  #   M::N.parents # => [M, Object]
  #   X.parents    # => [M, Object]
  ## 親モジュールすべてを配列にして返す
  def parents
    parents = []
    if parent_name
      parts = parent_name.split('::')
      until parts.empty?
        parents << ActiveSupport::Inflector.constantize(parts * '::')
        parts.pop
      end
    end
    parents << Object unless parents.include? Object
    parents
  end

  ## これなんだろうね
  ## 消されるらしいしどうでもいいんじゃない？
  def local_constants #:nodoc:
    ActiveSupport::Deprecation.warn(<<-MSG.squish)
      Module#local_constants is deprecated and will be removed in Rails 5.1.
      Use Module#constants(false) instead.
    MSG
    constants(false)
  end
end

```
*1 http://ref.xaio.jp/ruby/classes/module/name
