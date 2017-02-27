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
## 2. active_support/hash_with_indifferenct_access.rb
やってることわかりやすいけど、例のごとく配列の場合とか出てきてむずかしい　http://qiita.com/hamajyotan/items/085ec6c518fa1f692f24

```RUBY
require 'active_support/core_ext/hash/keys'
require 'active_support/core_ext/hash/reverse_merge'

module ActiveSupport
  # Implements a hash where keys <tt>:foo</tt> and <tt>"foo"</tt> are considered
  # to be the same.
  #
  #   rgb = ActiveSupport::HashWithIndifferentAccess.new
  #
  #   rgb[:black] = '#000000'
  #   rgb[:black]  # => '#000000'
  #   rgb['black'] # => '#000000'
  #
  #   rgb['white'] = '#FFFFFF'
  #   rgb[:white]  # => '#FFFFFF'
  #   rgb['white'] # => '#FFFFFF'
  #
  # Internally symbols are mapped to strings when used as keys in the entire
  # writing interface (calling <tt>[]=</tt>, <tt>merge</tt>, etc). This
  # mapping belongs to the public interface. For example, given:
 
  #   要は、シンボルで定義されたkeyは文字列で返されることが保証される
  #   hash = ActiveSupport::HashWithIndifferentAccess.new(a: 1)
  #
  # You are guaranteed that the key is returned as a string:
  #  
  #   hash.keys # => ["a"]
  #
  # Technically other types of keys are accepted:
  #
  #   hash = ActiveSupport::HashWithIndifferentAccess.new(a: 1)
  #   hash[0] = 0
  #   hash # => {"a"=>1, 0=>0}
  #
  # but this class is intended for use cases where strings or symbols are the
  # expected keys and it is convenient to understand both as the same. For
  # example the +params+ hash in Ruby on Rails.
  #
  # Note that core extensions define <tt>Hash#with_indifferent_access</tt>:
  #
  #   rgb = { black: '#000000', white: '#FFFFFF' }.with_indifferent_access
  #
  # which may be handy.
  #
  # To access this class outside of Rails, require the core extension with:
  #
  #   require "active_support/core_ext/hash/indifferent_access"
  #
  # which will, in turn, require this file.
  class HashWithIndifferentAccess < Hash
    # Returns +true+ so that <tt>Array#extract_options!</tt> finds members of
    # this class.

    def extractable_options?
      true
    end

    def with_indifferent_access
      dup
    end

    def nested_under_indifferent_access
      self
    end

    def initialize(constructor = {})
      if constructor.respond_to?(:to_hash)
        super()
        update(constructor)

        hash = constructor.to_hash
        self.default = hash.default if hash.default
        self.default_proc = hash.default_proc if hash.default_proc
      else
        super(constructor)
      end
    end

    def default(*args)
      arg_key = args.first

      if include?(key = convert_key(arg_key))
        self[key]
      else
        super
      end
    end

    def self.new_from_hash_copying_default(hash)
      ActiveSupport::Deprecation.warn(<<-MSG.squish)
        `ActiveSupport::HashWithIndifferentAccess.new_from_hash_copying_default`
        has been deprecated, and will be removed in Rails 5.1. The behavior of
        this method is now identical to the behavior of `.new`.
      MSG
      new(hash)
    end

    def self.[](*args)
      new.merge!(Hash[*args])
    end

    alias_method :regular_writer, :[]= unless method_defined?(:regular_writer)
    alias_method :regular_update, :update unless method_defined?(:regular_update)

    # Assigns a new value to the hash:
    #
    #   hash = ActiveSupport::HashWithIndifferentAccess.new
    #   hash[:key] = 'value'
    #
    # This value can be later fetched using either +:key+ or <tt>'key'</tt>.
    def []=(key, value)
      regular_writer(convert_key(key), convert_value(value, for: :assignment))
    end

    alias_method :store, :[]=

    # Updates the receiver in-place, merging in the hash passed as argument:
    #
    #   hash_1 = ActiveSupport::HashWithIndifferentAccess.new
    #   hash_1[:key] = 'value'
    #
    #   hash_2 = ActiveSupport::HashWithIndifferentAccess.new
    #   hash_2[:key] = 'New Value!'
    #
    #   hash_1.update(hash_2) # => {"key"=>"New Value!"}
    #
    # The argument can be either an
    # <tt>ActiveSupport::HashWithIndifferentAccess</tt> or a regular +Hash+.
    # In either case the merge respects the semantics of indifferent access.
    #
    # If the argument is a regular hash with keys +:key+ and +"key"+ only one
    # of the values end up in the receiver, but which one is unspecified.
    #
    # When given a block, the value for duplicated keys will be determined
    # by the result of invoking the block with the duplicated key, the value
    # in the receiver, and the value in +other_hash+. The rules for duplicated
    # keys follow the semantics of indifferent access:
    #
    #   hash_1[:key] = 10
    #   hash_2['key'] = 12
    #   hash_1.update(hash_2) { |key, old, new| old + new } # => {"key"=>22}
    def update(other_hash)
      if other_hash.is_a? HashWithIndifferentAccess
        # もともとhashクラスにはupdateメソッドがあります。https://docs.ruby-lang.org/ja/latest/method/Hash/i/update.html
        super(other_hash)
      else
        # to_hashすることで文字列とシンボルのkeyを区別している?(でもifの条件的に違う気もする。)


        other_hash.to_hash.each_pair do |key, value|
          if block_given? && key?(key)
            # convert_keyはkeyをstringにしているだけ
            value = yield(convert_key(key), self[key], value)
          end
          # regular_writerは[]=(Hashクラスのメソッド) のaliasです。
	    # convert_valueも基本はそのままvalue返してるけど、Hashや配列の時とかめんどいみたい
          regular_writer(convert_key(key), convert_value(value))
        end
        self
      end
    end

    alias_method :merge!, :update

    # Checks the hash for a key matching the argument passed in:
    #
    #   hash = ActiveSupport::HashWithIndifferentAccess.new
    #   hash['key'] = 'value'
    #   hash.key?(:key)  # => true
    #   hash.key?('key') # => true
    def key?(key)
      super(convert_key(key))
    end

    alias_method :include?, :key?
    alias_method :has_key?, :key?
    alias_method :member?, :key?


    # Same as <tt>Hash#[]</tt> where the key passed as argument can be
    # either a string or a symbol:
    #
    #   counters = ActiveSupport::HashWithIndifferentAccess.new
    #   counters[:foo] = 1
    #
    #   counters['foo'] # => 1
    #   counters[:foo]  # => 1
    #   counters[:zoo]  # => nil
    def [](key)
      super(convert_key(key))
    end

    # Same as <tt>Hash#fetch</tt> where the key passed as argument can be
    # either a string or a symbol:
    #
    #   counters = ActiveSupport::HashWithIndifferentAccess.new
    #   counters[:foo] = 1
    #
    #   counters.fetch('foo')          # => 1
    #   counters.fetch(:bar, 0)        # => 0
    #   counters.fetch(:bar) { |key| 0 } # => 0
    #   counters.fetch(:zoo)           # => KeyError: key not found: "zoo"
    def fetch(key, *extras)
      super(convert_key(key), *extras)
    end

    # Returns an array of the values at the specified indices:
    #
    #   hash = ActiveSupport::HashWithIndifferentAccess.new
    #   hash[:a] = 'x'
    #   hash[:b] = 'y'
    #   hash.values_at('a', 'b') # => ["x", "y"]
    def values_at(*indices)
      indices.collect { |key| self[convert_key(key)] }
    end

    # Returns a shallow copy of the hash.
    #
    #   hash = ActiveSupport::HashWithIndifferentAccess.new({ a: { b: 'b' } })
    #   dup  = hash.dup
    #   dup[:a][:c] = 'c'
    #
    #   hash[:a][:c] # => nil
    #   dup[:a][:c]  # => "c"
    def dup
      self.class.new(self).tap do |new_hash|
        set_defaults(new_hash)
      end
    end

    # This method has the same semantics of +update+, except it does not
    # modify the receiver but rather returns a new hash with indifferent
    # access with the result of the merge.
    def merge(hash, &block)
      self.dup.update(hash, &block)
    end

    # Like +merge+ but the other way around: Merges the receiver into the
    # argument and returns a new hash with indifferent access as result:
    #
    #   hash = ActiveSupport::HashWithIndifferentAccess.new
    #   hash['a'] = nil
    #   hash.reverse_merge(a: 0, b: 1) # => {"a"=>nil, "b"=>1}
    def reverse_merge(other_hash)
      super(self.class.new(other_hash))
    end

    # Same semantics as +reverse_merge+ but modifies the receiver in-place.
    def reverse_merge!(other_hash)
      replace(reverse_merge( other_hash ))
    end

    # Replaces the contents of this hash with other_hash.
    #
    #   h = { "a" => 100, "b" => 200 }
    #   h.replace({ "c" => 300, "d" => 400 }) # => {"c"=>300, "d"=>400}
    def replace(other_hash)
      super(self.class.new(other_hash))
    end

    # Removes the specified key from the hash.
    def delete(key)
      super(convert_key(key))
    end

    def stringify_keys!; self end
    def deep_stringify_keys!; self end
    def stringify_keys; dup end
    def deep_stringify_keys; dup end
    undef :symbolize_keys!
    undef :deep_symbolize_keys!
    def symbolize_keys; to_hash.symbolize_keys! end
    def deep_symbolize_keys; to_hash.deep_symbolize_keys! end
    def to_options!; self end

    def select(*args, &block)
      return to_enum(:select) unless block_given?
      dup.tap { |hash| hash.select!(*args, &block) }
    end

    def reject(*args, &block)
      return to_enum(:reject) unless block_given?
      dup.tap { |hash| hash.reject!(*args, &block) }
    end

    # Convert to a regular hash with string keys.
    def to_hash
      _new_hash = Hash.new
      set_defaults(_new_hash)

      each do |key, value|
        _new_hash[key] = convert_value(value, for: :to_hash)
      end
      _new_hash
    end

    protected
      def convert_key(key)
        key.kind_of?(Symbol) ? key.to_s : key
      end

      def convert_value(value, options = {})
        if value.is_a? Hash
          if options[:for] == :to_hash
            # hashを返す
            value.to_hash
          else
            # hash_with_indifferent_accessを返す
            value.nested_under_indifferent_access
          end
        elsif value.is_a?(Array)
          if options[:for] != :assignment || value.frozen?
            # どうやら副作用を防いでいる模様。frozenだとvalueを変えた時にエラーになって困る
            value = value.dup
          end
          value.map! { |e| convert_value(e, options) }
        else
          value
        end
      end

      def set_defaults(target)
        if default_proc
          target.default_proc = default_proc.dup
        else
          target.default = default
        end
      end
  end
end

HashWithIndifferentAccess = ActiveSupport::HashWithIndifferentAccess

```



