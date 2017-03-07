# Day5

## 今日の内容

1. active_model/errors.rb
2. active_support/deprecation.rb
3. active_model/forbidden_attributes_protection.rb
4. activesupport/lib/active_support/i18n_railtie.rb

## 1. active_model/errors.rb

```rb
module ActiveModel
  # == Active \Model \Errors
  # 
  # Provides a modified +Hash+ that you can include in your object
  # for handling error messages and interacting with Action View helpers.
  # オブジェクトにエラーメッセージの取り扱いやActionView::Helpersとの連携のために変更されたHashを提供します
  # * ActionViewはRailsのView部分で使われている
  #
  # A minimal implementation could be:
  #
  #   class Person
  #     # Required dependency for ActiveModel::Errors
  #     extend ActiveModel::Naming
  #
  #     def initialize
  #       # ActiveModel::Errorsオブジェクトをselfを引数にして生成
  #       @errors = ActiveModel::Errors.new(self)
  #     end
  #     
  #     attr_accessor :name
  #     attr_reader   :errors
  #
  #     def validate!
  #       # nameがnilの場合errorsにエラーメッセージを追加
  #       errors.add(:name, :blank, message: "cannot be nil") if name.nil?
  #     end
  # 
  #     # 必須メソッド : error messagesの生成に利用
  #     # ActiveModel::Translationをextendすれば不要になる
  #     # The following methods are needed to be minimally implemented
  #
  #     def read_attribute_for_validation(attr)
  #       send(attr)
  #     end
  #
  #     def self.human_attribute_name(attr, options = {})
  #       attr
  #     end
  #
  #     def self.lookup_ancestors
  #       [self]
  #     end
  #   end
  #
  # The last three methods are required in your object for +Errors+ to be
  # able to generate error messages correctly and also handle multiple
  # languages. Of course, if you extend your object with <tt>ActiveModel::Translation</tt>
  # you will not need to implement the last two. Likewise, using
  # <tt>ActiveModel::Validations</tt> will handle the validation related methods
  # for you.
  #
  # The above allows you to do:
  #
  #   person = Person.new
  #   person.validate!            # => ["cannot be nil"]
  #   person.errors.full_messages # => ["name cannot be nil"]
  #   # etc..
  class Errors
    include Enumerable

    CALLBACKS_OPTIONS = [:if, :unless, :on, :allow_nil, :allow_blank, :strict]
    MESSAGE_OPTIONS = [:message]

    attr_reader :messages, :details

    # Pass in the instance of the object that is using the errors object.
    #
    #   class Person
    #     def initialize
    #       @errors = ActiveModel::Errors.new(self)
    #     end
    #   end
    def initialize(base)
      @base     = base # ベースクラス
      @messages = apply_default_array({}) # エラーメッセージ
      @details = apply_default_array({}) 
    end

    # 浅いコピーを行う
    def initialize_dup(other) # :nodoc:
      @messages = other.messages.dup
      @details  = other.details.deep_dup
      super
    end

    # Copies the errors from <tt>other</tt>.
    #
    # other - The ActiveModel::Errors instance.
    #
    # Examples
    #
    #   person.errors.copy!(other)
    # 別のオブジェクトのエラーからコピーしてくる
    def copy!(other) # :nodoc:
      @messages = other.messages.dup
      @details  = other.details.dup
    end

    # Clear the error messages.
    #
    #   person.errors.full_messages # => ["name cannot be nil"]
    #   person.errors.clear
    #   person.errors.full_messages # => []
    # エラーのクリア
    def clear
      messages.clear
      details.clear
    end

    # Returns +true+ if the error messages include an error for the given key
    # +attribute+, +false+ otherwise.
    #
    #   person.errors.messages        # => {:name=>["cannot be nil"]}
    #   person.errors.include?(:name) # => true
    #   person.errors.include?(:age)  # => false
    # 対象のアトリビューションにエラーが存在するか確認
    def include?(attribute)
      messages.key?(attribute) && messages[attribute].present?
    end
    alias :has_key? :include?
    alias :key? :include?

    # Get messages for +key+.
    #
    #   person.errors.messages   # => {:name=>["cannot be nil"]}
    #   person.errors.get(:name) # => ["cannot be nil"]
    #   person.errors.get(:age)  # => []
    # 対象のアトリビューションのエラーを取得
    def get(key)
      ActiveSupport::Deprecation.warn(<<-MESSAGE.squish)
        ActiveModel::Errors#get is deprecated and will be removed in Rails 5.1.

        To achieve the same use model.errors[:#{key}].
      MESSAGE

      messages[key]
    end

    # Set messages for +key+ to +value+.
    #
    #   person.errors[:name] # => ["cannot be nil"]
    #   person.errors.set(:name, ["can't be nil"])
    #   person.errors[:name] # => ["can't be nil"]
    # 対象のアトリビューションにエラーを設定
    def set(key, value)
      ActiveSupport::Deprecation.warn(<<-MESSAGE.squish)
        ActiveModel::Errors#set is deprecated and will be removed in Rails 5.1.

        Use model.errors.add(:#{key}, #{value.inspect}) instead.
      MESSAGE

      messages[key] = value
    end

    # Delete messages for +key+. Returns the deleted messages.
    #
    #   person.errors[:name]        # => ["cannot be nil"]
    #   person.errors.delete(:name) # => ["cannot be nil"]
    #   person.errors[:name]        # => []
    #　対象のアトリビューションからエラーを削除
    def delete(key)
      details.delete(key)
      messages.delete(key)
    end

    # When passed a symbol or a name of a method, returns an array of errors
    # for the method.
    #
    #   person.errors[:name]  # => ["cannot be nil"]
    #   person.errors['name'] # => ["cannot be nil"]
    #
    # Note that, if you try to get errors of an attribute which has
    # no errors associated with it, this method will instantiate
    # an empty error list for it and +keys+ will return an array
    # of error keys which includes this attribute.
    #
    #   person.errors.keys    # => []
    #   person.errors[:name]  # => []
    #   person.errors.keys    # => [:name]
    # SymbolとStringの双方でアクセス可能にしている
    # 対象にしたアトリビューションが存在しなかった場合はその値をkeyにして空配列を生成
    def [](attribute)
      messages[attribute.to_sym]
    end

    # Adds to the supplied attribute the supplied error message.
    #
    #   person.errors[:name] = "must be set"
    #   person.errors[:name] # => ['must be set']
    # 対象のアトリビューションにメッセージを追加
    def []=(attribute, error)
      ActiveSupport::Deprecation.warn(<<-MESSAGE.squish)
        ActiveModel::Errors#[]= is deprecated and will be removed in Rails 5.1.

        Use model.errors.add(:#{attribute}, #{error.inspect}) instead.
      MESSAGE

      messages[attribute.to_sym] << error
    end

    # Iterates through each error key, value pair in the error messages hash.
    # Yields the attribute and the error for that attribute. If the attribute
    # has more than one error message, yields once for each error message.
    #
    #   person.errors.add(:name, :blank, message: "can't be blank")
    #   person.errors.each do |attribute, error|
    #     # Will yield :name and "can't be blank"
    #   end
    #
    #   person.errors.add(:name, :not_specified, message: "must be specified")
    #   person.errors.each do |attribute, error|
    #     # Will yield :name and "can't be blank"
    #     # then yield :name and "must be specified"
    #   end
    # オブジェクトに含まれるエラーキー、値を返す
    def each
      messages.each_key do |attribute|
        messages[attribute].each { |error| yield attribute, error }
      end
    end

    # Returns the number of error messages.
    #
    #   person.errors.add(:name, :blank, message: "can't be blank")
    #   person.errors.size # => 1
    #   person.errors.add(:name, :not_specified, message: "must be specified")
    #   person.errors.size # => 2
    # エラーのサイズを返す
    def size
      values.flatten.size
    end
    alias :count :size

    # Returns all message values.
    #
    #   person.errors.messages # => {:name=>["cannot be nil", "must be specified"]}
    #   person.errors.values   # => [["cannot be nil", "must be specified"]]
    # エラーメッセージを返す
    def values
      messages.values
    end

    # Returns all message keys.
    #
    #   person.errors.messages # => {:name=>["cannot be nil", "must be specified"]}
    #   person.errors.keys     # => [:name]
    # エラーメッセージのキーを返す
    def keys
      messages.keys
    end

    # Returns +true+ if no errors are found, +false+ otherwise.
    # If the error message is a string it can be empty.
    #
    #   person.errors.full_messages # => ["name cannot be nil"]
    #   person.errors.empty?        # => false
    # エラーが存在するか確認する
    def empty?
      size.zero?
    end
    alias :blank? :empty?

    # Returns an xml formatted representation of the Errors hash.
    #
    #   person.errors.add(:name, :blank, message: "can't be blank")
    #   person.errors.add(:name, :not_specified, message: "must be specified")
    #   person.errors.to_xml
    #   # =>
    #   #  <?xml version=\"1.0\" encoding=\"UTF-8\"?>
    #   #  <errors>
    #   #    <error>name can't be blank</error>
    #   #    <error>name must be specified</error>
    #   #  </errors>
    # XMLとして出力
    def to_xml(options={})
      to_a.to_xml({ root: "errors", skip_types: true }.merge!(options))
    end

    # Returns a Hash that can be used as the JSON representation for this
    # object. You can pass the <tt>:full_messages</tt> option. This determines
    # if the json object should contain full messages or not (false by default).
    #
    #   person.errors.as_json                      # => {:name=>["cannot be nil"]}
    #   person.errors.as_json(full_messages: true) # => {:name=>["name cannot be nil"]}
    # Jsonとして出力
    def as_json(options=nil)
      to_hash(options && options[:full_messages])
    end

    # Returns a Hash of attributes with their error messages. If +full_messages+
    # is +true+, it will contain full messages (see +full_message+).
    #
    #   person.errors.to_hash       # => {:name=>["cannot be nil"]}
    #   person.errors.to_hash(true) # => {:name=>["name cannot be nil"]}
    # hashとして出力
    def to_hash(full_messages = false)
      if full_messages
        self.messages.each_with_object({}) do |(attribute, array), messages|
          messages[attribute] = array.map { |message| full_message(attribute, message) }
        end
      else
        without_default_proc(self.messages)
      end
    end

    # Adds +message+ to the error messages and used validator type to +details+ on +attribute+.
    # More than one error can be added to the same +attribute+.
    # If no +message+ is supplied, <tt>:invalid</tt> is assumed.
    #
    #   person.errors.add(:name)
    #   # => ["is invalid"]
    #   person.errors.add(:name, :not_implemented, message: "must be implemented")
    #   # => ["is invalid", "must be implemented"]
    #
    #   person.errors.messages
    #   # => {:name=>["is invalid", "must be implemented"]}
    #
    #   person.errors.details
    #   # => {:name=>[{error: :not_implemented}, {error: :invalid}]}
    #
    # If +message+ is a symbol, it will be translated using the appropriate
    # scope (see +generate_message+).
    #
    # If +message+ is a proc, it will be called, allowing for things like
    # <tt>Time.now</tt> to be used within an error.
    #
    # If the <tt>:strict</tt> option is set to +true+, it will raise
    # ActiveModel::StrictValidationFailed instead of adding the error.
    # <tt>:strict</tt> option can also be set to any other exception.
    #
    #   person.errors.add(:name, :invalid, strict: true)
    #   # => ActiveModel::StrictValidationFailed: name is invalid
    #   person.errors.add(:name, :invalid, strict: NameIsInvalid)
    #   # => NameIsInvalid: name is invalid
    #
    #   person.errors.messages # => {}
    #
    # +attribute+ should be set to <tt>:base</tt> if the error is not
    # directly associated with a single attribute.
    #
    #   person.errors.add(:base, :name_or_email_blank,
    #     message: "either name or email must be present")
    #   person.errors.messages
    #   # => {:base=>["either name or email must be present"]}
    #   person.errors.details
    #   # => {:base=>[{error: :name_or_email_blank}]}
    # attributeのエラーメッセージを追加する
    #   strictオプションを使うとエラーを追加する代わりにActiveModel::StrictValidationFailedを返す
    def add(attribute, message = :invalid, options = {})
      # messageをブロックで渡せる？
      message = message.call if message.respond_to?(:call)
      # メッセージの他言語対応を行う
      detail  = normalize_detail(message, options)
      message = normalize_message(attribute, message, options)
      if exception = options[:strict]
        exception = ActiveModel::StrictValidationFailed if exception == true
        raise exception, full_message(attribute, message)
      end

      details[attribute.to_sym]  << detail
      messages[attribute.to_sym] << message
    end

    # Will add an error message to each of the attributes in +attributes+
    # that is empty.
    #
    #   person.errors.add_on_empty(:name)
    #   person.errors.messages
    #   # => {:name=>["can't be empty"]}
    # メッセージが存在しないアトリビューションに「:empty」のエラーメッセージを追加する
    def add_on_empty(attributes, options = {})
      ActiveSupport::Deprecation.warn(<<-MESSAGE.squish)
        ActiveModel::Errors#add_on_empty is deprecated and will be removed in Rails 5.1.

        To achieve the same use:

          errors.add(attribute, :empty, options) if value.nil? || value.empty?
      MESSAGE

      Array(attributes).each do |attribute|
        # read_attribute_for_validation
        # ActiveModel::Validationsに実装されたattributeの取得方法を定義するフックメソッド.
        # バリデーション名とattribute名が異なる場合に使用すると思われる
        # ex) ネストした要素とか？
        # http://qiita.com/yuku_t/items/11e6f13a6a7e2dbb88a4
        value = @base.send(:read_attribute_for_validation, attribute)
        is_empty = value.respond_to?(:empty?) ? value.empty? : false
        # is_emptyが存在しなければ「:empty」で追加
        add(attribute, :empty, options) if value.nil? || is_empty
      end
    end

    # Will add an error message to each of the attributes in +attributes+ that
    # is blank (using Object#blank?).
    #
    #   person.errors.add_on_blank(:name)
    #   person.errors.messages
    #   # => {:name=>["can't be blank"]}
    # メッセージが存在しないアトリビューションに「:blank」のエラーメッセージを追加する
    def add_on_blank(attributes, options = {})
      ActiveSupport::Deprecation.warn(<<-MESSAGE.squish)
        ActiveModel::Errors#add_on_blank is deprecated and will be removed in Rails 5.1.

        To achieve the same use:

          errors.add(attribute, :empty, options) if value.blank?
      MESSAGE

      Array(attributes).each do |attribute|
        value = @base.send(:read_attribute_for_validation, attribute)
        add(attribute, :blank, options) if value.blank?
      end
    end

    # Returns +true+ if an error on the attribute with the given message is
    # present, or +false+ otherwise. +message+ is treated the same as for +add+.
    #
    #   person.errors.add :name, :blank
    #   person.errors.added? :name, :blank           # => true
    #   person.errors.added? :name, "can't be blank" # => true
    #
    # If the error message requires an option, then it returns +true+ with
    # the correct option, or +false+ with an incorrect or missing option.
    #
    #  person.errors.add :name, :too_long, { count: 25 }
    #  person.errors.added? :name, :too_long, count: 25                     # => true
    #  person.errors.added? :name, "is too long (maximum is 25 characters)" # => true
    #  person.errors.added? :name, :too_long, count: 24                     # => false
    #  person.errors.added? :name, :too_long                                # => false
    #  person.errors.added? :name, "is too long"                            # => false
    # 
    # 対象のattributeに指定したメッセージが追加されているか確認
    def added?(attribute, message = :invalid, options = {})
      message = message.call if message.respond_to?(:call)
      # 
      message = normalize_message(attribute, message, options)
      self[attribute].include? message
    end

    # Returns all the full error messages in an array.
    #
    #   class Person
    #     validates_presence_of :name, :address, :email
    #     validates_length_of :name, in: 5..30
    #   end
    #
    #   person = Person.create(address: '123 First St.')
    #   person.errors.full_messages
    #   # => ["Name is too short (minimum is 5 characters)", "Name can't be blank", "Email can't be blank"]
    # エラーメッセージをArrayで出力
    def full_messages
      map { |attribute, message| full_message(attribute, message) }
    end
    alias :to_a :full_messages

    # Returns all the full error messages for a given attribute in an array.
    #
    #   class Person
    #     validates_presence_of :name, :email
    #     validates_length_of :name, in: 5..30
    #   end
    #
    #   person = Person.create()
    #   person.errors.full_messages_for(:name)
    #   # => ["Name is too short (minimum is 5 characters)", "Name can't be blank"]
    # 対象のアトリビューションのエラーメッセージを出力
    def full_messages_for(attribute)
      messages[attribute].map { |message| full_message(attribute, message) }
    end

    # Returns a full message for a given attribute.
    #
    #   person.errors.full_message(:name, 'is invalid') # => "Name is invalid"
    # 
    # 対象のアトリビューションに与えられたメッセージを付加して出力
    def full_message(attribute, message)
      return message if attribute == :base
      attr_name = attribute.to_s.tr('.', '_').humanize
      attr_name = @base.class.human_attribute_name(attribute, default: attr_name)
      I18n.t(:"errors.format", {
        default:  "%{attribute} %{message}",
        attribute: attr_name,
        message:   message
      })
    end

    # Translates an error message in its default scope
    # (<tt>activemodel.errors.messages</tt>).
    #
    # Error messages are first looked up in <tt>activemodel.errors.models.MODEL.attributes.ATTRIBUTE.MESSAGE</tt>,
    # if it's not there, it's looked up in <tt>activemodel.errors.models.MODEL.MESSAGE</tt> and if
    # that is not there also, it returns the translation of the default message
    # (e.g. <tt>activemodel.errors.messages.MESSAGE</tt>). The translated model
    # name, translated attribute name and the value are available for
    # interpolation.
    #
    # When using inheritance in your models, it will check all the inherited
    # models too, but only if the model itself hasn't been found. Say you have
    # <tt>class Admin < User; end</tt> and you wanted the translation for
    # the <tt>:blank</tt> error message for the <tt>title</tt> attribute,
    # it looks for these translations:
    #
    # * <tt>activemodel.errors.models.admin.attributes.title.blank</tt>
    # * <tt>activemodel.errors.models.admin.blank</tt>
    # * <tt>activemodel.errors.models.user.attributes.title.blank</tt>
    # * <tt>activemodel.errors.models.user.blank</tt>
    # * any default you provided through the +options+ hash (in the <tt>activemodel.errors</tt> scope)
    # * <tt>activemodel.errors.messages.blank</tt>
    # * <tt>errors.attributes.title.blank</tt>
    # * <tt>errors.messages.blank</tt>
    def generate_message(attribute, type = :invalid, options = {})
      # optionsにmessageの値があればそれを代入
      type = options.delete(:message) if options[:message].is_a?(Symbol)

      # モデルごとに多言語対応している場合のファイルパス
      if @base.class.respond_to?(:i18n_scope)
        defaults = @base.class.lookup_ancestors.map do |klass|
          [ :"#{@base.class.i18n_scope}.errors.models.#{klass.model_name.i18n_key}.attributes.#{attribute}.#{type}",
            :"#{@base.class.i18n_scope}.errors.models.#{klass.model_name.i18n_key}.#{type}" ]
        end
      else
        defaults = []
      end

      # 多言語対応用のファイルパス
      defaults << :"#{@base.class.i18n_scope}.errors.messages.#{type}" if @base.class.respond_to?(:i18n_scope)
      # デフォルトのファイルパス
      defaults << :"errors.attributes.#{attribute}.#{type}"
      defaults << :"errors.messages.#{type}"

      # 整理
      defaults.compact!
      defaults.flatten!

      # Array#shift: 配列の最初の要素を削除し、その要素を返す
      key = defaults.shift
      defaults = options.delete(:message) if options[:message]
      value = (attribute != :base ? @base.send(:read_attribute_for_validation, attribute) : nil)

      options = {
        default: defaults,
        model: @base.model_name.human,
        attribute: @base.class.human_attribute_name(attribute),
        value: value,
        object: @base
      }.merge!(options)

      # ファイルパスとオプション
      I18n.translate(key, options)
    end

    def marshal_dump
      [@base, without_default_proc(@messages), without_default_proc(@details)]
    end

    def marshal_load(array)
      @base, @messages, @details = array
      apply_default_array(@messages)
      apply_default_array(@details)
    end

  private
    def normalize_message(attribute, message, options)
      case message
      when Symbol
        generate_message(attribute, message, options.except(*CALLBACKS_OPTIONS))
      else
        message
      end
    end

    def normalize_detail(message, options)
      { error: message }.merge(options.except(*CALLBACKS_OPTIONS + MESSAGE_OPTIONS))
    end

    def without_default_proc(hash)
      hash.dup.tap do |new_h|
        new_h.default_proc = nil
      end
    end

    def apply_default_array(hash)
      # 存在しないkeyで呼ばれた場合の処理設定
      # 空の配列を生成
      hash.default_proc = proc { |h, key| h[key] = [] }
      hash
    end
  end

  # Raised when a validation cannot be corrected by end users and are considered
  # exceptional.
  #
  #   class Person
  #     include ActiveModel::Validations
  #
  #     attr_accessor :name
  #
  #     validates_presence_of :name, strict: true
  #   end
  #
  #   person = Person.new
  #   person.name = nil
  #   person.valid?
  #   # => ActiveModel::StrictValidationFailed: Name can't be blank
  class StrictValidationFailed < StandardError
  end

  # Raised when attribute values are out of range.
  class RangeError < ::RangeError
  end

  # Raised when unknown attributes are supplied via mass assignment.
  class UnknownAttributeError < NoMethodError
    attr_reader :record, :attribute

    def initialize(record, attribute)
      @record = record
      @attribute = attribute
      super("unknown attribute '#{attribute}' for #{@record.class}.")
    end
  end
end
```

