---
title: "JSON"
---

rubyで[JSON](https://www.json.org/json-ja.html)をパースしたいときは `JSON.parse` を使えばオブジェクトインジェクションの危険はありません。

### [JSON.load](https://docs.ruby-lang.org/ja/latest/method/JSON/m/load.html)

`JSON.load` (または `JSON.restore` )は任意のクラスのオブジェクトが復元できるわけではなく `self.json_create` が宣言されたクラスのみが復元できます。これが宣言されているクラスは少ないため、MarshalやYAMLと比べると危険性は低いです。

筆者が確認した範囲では `'json'` のみがrequiresされている場合は、Stringを `.to_json_raw_object` を使って変換したものがデシリアライズ可能です。

```ruby:json_load.rb
require 'json'

puts 'abc'.to_json_raw_object
# => {"json_class"=>"String", "raw"=>[97, 98, 99]}

puts JSON.load(JSON.dump('abc'.to_json_raw_object))
# => "abc"
```

`'json/add/core'` がrequireされている場合は、DateやBigDecimalなどで記述されているクラスデシリアライズできるようになります。(https://github.com/flori/json/tree/master/lib/json/add)
Range型もデシリアライズできるので、状況によってはRegexp Injectionによる[ReDoS](https://owasp.org/www-community/attacks/Regular_expression_Denial_of_Service_-_ReDoS)の可能性もあります。

また、[Cookiejar](https://github.com/dwaite/cookiejar)など一部のgemでは `self.json_create` が宣言されていることもあるようですがかなり珍しいようです。(https://github.com/dwaite/cookiejar/blob/master/lib/cookiejar/jar.rb#L163)

一方、[JSON.parse](https://docs.ruby-lang.org/ja/latest/method/JSON/m/parse.html)であればクラスは復元されないので、通常はこちらのみを使っていればオブジェクトインジェクションの問題は発生しません。`JSON.parse` であっても `create_additions: true` を渡すとデシリアライズされますが指定されていることは極稀でしょう。

```ruby:json.rb
require 'json'

class Cat
  def self.json_create(o)
    new(o["c"])
  end

  def initialize(color)
    @color = color
  end
end

puts JSON.load('{"json_class":"Cat","c":"white"}')
# => <Cat:0x00007ffa7305a7d8>

puts JSON.parse('{"json_class":"Cat","c":"white"}')
# => {"json_class"=>"Cat", "c"=>"white"}

puts JSON.parse('{"json_class":"Cat","c":"white"}', create_additions: true)
# => <Cat:0x00007ffa7305a210>
```

#### Ruby on Railsでの拡張されるJSON (ActiveSupport)

Rails内で使われる `ActiveSupport::JSON.decode` はJSONから少し拡張されていて `ActiveSupport.parse_json_times = true` の設定がされていると、日付や時刻の形式の文字列が含まれていたときにDate型やActiveSupport::TimeWithZone型に変換されます。
(https://github.com/rails/rails/blob/v6.1.4.1/activesupport/lib/active_support/json/decoding.rb#L47)

```ruby:active_support_json.rb
require "active_support"

require "active_support/core_ext"
Time.zone_default = Time.find_zone! "Asia/Tokyo"

puts ActiveSupport::JSON::decode('"2020-02-02"').class
# => String
puts ActiveSupport::JSON::decode('"2021-02-03T11:42:33.291Z"').class
# => String

ActiveSupport.parse_json_times = true
puts ActiveSupport::JSON::decode('"2020-02-02"').class
# => Date
puts ActiveSupport::JSON::decode('"2021-02-03T11:42:33.291"').class
# => ActiveSupport::TimeWithZone
```

Cookieのシリアライザとして使われているものも `ActiveSupport::JSON` です。(https://github.com/rails/rails/blob/v6.1.4.1/actionpack/lib/action_dispatch/middleware/cookies.rb#L541)

一方、`ActiveSupport::MessageVerifier.new('dummy_secret', serializer: JSON) ` と指定した場合に使われるものは `JSON.load` です。Ruby on Rails内では `'json/add/core'` がrequireされていないのであまり気にする必要はないですが、Rails以外に使用しているgemなどで利用してこともあるので、そちらには注意が必要です。


#### 過去の事例

以前の脆弱性として `JSON.parse` であっても `JSON.load` と同じようにデシリアライズされてしまう問題がありました。[JSON におけるサービス不能攻撃および安全でないオブジェクトの生成について (CVE-2013-0269)](https://www.ruby-lang.org/ja/news/2013/02/22/json-dos-cve-2013-0269/)。
2020年春に公開された[CVE-2020-10663: JSON における安全でないオブジェクトの生成の脆弱性について（追加の修正）](https://www.ruby-lang.org/ja/news/2020/03/19/json-dos-cve-2020-10663/)では `JSON(user_input)` , `JSON.parse(user_input, nil)` , `JSON(user_input, nil)` , `JSON[user_input, nil]` の場合にデシリアライズがされていました。この形式で使われていることはあまりなさそうですが、rubyのアプリケーションではメタプログラミング経由で使われていない、呼ぶ方法がないことを確認することはなかなか難しいですし、バージョンを上げたほうが無難でしょう。本を書いた時点での最新のバージョンは2.5.1です。

また、Redisを扱うgemの[Kreids](https://github.com/rails/kredis)で、`JSON.load` を利用していたため意図せずデシリアライズが発生していました。[[CVE-2023-27531] Possible Deserialization of Untrusted Data vulnerability in Kredis JSON](https://discuss.rubyonrails.org/t/cve-2023-27531-possible-deserialization-of-untrusted-data-vulnerability-in-kredis-json/82467) (https://hackerone.com/reports/1702859)


### [Oj.load](https://github.com/ohler55/oj)

JSONを扱うgemのいくつかありますが、[Oj](http://www.ohler.com/oj/)ではRuby標準のJSONとは違い `Oj.load` では任意のクラスが復元できます。

Ojには `.parse` がなく、また `Oj.load` をオプション無しで使用した場合にシリアライズができてしまうので、コードを確認していても見逃してしまいそうではあります。
[modeの指定と対応によれば](https://github.com/ohler55/oj/blob/master/pages/Modes.md#options-matrix)、 `mode: :strict` を指定していれば安全そうです。`mode: :compat` を使用していると `JSON.load` と同じ挙動になり `self.json_create` が宣言されたクラスが復元可能です。

```ruby:oj.rb
require 'oj'

class Cat
   attr_reader :color

   def initialize(color)
     @color = color
   end

end

cat = Cat.new('white')

dump_str = Oj.dump(cat)
p dump_str
# => "{\"^o\":\"Cat\",\"color\":\"white\"}"

cat_loaded = Oj.load(dump_str)
p cat_loaded
# => #<Cat:0x00007fb33808a668 @color="white">

compat_cat_loaded = Oj.load(dump_str, mode: :compat)
p compat_cat_loaded
# => {"^o"=>"Cat", "color"=>"white"}
```

安全なパース方法として `Oj.safe_load` もあります。([Security and Optimization](https://github.com/ohler55/oj/blob/master/pages/Security.md))

```ruby:oj_safe_load.rb
safe_cat_loaded = Oj.safe_load(dump_str)
p safe_cat_loaded
# => {"^o"=>"Cat", "color"=>"white"}
```

#### 過去の事例

https://honoki.net/2019/03/18/rce-in-slanger-0-6-0/

WebSocketを扱うPusherのRuby実装、[Slanger](https://github.com/stevegraham/slanger)での事例です。
クライアントから送信されたJSONのパースに `Oj.load` が使用されていたためRCEが可能でした。(https://github.com/stevegraham/slanger/pull/238/commits/5267b455caeb2e055cccf0d2b6a22727c111f5c3)


#### [MultiJSON](https://github.com/intridea/multi_json)

複数のJSONパーサをラップするMultiJSONの中でもOjが利用可能ですが、
load時の指定として `:mode => :strict` が指定されるので危険はありません。
(https://github.com/intridea/multi_json/blob/master/lib/multi_json/adapters/oj.rb#L9)


### YAMLを使ったJSONのパース

YAMLの仕様はJSONのsubsetとされているため、YAMLのパーサを使ってJSONのパースができます。(https://yaml.org/spec/1.2/spec.html#id2759572) しかし、現在はjsonのgemがあるので使うことはあまりないでしょう。

```ruby:yaml_json.rb
require 'yaml'
require 'json'

json_str = JSON.dump({cat: [1, 2, nil]})
puts json_str
# => {"cat":[1,2,null]}

puts YAML.load(json_str)
# => {"cat"=>[1, 2, nil]}
```


#### 過去の事例

Rails 3.0.19のバージョンまではjson形式のリクエストボディのパースに `YAML.load` が使われていました。しかし、前述のYAMLの章で記述したように `YAML.load` にリクエスト内容が入るとRCEが可能となるため大変危険な状態でした。3.0.20のバージョンでOkJsonを使うように修正されています。(CVE-2013-0333: [\[SEC\]\[ANN\] Rails 3.0.20, and 2.3.16 have been released!](https://weblog.rubyonrails.org/2013/1/28/Rails-3-0-20-and-2-3-16-have-been-released/))

現在でもcrackのコードでは `YAML.safe_load` を使ってJSONのパースが行われています。(https://github.com/jnunemaker/crack/blob/v0.4.4/lib/crack/json.rb#L14)





