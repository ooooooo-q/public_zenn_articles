---
title: "const_get, constantize"
---

### [Module#const_get](https://docs.ruby-lang.org/ja/latest/method/Module/i/const_get.html)

変数の値で定数を1つだけ取得します。第２引数のinheritを指定することで対象となるスコープが変化します。（rubyのバージョンによって挙動が少し異なります。）

```ruby:const_get.rb
module Bar
  BAR = 1
end

module BAR
  BAR = 2
end

class Baz
  BAR = 3
end

p Baz.const_get(:BAR)   
# => 3

p Baz.const_get('BAR')   
# => 3

p Baz.const_get("::BAR")
# => BAR

p Baz.const_get("::BAR", false)
# => BAR

p Baz.const_get("::BAR::BAR", false)
# => 2
```

inheritをfalseにしても、`::` から始まるパスを指定すれば取得できます。 `Module#const_defined?` に関しても同じ挙動です。
念の為、inheritをfalseにした上で、nameをシンボルにすれば回避できます。

```ruby:const_get.rb
p Baz.const_get("BAR".to_sym)
# => 3

p Baz.const_get("Bar".to_sym)
# => Bar

p Baz.const_get("::BAR".to_sym)
# =>  wrong constant name ::BAR (NameError)
```

呼び出せるのは1つの定数だけで内部状態も指定できないので、ユーザ入力が渡されてもMarshalのようなGadge chainにはなりません。しかし、意図しない定数が呼び出された上でユーザからの入力を引数としたメソッドが呼ばれた場合、前後のコードによっては危険なこともあります。


#### 過去の事例

https://hackerone.com/reports/1125425
https://snyk.io/vuln/SNYK-RUBY-KRAMDOWN-1087436

Gitlabで利用されていたKramdownでは `const_get` で `inherit = false` が設定されていなかったため、想定外のクラスを操作できました。
Gitlabで読み込まれていたRedisのクラスを組み合わせることでRCEが可能な事例でした。


### [constantize](https://api.rubyonrails.org/classes/String.html#method-i-constantize)

ActiveSupportを読み込むと `"文字列".constantize` で文字列に該当する定数がよびだせます。

```ruby:constantize.rb
require 'active_support/all'

p "Module".constantize
# => Module

p "Logger".constantize
# => Logger
```

Rails 6までは `Object.const_get` をwrapしていますが(https://github.com/rails/rails/blob/v6.1.4.1/activesupport/lib/active_support/inflector/methods.rb#L272)、Rails 7からはrubyの `Object.const_get` を呼び出すだけになっています。(https://github.com/rails/rails/blob/v7.0.0.alpha2/activesupport/lib/active_support/inflector/methods.rb#L279)

`safe_constantize` のメソッドもありますが、このsafeは呼び出した定数が存在しなかった場合に例外が発生しなくなることの意味であって、ユーザの入力を渡した場合に危険性があることは変わりません。(https://github.com/rails/rails/blob/v6.1.4.1/activesupport/lib/active_support/inflector/methods.rb#L329)

http://gavinmiller.io/2016/the-safesty-way-to-constantize/ 
危険性については上記の記事による解説が詳しく、他にもクラスやファイルの存在確認、DoSの手法が紹介されています。

ところで記事では `Logger.new(..)` を使ったRCEが書かれていますが、これはRuby2.6以降の環境では動きません。

Logger(とLoggerを継承したActiveSupport::Logger)にnewで与えた引数は、最終的に `File.open(filename)` へ渡されています。

```ruby
# https://github.com/ruby/logger/blob/v1.4.3/lib/logger/log_device.rb#L93
def open_logfile(filename)
  begin
    File.open(filename, (File::WRONLY | File::APPEND))
  rescue Errno::ENOENT
    create_logfile(filename)
  end
end
```

以前のRubyではFileクラスの引数へ `|` が先頭である場合にコマンドが実行されていましが、Ruby2.6からはその処理がなくなり安全になりました。(https://github.com/ruby/ruby/blob/v3_0_0/doc/NEWS-2.6.0#compatibility-issues-excluding-feature-bug-fixes-)

> File.read, File.binread, File.write, File.binwrite, File.foreach, and File.readlines do not invoke external commands even if the path starts with the pipe character '|'.. [Feature #14245]

とはいえRailsアプリケーションに含まれるすべてのクラスで危険性がないのかというとはっきりとはわかりません。Loggerについてもコマンドが実行できないだけで、任意のパスにファイルを作ることができるのでServer Side Template Injectionを使ったRCEの危険性も残っています。