## 2. active_support/deprecation.rb
廃止予定であることを通知するために使うクラス

```rb
require 'singleton'

module ActiveSupport
  # \Deprecation specifies the API used by Rails to deprecate methods, instance
  # variables, objects and constants.
  class Deprecation
    # active_support.rb sets an autoload for ActiveSupport::Deprecation.
    #
    # If these requires were at the top of the file the constant would not be
    # defined by the time their files were loaded. Since some of them reopen
    # ActiveSupport::Deprecation its autoload would be triggered, resulting in
    # a circular require warning for active_support/deprecation.rb.
    #
    # So, we define the constant first, and load dependencies later.
    require 'active_support/deprecation/instance_delegator'
    require 'active_support/deprecation/behaviors'
    require 'active_support/deprecation/reporting'
    require 'active_support/deprecation/method_wrappers'
    require 'active_support/deprecation/proxy_wrappers'
    require 'active_support/core_ext/module/deprecation'

    # Singleton モジュールを include することにより、クラスは 高々ひとつのインスタンスしか持たないことが保証されます。
    # require 'singleton'
    # class SomeSingletonClass
    #   include Singleton
    # end
    # a = SomeSingletonClass.instance
    # b = SomeSingletonClass.instance  # a and b are same object
    # https://docs.ruby-lang.org/ja/latest/class/Singleton.html
    include Singleton
    include InstanceDelegator
    include Behavior
    include Reporting
    include MethodWrapper

    # The version number in which the deprecated behavior will be removed, by default.
    attr_accessor :deprecation_horizon

    # It accepts two parameters on initialization. The first is a version of library
    # and the second is a library name
    #
    #   ActiveSupport::Deprecation.new('2.0', 'MyLibrary')
    def initialize(deprecation_horizon = '5.1', gem_name = 'Rails')
      self.gem_name = gem_name
      self.deprecation_horizon = deprecation_horizon
      # By default, warnings are not silenced and debugging is off.
      self.silenced = false
      self.debug = false
    end
  end
end
```

