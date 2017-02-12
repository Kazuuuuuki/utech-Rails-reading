#Day2

##はじめに　
***この勉強会は自主ゼミであり、参加する皆さんの自主性を期待します。***  
本勉強会はRailsのソースコードリーディングを進めていく自主ゼミです。　　
扱う内容はActive ModelとActive Supportです(全部は多分無理です)。　　

***本ゼミの目的はRailsの全てを理解するのではなく、優れたオープンソースのコードを読むことでプログラミング能力を向上しようということです***

##自己紹介

##団体説明
***uTech(ゆーてっく)***

##参考書籍
メタプログラミングRuby</br>
ぐうの音もでない名著です。

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

### activemodel/lib/active_model/attribute_methods.rb
- [x] 全体像
- [x] 定義されたメソッド
- [ ] 処理の全体像
- [ ] ユースケースごとの処理内容
- [ ] class_attributeについて
- [ ] 使用するライブラリについて
- [ ] 依存関係

#### 全体像
define_attribute_methodsで定義したattributeの接頭、接尾に定義した文字列を追加したメソッドを生成する。
これによって複数のattributeで共通の処理を行うことができる。

[サンプル](http://qiita.com/pekepek/items/8eead2021024f70f08f8)

```ruby
class Person
  include ActiveModel::AttributeMethods

  attr_accessor :name, :age, :address
  attribute_method_prefix 'clear_'
  define_attribute_methods :name, :age, :address

  def initialize(name:, age:, address:)
    @name = name
    @age = age
    @address = address
  end

  private

  def clear_attribute(attr)
    send("#{attr}=", nil)
  end
end

person = Person.new
person.name = 'Bob'
person.name         #=> "Bob"
person.clear_name
person.name         #=> nil
person.age = 20
person.clear_age
person.age          #=> nil
```

#### 定義されたメソッド
AttributeMethodsで定義されたメソッドは

- attribute_method_prefix(*prefixes) : 接頭にprefixesを付加したメソッドを定義
- attribute_method_suffix(*suffixes) : 末尾にsuffixesを付加したメソッドを定義
- attribute_method_affix(*affixes) : 接頭、末尾の双方にaffixesを追加したメソッドを定義
- alias_attribute(new_name, old_name) : 既存のattributeに対して別名でのアクセスを定義
- attribute_alias?(new_name) : alias_attributeによって定義されたattributeか?
- attribute_alias(name) : オリジナルのattributeを返す
- define_attribute_methods(*attr_names) : prefix, suffixで使用するattributeを宣言する(複数)
- define_attribute_method(attr_name) : prefix, suffixで使用するattributeを宣言する
- undefine_attribute_methods : define_attribute_methodでの定義を破棄する
- generated_attribute_methods : 
- protected
- instance_method_already_implemented?(method_name) : すでに定義されたメソッドか調べる(レシーバー付きで呼び出されてないのでprivateでいいのでは？)
- private
- attribute_method_matchers_cache : matched_attribute_methodメソッドの処理軽減に利用される
- attribute_method_matchers_matching(method_name) : method_nameに対してマッチするメソッド配列を返す
- define_proxy_call(include_private, mod, name, send, *extra) : sendに対してextraを送る処理をmod内にnameメソッドとして定義する

AttributeMethodMatcher
prefix, suffixをオプションとして生成できる.

- match(method_name) : method_nameが定義されていれば定義されたprefix, suffixから生成された構造体を返す
- method_name(attr_name) : ？
- plain? : prefix, suffixが存在しているか?

#### 処理の全体像
対象のクラス(ClassA)内でAttributeMethodsによって追加される特異メソッドを使いメソッドを定義
ClassAのオブジェクトから定義されたメソッドへのアクセスはmethod_missingパターンを利用して行われる。

[メソッド定義部分]
AttributeMethodsを使用してこのように定義されたクラスについて見ていく。

```ruby
class Person
  include ActiveModel::AttributeMethods

  attr_accessor :name, :age, :address
  attribute_method_prefix 'clear_'
  define_attribute_methods :name, :age, :address

  def initialize(name:, age:, address:)
    @name = name
    @age = age
    @address = address
  end

  private

  def clear_attribute(attr)
    send("#{attr}=", nil)
  end
end
```

`include ActiveModel::AttributeMethods`が行われると
AttributeMethodsのConcernによって`included do~end`と`module ClassMethods`が特異クラスに追加される。
このときクラス変数（特異クラス）として:attribute_aliases, :attribute_method_matchersを定義している。

```ruby
  included do
    class_attribute :attribute_aliases, :attribute_method_matchers, instance_writer: false
    self.attribute_aliases = {}
    self.attribute_method_matchers = [ClassMethods::AttributeMethodMatcher.new]
  end
```

定義された`attribute_aliases`は既存のattributeに別名でのアクセスを可能にする為に使われる。
具体的には新しく定義されたattributeをkeyとして既存のattributeを返すハッシュ値

```ruby
  def alias_attribute(new_name, old_name)
    self.attribute_aliases = attribute_aliases.merge(new_name.to_s => old_name.to_s)
    attribute_method_matchers.each do |matcher|
      matcher_new = matcher.method_name(new_name).to_s
      matcher_old = matcher.method_name(old_name).to_s
      define_proxy_call false, self, matcher_new, matcher_old
    end
  end
```

attribute_method_matchersはAttributeMethodMatcherクラスの配列を持つオブジェクト
AttributeMethodMatcherはprefixとsuffixをオプションの引数にしてAttributeMethodsで生成されるメソッド名を管理し
method_nameを引数にそのメソッドが定義されているかどうか判別するメソッドを提供する。

```ruby
  class AttributeMethodMatcher #:nodoc:
    attr_reader :prefix, :suffix, :method_missing_target

    AttributeMethodMatch = Struct.new(:target, :attr_name, :method_name)

    def initialize(options = {})
      @prefix, @suffix = options.fetch(:prefix, ""), options.fetch(:suffix, "")
      @regex = /^(?:#{Regexp.escape(@prefix)})(.*)(?:#{Regexp.escape(@suffix)})$/
      @method_missing_target = "#{@prefix}attribute#{@suffix}"
      @method_name = "#{prefix}%s#{suffix}"
    end

    def match(method_name)
      if @regex =~ method_name
        AttributeMethodMatch.new(method_missing_target, $1, method_name)
      end
    end

    def method_name(attr_name)
      # 何してる？
      @method_name % attr_name
    end

    def plain?
      prefix.empty? && suffix.empty?
    end
  end
```

[メソッド実行部分]
対象クラスでAttributeMethodsのメソッドが利用された場合の処理について

```rb
class Person
  include ActiveModel::AttributeMethods

  attr_accessor :name, :age, :address
  attribute_method_prefix 'clear_'
  define_attribute_methods :name, :age, :address

  def initialize(name:, age:, address:)
    @name = name
    @age = age
    @address = address
  end

  private

  def clear_attribute(attr)
    send("#{attr}=", nil)
  end
end

person = Person.new
person.name = 'Bob'
person.name         #=> "Bob"
person.clear_name
person.name         #=> nil
person.age = 20
person.clear_age
person.age          #=> nil
```

`attribute_method_prefix`が呼ばれると渡されたprefixesを引数にして`AttributeMethodMatcher`が生成され
`self.attribute_method_matchers`に格納される。
`undefine_attribute_methods`メソッドはこれまでに生成されたattribute_methodを消している。
つまり`define_attribute_methods`するのは`attribute_method_prefix`の後である必要がある。

```rb
  def attribute_method_prefix(*prefixes)
    self.attribute_method_matchers += prefixes.map! { |prefix| AttributeMethodMatcher.new prefix: prefix }
    undefine_attribute_methods
  end
```

`attribute_method_suffix`、`attribute_method_affix`も`attribute_method_prefix`と同様

```rb
  def attribute_method_suffix(*suffixes)
    self.attribute_method_matchers += suffixes.map! { |suffix| AttributeMethodMatcher.new suffix: suffix }
    undefine_attribute_methods
  end

  def attribute_method_affix(*affixes)
    self.attribute_method_matchers += affixes.map! { |affix| AttributeMethodMatcher.new prefix: affix[:prefix], suffix: affix[:suffix] }
    undefine_attribute_methods
  end
```

`define_attribute_methods`はprefix, suffixまたはaffixで定義した命名をどのattributeに適用するかを決定するメソッド。
ここで選択されたattributeに共通メソッドが割り当てられる。
具体的な処理としてはattr_nameを引数にattribute_method_matchers内に存在するAttributeMethodMatcherをループさせメソッド名を取得。
そのメソッド名ですでにメソッドが定義されれてないか調べる。
定義されていなければ`define_method_`で始まるmatcher.method_missing_targetに一致するメソッドがないか確認
存在すればそのメソッド呼び出し、存在しなければ`define_proxy_call`メソッドを実行。

```rb
def define_attribute_method(attr_name)
  attribute_method_matchers.each do |matcher|
    method_name = matcher.method_name(attr_name)

    unless instance_method_already_implemented?(method_name)
      generate_method = "define_method_#{matcher.method_missing_target}"

      if respond_to?(generate_method, true)
        send(generate_method, attr_name)
      else
        define_proxy_call true, generated_attribute_methods, method_name, matcher.method_missing_target, attr_name.to_s
      end
    end
  end
  attribute_method_matchers_cache.clear
end
```

`define_proxy_call`について
このメソッドではnameメソッドが呼ばれた時にsendメソッドに書かれた処理を行うようにメソッドを定義している。

1. name名のメソッドを定義する文字列を生成
2. sendメソッドの呼び出しを行う文字列を生成
3. mod内に1のメソッドを定義し、その処理内容として2を設定

ちなみにdefine_attribute_methodの場合modをgenerated_attribute_methodsとしているがgenerated_attribute_methodsはなんなのか

```rb
  # Define a method `name` in `mod` that dispatches to `send`
  # using the given `extra` args. This falls back on `define_method`
  # and `send` if the given names cannot be compiled.
  def define_proxy_call(include_private, mod, name, send, *extra) #:nodoc:
    defn = if NAME_COMPILABLE_REGEXP.match?(name)
      "def #{name}(*args)"
    else
      "define_method(:'#{name}') do |*args|"
    end

    # ?
    extra = (extra.map!(&:inspect) << "*args").join(", ".freeze)

    target = if CALL_COMPILABLE_REGEXP.match?(send)
      "#{"self." unless include_private}#{send}(#{extra})"
    else
      "send(:'#{send}', #{extra})"
    end

    mod.module_eval <<-RUBY, __FILE__, __LINE__ + 1
      #{defn}
        #{target}
      end
    RUBY
  end
```

#### 使用するライブラリについて
[concurrent/map](https://github.com/ruby-concurrency/concurrent-ruby/blob/master/lib/concurrent/map.rb)


#### 参考
- [Ruby の private と protected 。歴史と使い分け](http://qiita.com/tbpgr/items/6f1c0c7b77218f74c63e)
- [Rubyのattr_accessor等について](http://qiita.com/jordi/items/7baeb83788c7a8f2070d) 
- [Ruby入門 - 演算子](http://www.tohoho-web.com/ruby/operators.html)
- [これはMUST！ActiveSupport の Class#class_attribute を使おう！](http://qiita.com/cuzic/items/ffd115f1e17458020b1b)
- [ActiveModel::AttributeMethods で重複をなくす](http://qiita.com/pekepek/items/8eead2021024f70f08f8)