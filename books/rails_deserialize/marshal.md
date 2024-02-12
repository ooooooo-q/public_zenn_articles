---
title: "Marshal.load"
---

### [Marshal.load](https://docs.ruby-lang.org/ja/latest/class/Marshal.html)

RubyのMarshalは任意のオブジェクトを文字列にすること（シリアライズ）、そこからまた復元すること（デシリアライズ）ができます。

```ruby:marshal.rb
class Cat
   attr_reader :color

   def initialize(color)
     @color = color
   end
end

cat = Cat.new('white')
p cat
# => #<Cat:0x00007fb97f80f120 @color="white">

# シリアライズ
dump_str = Marshal.dump(cat)
p dump_str
# => "\x04\bo:\bCat\x06:\v@colorI\"\nwhite\x06:\x06ET"

# デシリアライズ
cat_loaded = Marshal.load(dump_str)
p cat_loaded
# => #<Cat:0x00007fb97f80ea90 @color="white">
p cat_loaded.color
# => "white"
```

Marshal.loadの引数としてユーザ由来の値が渡されると、任意のインスンタンスのオブジェクトを作ることができてしまうので危険な状態になります。脆弱性の種類としてはObject injection、Insecure Deserializationなどと呼ばれます。

オブジェクトインジェクションの危険性は定義されているクラスや、デシリアライズ後のオブジェクトへの操作によって変わります。特にRails(ActiveSupport)ではデシリアライズすることで、デシリアライズされたオブジェクトの任意のメソッドを呼べるクラスが存在するため、RCEが容易でした。また後の章で紹介するGadget chainによりほとんどのRubyの環境ではRCEが可能です。

Marshal.loadが実行されたときは各クラスの `#initialize` ではなく、各クラスがallocateで作られあとにinstance_variable_setを使って値がセットされたのに近い状態になります。またはクラスに `mardhal_load/marshal_dump` が定義されている場合はこちらが呼ばれます。method_missingは定義されていても利用されません。

```ruby:marshal_load.rb
class Cat
   attr_reader :color

   def marshal_dump
    puts "call marshal_dump"
    @color
   end

   def marshal_load(a)
    puts "call marshal_load"
   end
end

cat = Cat.new

# marshal_dumpが実行される
dump_str = Marshal.dump(cat)

# marshal_loadが実行される
cat_loaded = Marshal.load(dump_str)
```

```
$ ruby marshal_load.rb

call marshal_dump
call marshal_load
```

クラスの `#initialize` やsetterのメソッドが実行されるわけではないため、デシリアライズ後のオブジェクトを通常ならばありえない状態にできます。
一方で、IOなどシステムがオブジェクトの状態を保持するクラスを対象としたシリアライズ/デシリアライズはできないようになっています。 (https://docs.ruby-lang.org/ja/latest/method/Marshal/m/dump.html)

```ruby:marshal_tail.rb
class Cat
   attr_reader :color

   def initialize(color)
     @color = color
   end

end

str = "\x04\bo:\bCat\x06:\n@tailI\"\nwhite\x06:\x06ET"
cat_loaded = Marshal.load(str)

# 通常は設定できないインスタンス変数に値が設定されている
puts cat_loaded.instance_variable_get(:@tail)
# => white
```

:::message
Marshalでシリアライズしたデータは、データフォーマットのメジャーバージョンが上がったりRailsのアップグレードにより使っていたクラスがなくなったりすると復元できなくなる場合もあるため扱いに注意が必要です。
:::


### Cookie Store (Rails)

以前のCookie StoreではデフォルトのシリアライザとしてMarshalが使われていました。そのため、`secret_key_base` の値が漏洩した場合には細工したcookieが送信されるとRCEが可能でした。現在はシリアライザをJSONに切り替えるconfigがありますので、Marshalのままになっている場合は切り替えましょう。([Rails アップグレードガイド -8.5 Cookiesシリアライザ](https://railsguides.jp/upgrading_ruby_on_rails.html#cookies%E3%82%B7%E3%83%AA%E3%82%A2%E3%83%A9%E3%82%A4%E3%82%B6))

Active StorageのURLとCookie Storeとの違いとしてシリアライザに切り替えるconfigがないことと、HTTPでGETのURLが送信されれば攻撃可能であるという点があります。

cookieによる攻撃はサーバへ送信されたcookieの値までログを残す設定としているのは少ないため攻撃されたことに気づきにくいですが、GETのURLからの攻撃であればアクセスログなどに残っているので攻撃されたかどうかは判断しやすそうです。一方、このGETリクエストではヘッダへの細工もなく必要なのはURLだけであるため、インターネットに公開されていないサーバに対しても攻撃が可能です。サーバと同じネットワーク内のブラウザなどから直接リクエストが送信されれば良いので、ユーザに対してCSRFを仕掛ける、メールを送って細工した画像のURLをメーラーで閲覧させるなど経路は色々考えられます。


#### [Rack::Session::Cookie](https://github.com/rack/rack/blob/2.2.3/lib/rack/session/cookie.rb)

Rails側のCookieStoreとは違い `Rack::Session::Cookie` のシリアライザはデフォルトではMarshalが使用されています(https://github.com/rack/rack/blob/2.2.3/lib/rack/session/cookie.rb#L120)。

Railsと同じように設定でsecretが渡され、デシリアライズされる前に検証がなされています(https://github.com/rack/rack/blob/2.2.3/lib/rack/session/cookie.rb#L140)。
secretを指定しない状態でも起動できるのですが以下のように起動時に警告が表示されます。

```ruby:config.ru
require 'delegate'

class SessionCookieTest
  def call(env)
    [ 200,
      {},
      ["session cookie test"]
    ]
  end
end

use Rack::Session::Cookie
run SessionCookieTest.new
```

```
❯ RACK_ENV=production rackup
        SECURITY WARNING: No secret option provided to Rack::Session::Cookie.
        This poses a security threat. It is strongly recommended that you
        provide a secret to prevent exploits that may be possible from crafted
        cookies. This will not be supported in future versions of Rack, and
        future versions will even invalidate your existing user cookies.
```

[Sinatra](http://sinatrarb.com/)ではデフォルトのセッションに `Rack::Session::Cookie` が使用されており、secretが漏洩した場合はGadget ChainによりRCEが可能ですが、デフォルトでは起動時にランダムな値が設定されています。([Sinatra : セッションの使用](https://github.com/sinatra/sinatra/blob/v2.1.0/README.ja.md#%E3%82%BB%E3%83%83%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%AE%E4%BD%BF%E7%94%A8))


[Hanami](https://hanamirb.org/)でも同様にデフォルトのセッションに `Rack::Session::Cookie` が使用されており、こちらは環境変数でsecretの値を設定していました([Hanami : Enable Sessions](https://guides.hanamirb.org/actions/sessions/#enable-sessions))。v1.3.5のリリースでリアライザにはJSONが使われるように修正されました。
([Announcing Hanami v1.3.5](https://hanamirb.org/blog/2021/10/18/announcing-hanami-135/))


### Marshal.loadの利用箇所

他にもMarshal.loadは様々な箇所で利用されています。
通常であればユーザ入力が直接入るわけでなければ問題ないのですが、後述する他の脆弱性にによって想定外の値が入ってきた場合はRCEが起こりえます。

