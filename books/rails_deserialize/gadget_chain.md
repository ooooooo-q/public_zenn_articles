---
title: "Gadget Chain"
---

Marshal.loadによるオブジェクトの生成だけでは危険なコードの実行がすぐにはできないようにも見えますが、いくつかのクラスやオブジェクトの生成をつなげていくことでRCEが可能になります。これはGadgetまたはGadget Chainと呼ばれているようです。


### Ruby on Rails での Gadget Chain

[Rails 3.2.10 Remote Code Execution](https://github.com/charliesome/charlie.bz/blob/master/posts/rails-3.2.10-remote-code-execution.md) で言及されているDeprecationProxyを使って説明します。

```ruby
# https://github.com/rails/rails/blob/v6.1.4.1/activesupport/lib/active_support/deprecation/proxy_wrappers.rb#L3
module ActiveSupport
  class Deprecation
    class DeprecationProxy #:nodoc:
      
      ...

      private
        def method_missing(called, *args, &block)
          warn caller_locations, called, args
          target.__send__(called, *args, &block)
        end
    end
```

```ruby
# https://github.com/rails/rails/blob/v6.1.4.1/activesupport/lib/active_support/deprecation/proxy_wrappers.rb#L88
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
```

`ActiveSupport::Deprecation::DeprecatedInstanceVariableProxy#target` が呼ばれると `@instance.__send__(@method)` により、デシリアライズしたクラスのなかで引数なしのメソッドが自由に呼び出せます。また `DeprecationProxy` を継承しており、missing_methodでは `target` が呼び出されるため、何かのメソッドが呼ばれる時点で任意のコードを実行可能です。

```ruby:deprecation_proxy.rb
❯ bundle exec irb
irb(main):001:0> require 'active_support'
=> true

irb(main):002:0> ActiveSupport::Deprecation::DeprecatedInstanceVariableProxy.new(Kernel, 'rand')
... # DEPRECATION WARNINGが表示される。
=> 0.3367514037411955
```

#### ERBの対策

既存のPoCでは、引数がないメソッドから任意のコードを呼び出すためにERBが使われていますが、ruby2.7からはERB側に対策が入りました。デシリアライズからの操作には制限がかかり、実行時にエラーとなるようになりました(https://github.com/ruby/ruby/commit/b3507bf147ff47e331da36ba7c8e6b700c513633)。しかし、これは緩和策であり他のクラスを使うことで現在でもRCEは可能です。


```ruby:erb.rb
require 'erb'

erb = ERB.allocate

str = Marshal.dump(erb)
pp str

loaded =  Marshal.load(str)
loaded.result
```

```ruby:erb_run.rb
❯ ruby erb.rb
"\x04\bo:\bERB\x00"
.../lib/ruby/3.0.0/erb.rb:903:in `result': not initialized (ArgumentError)
  from erb.rb:13:in `<main>'
```


### Ruby での Gadget Chain

Railsを使っている場合だけが危険なのかというとそうではなく、rubygemsのクラスだけでもRCEが可能です。

2021年に公開された手法に対しては、rubygemsの3.2.25で入った修正(https://github.com/rubygems/rubygems/pull/4651)やnet-protocolの0.1.3で入った修正(https://github.com/ruby/net-protocol/pull/3)により緩和されました。

しかし、2022年春に修正の回避方法が公開されたため、Railsを使わない環境でもRCEが可能となっています。