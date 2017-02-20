# Day3

## 今日の内容

1. active_support / descendants_tracker.rb
2. active_support / callbacks.rb
3. active_model / callbacks.rb

active_model/callbacks.rb は、active_support/callbacks.rb の拡張.
active_support/callbacks.rb では、active_support / descendants_tracker.rb をincludeしている.

## 1. activesupport / descendants_tracker.rb

ここでは
子クラスとは、あるクラス(モジュール)を直接継承しているクラス(モジュール)のことであり、
子孫クラスとは、あるクラス(モジュール)の子クラスと、それより下のクラス(モジュール)のことである。

```RUBY
module ActiveSupport
  # This module provides an internal implementation to track descendants
  # which is faster than iterating through ObjectSpace.

  ## このモジュールは子孫クラスを追跡するための内部実装を提供する.
  ## ObjectSpace(全オブジェクト)を走査するよりも速い.

  ## ObjectSpaceを走査することによってsubclassをみつけてくるのが
  ## active_support/core_ext/class/subclasses.rb

  module DescendantsTracker
    ## 直接の子クラスについての情報を保存するためのハッシュ
    ## keyはクラスであり、valueは子クラスの配列
    ## Moduleのクラス変数（モジュール変数？）はモジュールに共通の変数となる.
    ## インクルードしたクラスで共通ということ. *1
    @@direct_descendants = {}


    ## クラスメソッド
    class << self

      ## 子クラスを配列で返す
      def direct_descendants(klass)
        @@direct_descendants[klass] || []
      end

      ## 再帰的に子孫クラスを返す
      def descendants(klass)
        arr = []
        accumulate_descendants(klass, arr)
        arr
      end

      ## @@direct_descendantsをclearする.
      ## ActiveSupport::Dependenciesが定義されているとautoloadされているクラスの子クラスと、autoloadされている子クラスのみを消す.
      ## なぜかはわからない
      def clear
        if defined? ActiveSupport::Dependencies
          @@direct_descendants.each do |klass, descendants|
            if ActiveSupport::Dependencies.autoloaded?(klass)
              @@direct_descendants.delete(klass)
            else
              ## reject:配列を走査し、ブロックがtrueを返した要素を排除する.
              ## つまり、falseの要素だけが残る.
              descendants.reject! { |v| ActiveSupport::Dependencies.autoloaded?(v) }
            end
          end
        else
          ## ハッシュを空にする.
          @@direct_descendants.clear
        end
      end

      # This is the only method that is not thread safe, but is only ever called
      # during the eager loading phase.
      ## これは、スレッドセーフでない唯一のメソッドである.
      ## しかし、eager loadingの段階でしか呼ばれない.
      def store_inherited(klass, descendant)
        (@@direct_descendants[klass] ||= []) << descendant
      end

      private
      ## 子孫クラスを探すためのメソッド
      ## クラスと、積算した結果を保存する引数を取る
      def accumulate_descendants(klass, acc)
        ## 子クラスがあるかの判定と代入を同時に行っている.
        if direct_descendants = @@direct_descendants[klass]

          ## concat:配列の結合　*2
          acc.concat(direct_descendants)

          ## 子クラスについて再帰的に行う
          direct_descendants.each { |direct_descendant| accumulate_descendants(direct_descendant, acc) }
        end
      end
    end

    ##インスタンスメソッド（このmoduleをincludeしたclassのクラスメソッド）

    ## classが継承されたとき
    def inherited(base)
      ## @@direct_descendants に子孫関係を記録する
      DescendantsTracker.store_inherited(self, base)

      ## superを呼び出すの、他で定義されたinheritedを呼び出すため
      super
    end

    ## 子クラスを返す
    def direct_descendants
      DescendantsTracker.direct_descendants(self)
    end

    ## 子孫クラスを返す
    def descendants
      DescendantsTracker.descendants(self)
    end
  end
end
```

*1 Module変数について http://d.hatena.ne.jp/shunsuk/20101105/1288948776

*2 配列の結合色々 http://qiita.com/na1412/items/65f883896c85011d6509

## 2. active_support / callbacks.rb
