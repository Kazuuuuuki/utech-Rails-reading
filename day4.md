# Day4

## 今日の内容

1. active_model/conversion.rb
2. active_model/dirty.rb
3. active_model / callbacks.rb

## 1. active_model/conversion.rb

```RUBY
module ActiveModel
  # == Active \Model \Conversion
  #
  # Handles default conversions: to_model, to_key, to_param, and to_partial_path.
  #
  # Let's take for example this non-persisted object.
  # dbに依存していないmodelについてもそのインスタンスにidをつけることは重要で、そのためのものをまとめたような感じだと思う。
　# 具体的には例を読んでいけばわかる。
  #
  #   class ContactMessage
  #     include ActiveModel::Conversion
  #
  #     # ContactMessage are never persisted in the DB
  #     def persisted?
  #       false
  #     end
  #   end
  #
  #   cm = ContactMessage.new
  #   cm.to_model == cm  # => true
  #   cm.to_key          # => nil
  #   cm.to_param        # => nil
  #   cm.to_partial_path # => "contact_messages/contact_message"
  module Conversion
    extend ActiveSupport::Concern

    # If your object is already designed to implement all of the \Active \Model
    # you can use the default <tt>:to_model</tt> implementation, which simply
    # returns +self+.
    #
    #   class Person
    #     include ActiveModel::Conversion
    #   end
    #
    #   person = Person.new
    #   person.to_model == person # => true
    #
    # If your model does not act like an \Active \Model object, then you should
    # define <tt>:to_model</tt> yourself returning a proxy object that wraps
    # your object with \Active \Model compliant methods.
    
    # 単にレシーバを返しているだけのよう。
　　def to_model
      self
    end

    # Returns an Array of all key attributes if any of the attributes is set, whether or not
    # the object is persisted. Returns +nil+ if there are no key attributes.
    #
    #   class Person
    #     include ActiveModel::Conversion
    #     attr_accessor :id
    #   end
    #
    #   person = Person.create(id: 1)
    #   person.to_key # => [1]
    # 見ての通り
    def to_key
      key = respond_to?(:id) && id
      key ? [key] : nil
    end

    # Returns a +string+ representing the object's key suitable for use in URLs,
    # or +nil+ if <tt>persisted?</tt> is +false+.
    #
    #   class Person
    #     include ActiveModel::Conversion
    #     attr_accessor :id
    #     def persisted?
    #       true
    #     end
    #   end
    #
    #   person = Person.create(id: 1)
    #   person.to_param # => "1"
　  # 見ての通り
    def to_param
      (persisted? && key = to_key) ? key.join('-') : nil
    end

    # Returns a +string+ identifying the path associated with the object.
    # ActionPack uses this to find a suitable partial to represent the object.
    #
    #   class Person
    #     include ActiveModel::Conversion
    #   end
    #
    #   person = Person.new
    #   person.to_partial_path # => "people/person"
    # ここで面白いのが、インスタンスからクラスのメソッドにアクセスする方法。self.classでもとのclassをレシーバに出来る。
    def to_partial_path
      self.class._to_partial_path
    end

    module ClassMethods #:nodoc:
      # Provide a class level cache for #to_partial_path. This is an
      # internal method and should not be accessed directly.
　　　# beginは例外処理で、
      # underscoreの例 underscore('ActiveModel')         # => "active_model"
      #                underscore('ActiveModel::Errors') # => "active_model/errors"

      # demodulizeの例 demodulize('ActiveRecord::CoreExtensions::String::Inflections') # => "Inflections"
      #	demodulize('Inflections')                                       # => "Inflections"
      #	demodulize('::Inflections')                                     # => "Inflections"
      #	demodulize('')                                                  # => ""
      
      # tableizeの例　tableize('RawScaledScorer') # => "raw_scaled_scorers"
      #               tableize('ham_and_egg')     # => "ham_and_eggs"
      #               tableize('fancyCategory')   # => "fancy_categories"    
  
      #  moduleやクラスはクラスメソッドnameを持つ http://ref.xaio.jp/ruby/classes/module/name
  
      def _to_partial_path #:nodoc:
        @_to_partial_path ||= begin
          element = ActiveSupport::Inflector.underscore(ActiveSupport::Inflector.demodulize(name))
          collection = ActiveSupport::Inflector.tableize(name)
          "#{collection}/#{element}".freeze
        end
      end
    end
  end
end


```