active_support/deprecation/instance_delegator
DeprecationにincloudするModuleの振る舞い
インスタンスメソッドを

```rb
require 'active_support/core_ext/kernel/singleton_class'
require 'active_support/core_ext/module/delegation'

module ActiveSupport
  class Deprecation
    module InstanceDelegator # :nodoc:
      def self.included(base)
        # クラスメソッドにするやつ
        base.extend(ClassMethods)
        # baseクラスの継承直下にOverrideDelegatorsモジュールを入れる
        base.singleton_class.prepend(OverrideDelegators)
        # public_class_method: クラスやモジュールのクラスメソッドをpublicなメソッドに変える
        base.public_class_method :new
      end

      module ClassMethods # :nodoc:
        # DeprecationでInstanceDelegatorが読み込まれて以降includeを使う際にこのメソッドを通る
        def include(included_module)
          # included_moduleに存在するインスタンスメソッドをmethod_addedに渡す
          included_module.instance_methods.each { |m| method_added(m) }
          super
        end

        # method_nameはDeprecationにincloudされたmoduleのインスタンスメソッドのはず
        def method_added(method_name)
          # インスタンスメソッドとして定義されたメソッドをクラスに対して呼び出した時ににも使えるよう
          # デリゲートしている？
          singleton_class.delegate(method_name, to: :instance)
        end
      end

      # オーバーライトメソッド
      module OverrideDelegators # :nodoc:
        def warn(message = nil, callstack = nil)
          callstack ||= caller_locations(2)
          super
        end

        def deprecation_warning(deprecated_method_name, message = nil, caller_backtrace = nil)
          caller_backtrace ||= caller_locations(2)
          super
        end
      end
    end
  end
end
```

