---
title: "対策"
---

### `secret_key_base` や `Marshal` などが使われている場所を把握する

CVE-2019-5420の説明にはActive Storageの話が出てこないので、Cookie Storeだけの問題だと思われていそうな記事を何度か見かけました（Cookie storeではなくdeviseを使えばよいなど）。現在のRailsのサイトではCookieStoreでの話が中心に記述されているため、secret_key_baseが他のところで使われていることに気づかれにくそうではあります。

`MessageVerifier` と `MessageEncryptor` のデフォルトのシリアライザは互換性のためにMarshalから変更されていないため、Railsでなにか機能が増えるたびに同じ問題が発生しないか気になるところですが、Rais6.1で導入される `ActiveRecord::SignedId` では[シリアライザにJSONが指定されていました](https://github.com/rails/rails/blob/v6.1.4.1/activerecord/lib/active_record/signed_id.rb#L79)。


### gemの実装を確認する

例としてSigned Global IDはAction Textで使われています。（[Rails 6: Action Textのファイルアップロードを分解調査する（翻訳）](https://techracho.bpsinc.jp/hachi8833/2019_09_03/79149])）。

他にもgemの中で使っている可能性はあるため、実際にどのような影響があるかはアプリケーション次第になります。Marshal.loadなどの動作を上書きすることでもある程度見つけられそうですが、通常使われない機能などでは難しいですね。できるだけgemのコードの中身も把握することが安全につながりますが、なかなか大変なことでもあります。

jobを扱うgemでは、jobの引数をDBのレコードにYAMLでシリアライズして保存するものもいくつかあるようです。任意のテーブルにinsertするSQL injectionが起こせた場合は、そこからRCEにつながる可能性もあります。

また、いくつかのgemではデシリアライズでクラスを復元できることが仕様であることもあります(Oj、Plistなど)。その場合は脆弱性としては認識されないので、使用者が十分に気をつける必要があります。


### シリアライザを安全なものに切り替えていく

キャッシュや設定ファイル以外の用途であれば、大体の場合はシリアライザをJSONに切り替えることができるのではないでしょうか。逆に切り替えて問題がある場合はRubyやRailsのバージョンアップでクラスの復元に失敗する可能性があります。さらにJSON.loadは使わずにJSON.parseで統一しましょう。YAMLであればYAML.safe_loadを使う、またはPsychのバージョンを4系に切り替えることです。

もしMarshalから切り替えるのが難しい場合、以下のようにデシリアライズするクラスを制限することで対策可能かもしれませんが、十分検証されているわけではありません。この方法ではエラーとなるのは許可されていないクラスが1つデシリアライズされた後なので、クラス生成時の挙動などは完全には防ぐことができません。

```ruby
# https://github.com/ooooooo-q/marshal.safe_load/blob/main/safe_load.rb
module Marshal

  def self.safe_load(str, permitted_classes: [])
    permitted_classes = permitted_classes + [String, Hash, FalseClass, TrueClass, Symbol, Numeric, NilClass, Array]

    f = proc { |obj|
      unless permitted_classes.any? {|c| obj.is_a?(c) }
        raise ::TypeError, "unexepected type #{obj.class}" 
      end
      obj
    }

    self.load(str, f)
  end

end
```

```ruby
❯ irb
irb(main):001:0> require './safe_load.rb'
=> true
irb(main):002:0> Marshal.safe_load(Marshal.dump({cat: "black"}))
=> {:cat=>"black"}
irb(main):003:0>  Marshal.load(Marshal.dump(Module))
=> Module
irb(main):004:0> Marshal.safe_load(Marshal.dump(Module))
...
StandardError (unexepected type Class)
```

ActiveStorageのシリアライザの切り替えは、公式で案内されている方法がありません。試しにJSONへ切り替える実装を作ってみましたが多少癖があり、`secret_key_base` が漏洩した場合にはシリアライザ以外の部分にも問題が出るようでしたので、それだけでは不十分なようです。


### `secret_key_base` などの鍵が漏洩したときに、鍵のrotationが直ぐにできる実装にする

秘密だったはずのものが秘密でなくなる、ということはよくある話ですので、鍵の交換ができるような実装へ事前にしておいたほうが良いでしょう。そうした場合、有効期限が長い署名を使っていると対応が難しくなります。

例としては `Rack::Session::Cookie` ではsecretだけではなく、old_secretも指定可能です。
認証時にsecretでの検証で失敗すると[old_secretでも検証を試します](https://github.com/rack/rack/blob/2.2.3/lib/rack/session/cookie.rb#L187)。cookieを発行する場合はsecretだけが用いられます。
