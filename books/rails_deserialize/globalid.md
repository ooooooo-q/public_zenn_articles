---
title: "GlobalID"
---


### [GlobalID::Locator.locate](https://github.com/rails/globalid)

GlobalIDはRailsのモデルをURIで表現できるgemです。Rails内ではActive jobに利用できる引数の1つとして利用されています。([Active Job Basics - 9.1 GlobalID](https://guides.rubyonrails.org/active_job_basics.html#globalid))

```ruby:locate.rb
require 'globalid'

GlobalID.app = "cat"

class Cat
  include GlobalID::Identification
  attr_reader :id

  def initialize(id = 1)
    @id = id
  end

  def self.find(id)
    new(id)
  end
end

cat = Cat.new(1)
cat_gid = cat.to_global_id

p cat_gid
# => #<GlobalID:0x00007fb4c8967ec0 @uri=#<URI::GID gid://cat/Cat/1>>
p cat_gid.to_s
# => "gid://cat/Cat/1"

# Catが復元される
p GlobalID::Locator.locate cat_gid.to_s
# => #<Cat:0x00007fb4c51d4800 @id="1">
```

URIの構成としては `gid://アプリ名/モデル名/ID` です。
`GlobalID::Locator.locate(gid)` が実行されると内部的には `モデル名.find(ID)` が呼ばれます。おそらくActiveRecordのモデルを対象としたものです。URI.parseでも扱えますが、open-uriによる `URI.pasre(...).read` などは利用できません。

モデルの取得には `constantize` が使われています。

```ruby
# https://github.com/rails/globalid/blob/v0.5.2/lib/global_id/global_id.rb#L59
  def model_class
    model_name.constantize
  end
```

```ruby
# https://github.com/rails/globalid/blob/v0.5.2/lib/global_id/locator.rb#L129
  def locate(gid)
    gid.model_class.find gid.model_id
  end
```

ActiveRecord以外のクラスであっても、`self.find` で引数を1つ受け取るものか、`self.method_missing` が定義されているものであれば呼び出すことができます。

Enumerableを継承していると[Enumerable#find](https://docs.ruby-lang.org/ja/latest/method/Enumerable/i/detect.html)を呼び出せるクラスもあります。しかし、これはprocとして渡されることが期待されており、gidから文字列のmodel_idを渡されるとエラーになります。
他に `self.find(...)` のメソッドが使えるものとして
[Encoding.find](https://docs.ruby-lang.org/ja/latest/method/Encoding/s/find.html)と[Find.find](https://docs.ruby-lang.org/ja/latest/method/Find/m/find.html)がありましたが、これらはRCEにはつながらなそうです。

```ruby:encoding.rb
require 'globalid'

gid = URI.parse("gid://bcx/Encoding/utf-8")
p GlobalID::Locator.locate gid
# => #<Encoding:UTF-8>

p GlobalID::Locator.locate "gid://bcx/Encoding/utf-8"
# => #<Encoding:UTF-8>
```

ActiveRecord以外のクラスに危険なものとして、SignedGlobalIDがあります。これについては後述します。


#### 過去の事例

過去の脆弱性として、ユーザ入力をActiveJobの引数として直接渡しているコードが存在した場合に、意図せずモデルが復元されてしまうという問題がありました。
[\[CVE-2018-16476\] Broken Access Control vulnerability in Active Job](https://groups.google.com/g/rubyonrails-security/c/FL4dSdzr2zw/m/zjKVhF4qBAAJ)
現在はブロックする処理が入っています。(https://github.com/rails/rails/blob/v6.1.4.1/activejob/lib/active_job/arguments.rb#L180)


### GraphQLでの利用

GraphQLではモデルの復元に利用されることもあるようです。(https://graphql-ruby.org/schema/object_identification.html)

その際にはGitlabのように想定外のクラスが復元できない制限を設けたほうがよいでしょう。

```ruby:gitlab_schema.rb
# https://gitlab.com/gitlab-org/gitlab/-/blob/v14.3.0-ee/app/graphql/gitlab_schema.rb#L114
  def parse_gid(global_id, ctx = {})
    expected_types = Array(ctx[:expected_type])
    gid = GlobalID.parse(global_id)

    raise Gitlab::Graphql::Errors::ArgumentError, "#{global_id} is not a valid GitLab ID." unless gid

    if expected_types.any? && expected_types.none? { |type| gid.model_class.ancestors.include?(type) }
      vars = { global_id: global_id, expected_types: expected_types.join(', ') }
      msg = _('%{global_id} is not a valid ID for %{expected_types}.') % vars
      raise Gitlab::Graphql::Errors::ArgumentError, msg
    end

    gid
  end
```


### Signed GlobalID

Signed GlobalIDはGlobalIDに署名がついたもので、内部的に `ActiveSupport::MessageVerifier` が使われています。(https://github.com/rails/globalid/blob/v0.5.2/lib/global_id/verifier.rb#L5)

```ruby:locate_signed.rb
require 'globalid'

class Person
  include GlobalID::Identification

  attr_reader :id
  def initialize(id = 1)
    @id = id
  end
  
  def self.find(id)
    new(id)
  end
end

SignedGlobalID.verifier = GlobalID::Verifier.new("test")
person_sgid = SignedGlobalID.create(Person.new(5), app: "test")

# 署名付きのBase64になる
puts person_sgid
# => "BAh7CEkiCGdpZAY6BkVUSSIYZ2lkOi8vdGVzdC9QZXJzb24vNQY7AFRJIgxwdXJwb3NlBjsAVEkiDGRlZmF1bHQGOwBUSSIPZXhwaXJlc19hdAY7AFQw--3ece953c9870bd0c3721a26142666e91e4efa6a3"

# Personクラスが復元される
puts GlobalID::Locator.locate_signed person_sgid
# => #<Person:0x00007ffa171d3738>
```

デシリアライズには `GlobalID::Locator.locate_signed` または `GlobalID::Locator.locate_many_signed` が使われます。

Railsに組み込まれる設定では以下のように `app.key_generator` が使われています。

```ruby
# https://github.com/rails/globalid/blob/v0.5.2/lib/global_id/railtie.rb#L24
  config.after_initialize do
    GlobalID.app = app.config.global_id.app ||= default_app_name
    SignedGlobalID.expires_in = app.config.global_id.fetch(:expires_in, default_expires_in)

    app.config.global_id.verifier ||= begin
      GlobalID::Verifier.new(app.key_generator.generate_key('signed_global_ids'))
    rescue ArgumentError
      nil
    end
    SignedGlobalID.verifier = app.config.global_id.verifier
  end
```      

Signed GlobalIDのデフォルトのシリアライザはMarshalであるため、secret_key_baseの値がわかる場合にはRCEが可能です。


### GlobalID.findからSignedGlobalIDを呼び出す

GlobalIDとSignedGlobalID自体も `self.find` を定義しています。(https://github.com/rails/globalid/blob/v1.0.0/lib/global_id/global_id.rb#L22)
そのため `GlobalID.find` などのデシリアライズの対象としてSignedGlobalIDが指定できると、secret_key_baseの値を知っている場合にはRCEが可能となります。


```ruby
sgid = SignedGlobalID.create(...)
gid = "gid://bcx/SignedGlobalID/#{CGI.escape(sgid)}"

# v1.0.1以前ではSignedGlobalIDでデシリアライズされた値が復元できる
result = GlobalID.find gid
```

https://github.com/rails/globalid/pull/148 の変更によりGlobalIDやSignedGlobalIDがデシリアライズ対象である場合に例外が発生するようになりました。v1.1.0のリリース内容に含まれています。