active_support/core_ext/kernel/singleton_class

```rb
module Kernel
  # class_eval on an object acts like singleton_class.class_eval.
  def class_eval(*args, &block)
    singleton_class.class_eval(*args, &block)
  end
end
```

active_support/core_ext/module/delegation

```rb
# 集合を表すクラス
require 'set'

class Module
  # Error generated by +delegate+ when a method is called on +nil+ and +allow_nil+
  # option is not used.
  class DelegationError < NoMethodError; end

  RUBY_RESERVED_KEYWORDS = %w(alias and BEGIN begin break case class def defined? do
  else elsif END end ensure false for if in module next nil not or redo rescue retry
  return self super then true undef unless until when while yield)
  DELEGATION_RESERVED_KEYWORDS = %w(_ arg args block)
  # 集合を生成
  DELEGATION_RESERVED_METHOD_NAMES = Set.new(
    RUBY_RESERVED_KEYWORDS + DELEGATION_RESERVED_KEYWORDS
  ).freeze

  # Provides a +delegate+ class method to easily expose contained objects'
  # public methods as your own.
  #
  # ==== Options
  # * <tt>:to</tt> - Specifies the target object
  # * <tt>:prefix</tt> - Prefixes the new method with the target name or a custom prefix
  # * <tt>:allow_nil</tt> - if set to true, prevents a +NoMethodError+ from being raised
  #
  # The macro receives one or more method names (specified as symbols or
  # strings) and the name of the target object via the <tt>:to</tt> option
  # (also a symbol or string).
  #
  # Delegation is particularly useful with Active Record associations:
  #
  #   class Greeter < ActiveRecord::Base
  #     def hello
  #       'hello'
  #     end
  #
  #     def goodbye
  #       'goodbye'
  #     end
  #   end
  #
  #   class Foo < ActiveRecord::Base
  #     belongs_to :greeter
  #     delegate :hello, to: :greeter
  #   end
  #
  #   Foo.new.hello   # => "hello"
  #   Foo.new.goodbye # => NoMethodError: undefined method `goodbye' for #<Foo:0x1af30c>
  #
  # Multiple delegates to the same target are allowed:
  #
  #   class Foo < ActiveRecord::Base
  #     belongs_to :greeter
  #     delegate :hello, :goodbye, to: :greeter
  #   end
  #
  #   Foo.new.goodbye # => "goodbye"
  #
  # Methods can be delegated to instance variables, class variables, or constants
  # by providing them as a symbols:
  #
  #   class Foo
  #     CONSTANT_ARRAY = [0,1,2,3]
  #     @@class_array  = [4,5,6,7]
  #
  #     def initialize
  #       @instance_array = [8,9,10,11]
  #     end
  #     delegate :sum, to: :CONSTANT_ARRAY
  #     delegate :min, to: :@@class_array
  #     delegate :max, to: :@instance_array
  #   end
  #
  #   Foo.new.sum # => 6
  #   Foo.new.min # => 4
  #   Foo.new.max # => 11
  #
  # It's also possible to delegate a method to the class by using +:class+:
  #
  #   class Foo
  #     def self.hello
  #       "world"
  #     end
  #
  #     delegate :hello, to: :class
  #   end
  #
  #   Foo.new.hello # => "world"
  #
  # Delegates can optionally be prefixed using the <tt>:prefix</tt> option. If the value
  # is <tt>true</tt>, the delegate methods are prefixed with the name of the object being
  # delegated to.
  #
  #   Person = Struct.new(:name, :address)
  #
  #   class Invoice < Struct.new(:client)
  #     delegate :name, :address, to: :client, prefix: true
  #   end
  #
  #   john_doe = Person.new('John Doe', 'Vimmersvej 13')
  #   invoice = Invoice.new(john_doe)
  #   invoice.client_name    # => "John Doe"
  #   invoice.client_address # => "Vimmersvej 13"
  #
  # It is also possible to supply a custom prefix.
  #
  #   class Invoice < Struct.new(:client)
  #     delegate :name, :address, to: :client, prefix: :customer
  #   end
  #
  #   invoice = Invoice.new(john_doe)
  #   invoice.customer_name    # => 'John Doe'
  #   invoice.customer_address # => 'Vimmersvej 13'
  #
  # If the target is +nil+ and does not respond to the delegated method a
  # +NoMethodError+ is raised, as with any other value. Sometimes, however, it
  # makes sense to be robust to that situation and that is the purpose of the
  # <tt>:allow_nil</tt> option: If the target is not +nil+, or it is and
  # responds to the method, everything works as usual. But if it is +nil+ and
  # does not respond to the delegated method, +nil+ is returned.
  #
  #   class User < ActiveRecord::Base
  #     has_one :profile
  #     delegate :age, to: :profile
  #   end
  #
  #   User.new.age # raises NoMethodError: undefined method `age'
  #
  # But if not having a profile yet is fine and should not be an error
  # condition:
  #
  #   class User < ActiveRecord::Base
  #     has_one :profile
  #     delegate :age, to: :profile, allow_nil: true
  #   end
  #
  #   User.new.age # nil
  #
  # Note that if the target is not +nil+ then the call is attempted regardless of the
  # <tt>:allow_nil</tt> option, and thus an exception is still raised if said object
  # does not respond to the method:
  #
  #   class Foo
  #     def initialize(bar)
  #       @bar = bar
  #     end
  #
  #     delegate :name, to: :@bar, allow_nil: true
  #   end
  #
  #   Foo.new("Bar").name # raises NoMethodError: undefined method `name'
  #
  # The target method must be public, otherwise it will raise +NoMethodError+.
  #
  def delegate(*methods, to: nil, prefix: nil, allow_nil: nil)
    unless to
      raise ArgumentError, 'Delegation needs a target. Supply an options hash with a :to key as the last argument (e.g. delegate :hello, to: :greeter).'
    end

    if prefix == true && to =~ /^[^a-z_]/
      raise ArgumentError, 'Can only automatically set the delegation prefix when delegating to a method.'
    end

    method_prefix = \
      if prefix
        "#{prefix == true ? to : prefix}_"
      else
        ''
      end

    location = caller_locations(1, 1).first
    file, line = location.path, location.lineno

    to = to.to_s
    to = "self.#{to}" if DELEGATION_RESERVED_METHOD_NAMES.include?(to)

    methods.each do |method|
      # Attribute writer methods only accept one argument. Makes sure []=
      # methods still accept two arguments.
      definition = (method =~ /[^\]]=$/) ? 'arg' : '*args, &block'

      # The following generated method calls the target exactly once, storing
      # the returned value in a dummy variable.
      #
      # Reason is twofold: On one hand doing less calls is in general better.
      # On the other hand it could be that the target has side-effects,
      # whereas conceptually, from the user point of view, the delegator should
      # be doing one call.
      #
      # methodという名前(prefixの指定可能)のメソッドをtoで指定された対象に自動的に送る処理
      if allow_nil
        method_def = [
          "def #{method_prefix}#{method}(#{definition})",
          "_ = #{to}",
          "if !_.nil? || nil.respond_to?(:#{method})",
          "  _.#{method}(#{definition})",
          "end",
        "end"
        ].join ';'
      else
        exception = %(raise DelegationError, "#{self}##{method_prefix}#{method} delegated to #{to}.#{method}, but #{to} is nil: \#{self.inspect}")

        method_def = [
          "def #{method_prefix}#{method}(#{definition})",
          " _ = #{to}",
          "  _.#{method}(#{definition})",
          "rescue NoMethodError => e",
          "  if _.nil? && e.name == :#{method}",
          "    #{exception}",
          "  else",
          "    raise",
          "  end",
          "end"
        ].join ';'
      end

      # メソッド定義
      module_eval(method_def, file, line)
    end
  end
