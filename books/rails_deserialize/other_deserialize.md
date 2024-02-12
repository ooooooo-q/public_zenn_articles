---
title: "その他のデシリアライズ"
---


### [msgpack-ruby]( https://github.com/msgpack/msgpack-ruby)

[MessagePack](https://msgpack.org/mesasge)のruby実装である `msgpack-ruby` では、基本のクラス以外にも事前に登録されたクラスであればデシリアライズが可能です。(https://github.com/msgpack/msgpack/blob/master/spec.md#extension-types)

```ruby:msgpack.rb
require 'msgpack'

class Cat

  def initialize(color)
    @color = color
  end

  def to_msgpack_ext
    @color
  end
  
  def self.from_msgpack_ext(value)
    new(value)
  end
end

MessagePack::DefaultFactory.register_type(0x01, Cat) 

cat = Cat.new('black')
msg = MessagePack.pack(cat)
pp msg
# "\xC7\x05\x01black"

pp MessagePack.unpack(msg) 
# #<Cat:0x000000014219bbd8 @color="black">
```


### デストラクタ

PHPのデシリアライズについて書かれた[安全でないデシリアライゼーション(Insecure Deserialization)入門](https://blog.tokumaru.org/2017/09/introduction-to-object-injection.html)では `__destruct()` を使った事例が紹介されていますが、Rubyでは該当するものがなさそうです。
[ObjectSpace.#define_finalizer](https://docs.ruby-lang.org/ja/latest/method/ObjectSpace/m/define_finalizer.html)が近いですが、Tempfileなどでの実装を見る限りinitializeのなかで設定されるため、Marshalを使ったデシリアライズでは動作しないようにみえます。