## 3. active_model/dirty.rb




```RUBY
require 'active_support/hash_with_indifferent_access'
require 'active_support/core_ext/object/duplicable'

module ActiveModel
  # == Active \Model \Dirty
  #
  # Provides a way to track changes in your object in the same way as
  # Active Record does.

  # Dirtyというのは変数が変わってしまった時のことを言ったりする。(何度も変数の値が変わったら追うの大変だよね?)
  # 例が素晴らしく優秀なので、例を追うと非常にわかりやすい

  # The requirements for implementing ActiveModel::Dirty are:
  #
  # * <tt>include ActiveModel::Dirty</tt> in your object.
  # * Call <tt>define_attribute_methods</tt> passing each method you want to
  #   track.
  # * Call <tt>[attr_name]_will_change!</tt> before each change to the tracked
  #   attribute.
  # * Call <tt>changes_applied</tt> after the changes are persisted.
  # * Call <tt>clear_changes_information</tt> when you want to reset the changes
  #   information.
  # * Call <tt>restore_attributes</tt> when you want to restore previous data.
  #
  # A minimal implementation could be:
  #
  #   class Person
  #     include ActiveModel::Dirty
  #
  #     define_attribute_methods :name
  #
  #     def initialize(name)
  #       @name = name
  #     end
  #
  #     def name
  #       @name
  #     end
  #
  #     def name=(val)
  #       name_will_change! unless val == @name
  #       @name = val
  #     end
  #
  #     def save
  #       # do persistence work
  #
  #       changes_applied
  #     end
  #
  #     def reload!
  #       # get the values from the persistence layer
  #
  #       clear_changes_information
  #     end
  #
  #     def rollback!
  #       restore_attributes
  #     end
  #   end
  #
  # A newly instantiated +Person+ object is unchanged:
  #
  #   person = Person.new("Uncle Bob")
  #   person.changed? # => false
  #
  # Change the name:
  #
  #   person.name = 'Bob'
  #   person.changed?       # => true
  #   person.name_changed?  # => true
  #   person.name_changed?(from: "Uncle Bob", to: "Bob") # => true
  #   person.name_was       # => "Uncle Bob"
  #   person.name_change    # => ["Uncle Bob", "Bob"]
  #   person.name = 'Bill'
  #   person.name_change    # => ["Uncle Bob", "Bill"]
  #
  # Save the changes:
  #
  #   person.save
  #   person.changed?      # => false
  #   person.name_changed? # => false
  #
  # Reset the changes:
  #
  #   person.previous_changes         # => {"name" => ["Uncle Bob", "Bill"]}
  #   person.name_previously_changed? # => true
  #   person.name_previous_change     # => ["Uncle Bob", "Bill"]
  #   person.reload!
  #   person.previous_changes         # => {}
  #
  # Rollback the changes:
  #
  #   person.name = "Uncle Bob"
  #   person.rollback!
  #   person.name          # => "Bill"
  #   person.name_changed? # => false
  #
  # Assigning the same value leaves the attribute unchanged:
  #
  #   person.name = 'Bill'
  #   person.name_changed? # => false
  #   person.name_change   # => nil
  #
  # Which attributes have changed?
  #
  #   person.name = 'Bob'
  #   person.changed # => ["name"]
  #   person.changes # => {"name" => ["Bill", "Bob"]}
  #
  # If an attribute is modified in-place then make use of
  # <tt>[attribute_name]_will_change!</tt> to mark that the attribute is changing.
  # Otherwise \Active \Model can't track changes to in-place attributes. Note
  # that Active Record can detect in-place modifications automatically. You do
  # not need to call <tt>[attribute_name]_will_change!</tt> on Active Record models.
  #
  #   person.name_will_change!
  #   person.name_change # => ["Bill", "Bill"]
  #   person.name << 'y'
  #   person.name_change # => ["Bill", "Billy"]
  module Dirty
    extend ActiveSupport::Concern
    include ActiveModel::AttributeMethods

    OPTION_NOT_GIVEN = Object.new # :nodoc:
    private_constant :OPTION_NOT_GIVEN

    included do
      # selfはDirty
      # ここはAttributeMethodsでやったやつですね
      attribute_method_suffix '_changed?', '_change', '_will_change!', '_was'
      attribute_method_suffix '_previously_changed?', '_previous_change'
      attribute_method_affix prefix: 'restore_', suffix: '!'
    end

    # Returns +true+ if any of the attributes have unsaved changes, +false+ otherwise.
    #
    #   person.changed? # => false
    #   person.name = 'bob'
    #   person.changed? # => true
    
    # changed_attributesが変化を追っていることがわかる
    def changed?
      changed_attributes.present?
    end

    # Returns an array with the name of the attributes with unsaved changes.
    #
    #   person.changed # => []
    #   person.name = 'bob'
    #   person.changed # => ["name"]
  
    # keysはhashのkey一覧を配列にして返す
　　def changed
      changed_attributes.keys
    end

    # Returns a hash of changed attributes indicating their original
    # and new values like <tt>attr => [original value, new value]</tt>.
    #
    #   person.changes # => {}
    #   person.name = 'bob'
    #   person.changes # => { "name" => ["bill", "bob"] }
    # 飛ばす。あとで戻る予定。
    def changes
      ActiveSupport::HashWithIndifferentAccess[changed.map { |attr| [attr, attribute_change(attr)] }]
    end

    # Returns a hash of attributes that were changed before the model was saved.
    #
    #   person.name # => "bob"
    #   person.name = 'robert'
    #   person.save
    #   person.previous_changes # => {"name" => ["bob", "robert"]}
    def previous_changes
      @previously_changed ||= ActiveSupport::HashWithIndifferentAccess.new
    end

    # Returns a hash of the attributes with unsaved changes indicating their original
    # values like <tt>attr => original value</tt>.
    #
    #   person.name # => "bob"
    #   person.name = 'robert'
    #   person.changed_attributes # => {"name" => "bob"}
    def changed_attributes
      @changed_attributes ||= ActiveSupport::HashWithIndifferentAccess.new
    end

    # Handles <tt>*_changed?</tt> for +method_missing+.
    def attribute_changed?(attr, from: OPTION_NOT_GIVEN, to: OPTION_NOT_GIVEN) # :nodoc:
    # 真偽値が返ってきているから、!!いらない気がする。 
　　 !!changes_include?(attr) &&
        (to == OPTION_NOT_GIVEN || to == __send__(attr)) &&
        (from == OPTION_NOT_GIVEN || from == changed_attributes[attr])
    end

    # Handles <tt>*_was</tt> for +method_missing+.
    def attribute_was(attr) # :nodoc:
      attribute_changed?(attr) ? changed_attributes[attr] : __send__(attr)
    end

    # Handles <tt>*_previously_changed?</tt> for +method_missing+.
    def attribute_previously_changed?(attr) #:nodoc:
      previous_changes_include?(attr)
    end

    # Restore all previous data of the provided attributes.
    def restore_attributes(attributes = changed)
      attributes.each { |attr| restore_attribute! attr }
    end

    private

      # Returns +true+ if attr_name is changed, +false+ otherwise.
      def changes_include?(attr_name)
        #attributes_changed_by_setterはchanged_attributeの別名
        attributes_changed_by_setter.include?(attr_name)
      end
      alias attribute_changed_by_setter? changes_include?

      # Returns +true+ if attr_name were changed before the model was saved,
      # +false+ otherwise.
      def previous_changes_include?(attr_name)
        previous_changes.include?(attr_name)
      end

      # Removes current changes and makes them accessible through +previous_changes+.
      def changes_applied # :doc:
        @previously_changed = changes
        @changed_attributes = ActiveSupport::HashWithIndifferentAccess.new
      end

      # Clears all dirty data: current changes and previous changes.
      def clear_changes_information # :doc:
        @previously_changed = ActiveSupport::HashWithIndifferentAccess.new
        @changed_attributes = ActiveSupport::HashWithIndifferentAccess.new
      end

      # Handles <tt>*_change</tt> for +method_missing+.
      def attribute_change(attr)
        [changed_attributes[attr], __send__(attr)] if attribute_changed?(attr)
      end

      # Handles <tt>*_previous_change</tt> for +method_missing+.
      def attribute_previous_change(attr)
        previous_changes[attr] if attribute_previously_changed?(attr)
      end

      # Handles <tt>*_will_change!</tt> for +method_missing+.
      def attribute_will_change!(attr)
        return if attribute_changed?(attr)

        begin
          value = __send__(attr)
          value = value.duplicable? ? value.clone : value
        rescue TypeError, NoMethodError
        end

        set_attribute_was(attr, value)
      end

      # Handles <tt>restore_*!</tt> for +method_missing+.
      def restore_attribute!(attr)
        if attribute_changed?(attr)
          __send__("#{attr}=", changed_attributes[attr])
          clear_attribute_changes([attr])
        end
      end

      # This is necessary because `changed_attributes` might be overridden in
      # other implementations (e.g. in `ActiveRecord`)
      alias_method :attributes_changed_by_setter, :changed_attributes # :nodoc:

      # Force an attribute to have a particular "before" value
      def set_attribute_was(attr, old_value)
        attributes_changed_by_setter[attr] = old_value
      end

      # Remove changes information for the provided attributes.
      def clear_attribute_changes(attributes) # :doc:
        attributes_changed_by_setter.except!(*attributes)
      end
  end
end


```