end
```

active_support/deprecation/behaviors
エラーの表示内容をカスタマイズできる

```rb
require "active_support/notifications"

module ActiveSupport
  # Raised when <tt>ActiveSupport::Deprecation::Behavior#behavior</tt> is set with <tt>:raise</tt>.
  # You would set <tt>:raise</tt>, as a behaviour to raise errors and proactively report exceptions from deprecations.
  class DeprecationException < StandardError
  end

  class Deprecation
    # Default warning behaviors per Rails.env.
    DEFAULT_BEHAVIORS = {
      raise: ->(message, callstack) {
        e = DeprecationException.new(message)
        e.set_backtrace(callstack.map(&:to_s))
        raise e
      },

      stderr: ->(message, callstack) {
        $stderr.puts(message)
        $stderr.puts callstack.join("\n  ") if debug
      },

      log: ->(message, callstack) {
        logger =
            if defined?(Rails.logger) && Rails.logger
              Rails.logger
            else
              require 'active_support/logger'
              ActiveSupport::Logger.new($stderr)
            end
        logger.warn message
        logger.debug callstack.join("\n  ") if debug
      },

      notify: ->(message, callstack) {
        ActiveSupport::Notifications.instrument("deprecation.rails",
                                                :message => message, :callstack => callstack)
      },

      silence: ->(message, callstack) {},
    }

    # Behavior module allows to determine how to display deprecation messages.
    # You can create a custom behavior or set any from the +DEFAULT_BEHAVIORS+
    # constant. Available behaviors are:
    #
    # [+raise+]   Raise <tt>ActiveSupport::DeprecationException</tt>.
    # [+stderr+]  Log all deprecation warnings to +$stderr+.
    # [+log+]     Log all deprecation warnings to +Rails.logger+.
    # [+notify+]  Use +ActiveSupport::Notifications+ to notify +deprecation.rails+.
    # [+silence+] Do nothing.
    #
    # Setting behaviors only affects deprecations that happen after boot time.
    # For more information you can read the documentation of the +behavior=+ method.
    module Behavior
      # Whether to print a backtrace along with the warning.
      attr_accessor :debug

      # Returns the current behavior or if one isn't set, defaults to +:stderr+.
      def behavior
        @behavior ||= [DEFAULT_BEHAVIORS[:stderr]]
      end

      # Sets the behavior to the specified value. Can be a single value, array,
      # or an object that responds to +call+.
      #
      # Available behaviors:
      #
      # [+raise+]   Raise <tt>ActiveSupport::DeprecationException</tt>.
      # [+stderr+]  Log all deprecation warnings to +$stderr+.
      # [+log+]     Log all deprecation warnings to +Rails.logger+.
      # [+notify+]  Use +ActiveSupport::Notifications+ to notify +deprecation.rails+.
      # [+silence+] Do nothing.
      #
      # Setting behaviors only affects deprecations that happen after boot time.
      # Deprecation warnings raised by gems are not affected by this setting
      # because they happen before Rails boots up.
      #
      #   ActiveSupport::Deprecation.behavior = :stderr
      #   ActiveSupport::Deprecation.behavior = [:stderr, :log]
      #   ActiveSupport::Deprecation.behavior = MyCustomHandler
      #   ActiveSupport::Deprecation.behavior = ->(message, callstack) {
      #     # custom stuff
      #   }
      # 
      # StandardErrorのエラー挙動をカスタマイズできる
      def behavior=(behavior)
        @behavior = Array(behavior).map { |b| DEFAULT_BEHAVIORS[b] || b }
      end
    end
  end
end
```

active_support/deprecation/reporting
警告文の生成などを行う

```rb
require 'rbconfig'

module ActiveSupport
  class Deprecation
    module Reporting
      # Whether to print a message (silent mode)
      attr_accessor :silenced
      # Name of gem where method is deprecated
      attr_accessor :gem_name

      # Outputs a deprecation warning to the output configured by
      # <tt>ActiveSupport::Deprecation.behavior</tt>.
      #
      #   ActiveSupport::Deprecation.warn('something broke!')
      #   # => "DEPRECATION WARNING: something broke! (called from your_code.rb:1)"
      # 
      # 警告メッセージを表示するためのインターフェイス
      def warn(message = nil, callstack = nil)
        # サイレントモード
        return if silenced

        callstack ||= caller_locations(2)
        deprecation_message(callstack, message).tap do |m|
          behavior.each { |b| b.call(m, callstack) }
        end
      end

      # Silence deprecation warnings within the block.
      #
      #   ActiveSupport::Deprecation.warn('something broke!')
      #   # => "DEPRECATION WARNING: something broke! (called from your_code.rb:1)"
      #
      #   ActiveSupport::Deprecation.silence do
      #     ActiveSupport::Deprecation.warn('something broke!')
      #   end
      #   # => nil
      def silence
        old_silenced, @silenced = @silenced, true
        yield
      ensure
        @silenced = old_silenced
      end

      # 対象のメソッドを決めて警告文を表示できる
      def deprecation_warning(deprecated_method_name, message = nil, caller_backtrace = nil)
        caller_backtrace ||= caller_locations(2)
        deprecated_method_warning(deprecated_method_name, message).tap do |msg|
          warn(msg, caller_backtrace)
        end
      end

      private
        # Outputs a deprecation warning message
        #
        #   ActiveSupport::Deprecation.deprecated_method_warning(:method_name)
        #   # => "method_name is deprecated and will be removed from Rails #{deprecation_horizon}"
        #   ActiveSupport::Deprecation.deprecated_method_warning(:method_name, :another_method)
        #   # => "method_name is deprecated and will be removed from Rails #{deprecation_horizon} (use another_method instead)"
        #   ActiveSupport::Deprecation.deprecated_method_warning(:method_name, "Optional message")
        #   # => "method_name is deprecated and will be removed from Rails #{deprecation_horizon} (Optional message)"
        #
        # 対象のメソッドに対しての警告文の生成
        def deprecated_method_warning(method_name, message = nil)
          warning = "#{method_name} is deprecated and will be removed from #{gem_name} #{deprecation_horizon}"
          case message
            when Symbol then "#{warning} (use #{message} instead)"
            when String then "#{warning} (#{message})"
            else warning
          end
        end

        # 警告文の生成
        def deprecation_message(callstack, message = nil)
          message ||= "You are using deprecated behavior which will be removed from the next major or minor release."
          "DEPRECATION WARNING: #{message} #{deprecation_caller_message(callstack)}"
        end

        # 対象の位置
        def deprecation_caller_message(callstack)
          file, line, method = extract_callstack(callstack)
          if file
            if line && method
              "(called from #{method} at #{file}:#{line})"
            else
              "(called from #{file}:#{line})"
            end
          end
        end

        # callstackからファイル、行、メソッドの情報を抜き取る
        def extract_callstack(callstack)
          return _extract_callstack(callstack) if callstack.first.is_a? String

          offending_line = callstack.find { |frame|
            frame.absolute_path && !ignored_callstack(frame.absolute_path)
          } || callstack.first

          [offending_line.path, offending_line.lineno, offending_line.label]
        end

        def _extract_callstack(callstack)
          warn "Please pass `caller_locations` to the deprecation API" if $VERBOSE
          offending_line = callstack.find { |line| !ignored_callstack(line) } || callstack.first

          if offending_line
            if md = offending_line.match(/^(.+?):(\d+)(?::in `(.*?)')?/)
              md.captures
            else
              offending_line
            end
          end
        end

        RAILS_GEM_ROOT = File.expand_path("../../../../..", __FILE__) + "/"

        def ignored_callstack(path)
          path.start_with?(RAILS_GEM_ROOT) || path.start_with?(RbConfig::CONFIG['rubylibdir'])
        end
    end
  end
end
```

active_support/deprecation/method_wrappers
特定のModule内に存在するメソッドに対して警告文を表示するためのモジュール

```rb
require 'active_support/core_ext/module/aliasing'
require 'active_support/core_ext/array/extract_options'

module ActiveSupport
  class Deprecation
    module MethodWrapper
      # Declare that a method has been deprecated.
      #
      #   module Fred
      #     extend self
      #
      #     def aaa; end
      #     def bbb; end
      #     def ccc; end
      #     def ddd; end
      #     def eee; end
      #   end
      #
      # Using the default deprecator:
      #   ActiveSupport::Deprecation.deprecate_methods(Fred, :aaa, bbb: :zzz, ccc: 'use Bar#ccc instead')
      #   # => [:aaa, :bbb, :ccc]
      #
      #   Fred.aaa
      #   # DEPRECATION WARNING: aaa is deprecated and will be removed from Rails 5.1. (called from irb_binding at (irb):10)
      #   # => nil
      #
      #   Fred.bbb
      #   # DEPRECATION WARNING: bbb is deprecated and will be removed from Rails 5.1 (use zzz instead). (called from irb_binding at (irb):11)
      #   # => nil
      #
      #   Fred.ccc
      #   # DEPRECATION WARNING: ccc is deprecated and will be removed from Rails 5.1 (use Bar#ccc instead). (called from irb_binding at (irb):12)
      #   # => nil
      #
      # Passing in a custom deprecator:
      #   custom_deprecator = ActiveSupport::Deprecation.new('next-release', 'MyGem')
      #   ActiveSupport::Deprecation.deprecate_methods(Fred, ddd: :zzz, deprecator: custom_deprecator)
      #   # => [:ddd]
      #
      #   Fred.ddd
      #   DEPRECATION WARNING: ddd is deprecated and will be removed from MyGem next-release (use zzz instead). (called from irb_binding at (irb):15)
      #   # => nil
      #
      # Using a custom deprecator directly:
      #   custom_deprecator = ActiveSupport::Deprecation.new('next-release', 'MyGem')
      #   custom_deprecator.deprecate_methods(Fred, eee: :zzz)
      #   # => [:eee]
      #
      #   Fred.eee
      #   DEPRECATION WARNING: eee is deprecated and will be removed from MyGem next-release (use zzz instead). (called from irb_binding at (irb):18)
      #   # => nil
      # 対象のモジュールとメソッドを決めて警告文を表示する
      def deprecate_methods(target_module, *method_names)
        # 可変長引数からハッシュ形式になっている対象の値を取得する
        options = method_names.extract_options!
        # カスタムの警告文があればそれを利用
        deprecator = options.delete(:deprecator) || self
        # 対象のメソッド名を取得
        method_names += options.keys

        # 無名モジュール？
        mod = Module.new do
          method_names.each do |method_name|
            # 対象のメソッドに対して警告文を表示する関数を生成 
            # 対象のメソッドを上書きしている
            define_method(method_name) do |*args, &block|
              deprecator.deprecation_warning(method_name, options[method_name])
              # 元メソッド
              super(*args, &block)
            end
          end
        end

        # 対象のモジュールの下に入れる
        target_module.prepend(mod)
      end
    end
  end
end
```

active_support/deprecation/proxy_wrappers

```rb
require 'active_support/inflector/methods'

module ActiveSupport
  class Deprecation
    # 以下の３つのクラスのスーパークラスとなる
    # - DeprecatedObjectProxy
    # - DeprecatedInstanceVariableProxy
    # - DeprecatedConstantProxy
    class DeprecationProxy #:nodoc:
      def self.new(*args, &block)
        object = args.first

        # 引数にオブジェクトが設定されていることを確認
        return object unless object
        super
      end

      # 正規表現に一致しない場合はそのメソッドを未定義にする
      instance_methods.each { |m| undef_method m unless m =~ /^__|^object_id$/ }

      # Don't give a deprecation warning on inspect since test/unit and error
      # logs rely on it for diagnostics.
      def inspect
        target.inspect
      end

      private
        def method_missing(called, *args, &block)
          # 警告文を呼び出して元のメソッドを呼び出している
          warn caller_locations, called, args
          target.__send__(called, *args, &block)
        end
    end

    # DeprecatedObjectProxy transforms an object into a deprecated one. It
    # takes an object, a deprecation message and optionally a deprecator. The
    # deprecator defaults to +ActiveSupport::Deprecator+ if none is specified.
    #
    #   deprecated_object = ActiveSupport::Deprecation::DeprecatedObjectProxy.new(Object.new, "This object is now deprecated")
    #   # => #<Object:0x007fb9b34c34b0>
    #
    #   deprecated_object.to_s
    #   DEPRECATION WARNING: This object is now deprecated.
    #   (Backtrace)
    #   # => "#<Object:0x007fb9b34c34b0>"
    class DeprecatedObjectProxy < DeprecationProxy
      def initialize(object, message, deprecator = ActiveSupport::Deprecation.instance)
        @object = object
        @message = message
        @deprecator = deprecator
      end

      private
        def target
          @object
        end

        def warn(callstack, called, args)
          @deprecator.warn(@message, callstack)
        end
    end

    # DeprecatedInstanceVariableProxy transforms an instance variable into a
    # deprecated one. It takes an instance of a class, a method on that class
    # and an instance variable. It optionally takes a deprecator as the last
    # argument. The deprecator defaults to +ActiveSupport::Deprecator+ if none
    # is specified.
    #
    #   class Example
    #     def initialize
    #       @request = ActiveSupport::Deprecation::DeprecatedInstanceVariableProxy.new(self, :request, :@request)
    #       @_request = :special_request
    #     end
    #
    #     def request
    #       @_request
    #     end
    #
    #     def old_request
    #       @request
    #     end
    #   end
    #
    #   example = Example.new
    #   # => #<Example:0x007fb9b31090b8 @_request=:special_request, @request=:special_request>
    #
    #   example.old_request.to_s
    #   # => DEPRECATION WARNING: @request is deprecated! Call request.to_s instead of
    #      @request.to_s
    #      (Backtrace information…)
    #      "special_request"
    #
    #   example.request.to_s
    #   # => "special_request"
    class DeprecatedInstanceVariableProxy < DeprecationProxy
      def initialize(instance, method, var = "@#{method}", deprecator = ActiveSupport::Deprecation.instance)
        @instance = instance
        @method = method
        @var = var
        @deprecator = deprecator
      end

      private
        def target
          @instance.__send__(@method)
        end

        def warn(callstack, called, args)
          @deprecator.warn("#{@var} is deprecated! Call #{@method}.#{called} instead of #{@var}.#{called}. Args: #{args.inspect}", callstack)
        end
    end

    # DeprecatedConstantProxy transforms a constant into a deprecated one. It
    # takes the names of an old (deprecated) constant and of a new constant
    # (both in string form) and optionally a deprecator. The deprecator defaults
    # to +ActiveSupport::Deprecator+ if none is specified. The deprecated constant
    # now returns the value of the new one.
    #
    #   PLANETS = %w(mercury venus earth mars jupiter saturn uranus neptune pluto)
    #
    #   (In a later update, the original implementation of `PLANETS` has been removed.)
    #
    #   PLANETS_POST_2006 = %w(mercury venus earth mars jupiter saturn uranus neptune)
    #   PLANETS = ActiveSupport::Deprecation::DeprecatedConstantProxy.new('PLANETS', 'PLANETS_POST_2006')
    #
    #   PLANETS.map { |planet| planet.capitalize }
    #   # => DEPRECATION WARNING: PLANETS is deprecated! Use PLANETS_POST_2006 instead.
    #        (Backtrace information…)
    #        ["Mercury", "Venus", "Earth", "Mars", "Jupiter", "Saturn", "Uranus", "Neptune"]
    class DeprecatedConstantProxy < DeprecationProxy
      def initialize(old_const, new_const, deprecator = ActiveSupport::Deprecation.instance)
        @old_const = old_const
        @new_const = new_const
        @deprecator = deprecator
      end

      # Returns the class of the new constant.
      #
      #   PLANETS_POST_2006 = %w(mercury venus earth mars jupiter saturn uranus neptune)
      #   PLANETS = ActiveSupport::Deprecation::DeprecatedConstantProxy.new('PLANETS', 'PLANETS_POST_2006')
      #   PLANETS.class # => Array
      def class
        target.class
      end

      private
        def target
          ActiveSupport::Inflector.constantize(@new_const.to_s)
        end

        def warn(callstack, called, args)
          @deprecator.warn("#{@old_const} is deprecated! Use #{@new_const} instead.", callstack)
        end
    end
  end
end
```