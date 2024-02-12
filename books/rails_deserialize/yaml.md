---
title: "YAML"
---

### [YAML.load (Psych.load)](https://docs.ruby-lang.org/ja/latest/method/Psych/s/load.html)

#### Psych 3.x までの挙動

YAMLもMarshalと同じ問題を抱えており、ruby標準のPsychモジュールによる `YAML.load` では[YAML](https://yaml.org/spec/1.2/spec.html)のタグとして `!ruby/xxx` で指定したクラスなどを復元できるため、Gadget ChainによってRCEが実行可能でした。(https://github.com/ruby/psych/blob/v3.3.0/lib/psych/visitors/to_ruby.rb#L66)

```ruby:yaml_sample.rb
require 'yaml'

class Cat
   attr_reader :color

   def initialize(color)
    @color = color
  end
end

cat = Cat.new("white")

# dumpでシリアライズするとruby向けのタグが含まれている
dump_str = YAML.dump(cat)
pp dump_str
# => "--- !ruby/object:Cat\n" + "color: white\n"

# Catのインスタンスが生成される
cat_loaded = YAML.load(dump_str)
pp cat_loaded.color
# => "white"

# クラスに存在していないattributeに値が設定されている
cat_loaded = YAML.load("--- !ruby/object:Cat\n" + "unexpect_attr: white\n")
pp cat_loaded.color
# => nil
pp cat_loaded.instance_variable_get(:@unexpect_attr)
# => "white"
```

Marshalと違い、YAMLでは[YAML.safe_load(Psych.safe_load)](https://docs.ruby-lang.org/ja/latest/method/Psych/s/safe_load.html)
を使うと復元可能なクラスを制限できるため、ある程度安全になります。

```ruby:yaml_safe_sample.rb
# 許可されていないクラスをデシリアライズしようとするとエラーが発生
cat_loaded = YAML.safe_load(dump_str)
# => Tried to load unspecified class: Cat (Psych::DisallowedClass)

# 第2引数に指定を追加すれば復元できる
cat_loaded = YAML.safe_load(dump_str, [Cat])
pp cat_loaded.color
# => "white"
```

[YAML.load_file(Psych.load_file)](https://docs.ruby-lang.org/ja/latest/method/Psych/s/load_file.html)はファイルを指定してデシリアライズを行いますが、復元するクラスを制限できずsafe_load_fileのメソッドもないので注意が必要です。

なお、database.yamlなどでrubyのコードが実行できるのは、RailsがERBを通しているためです。(https://github.com/rails/rails/blob/v6.1.4.1/railties/lib/rails/application/configuration.rb#L259, https://github.com/rails/rails/blob/v6.1.4.1/activesupport/lib/active_support/configuration_file.rb#L47)


#### 過去の事例

実際にRCEが可能だった事例として https://rubygems.org/ があります。

https://justi.cz/security/2017/10/07/rubygems-org-rce.html
https://blog.rubygems.org/2017/10/09/unsafe-object-deserialization-vulnerability.html

こちらの記事によれば、rubygems.orgへgemの登録時、gemのメタ情報のYAMLファイルのパースに `YAML.load` が使われていたため発生する問題だったようです。rubygems 2.6.14で修正されて、safe_loadが使用されています。(https://github.com/rubygems/rubygems/commit/510b1638ac9bba3ceb7a5d73135dafff9e5bab49)


### Psych 4.0 でデフォルトがsafe_loadへ

この本を書いている途中でPsych 4.0がリリースされ、デフォルトでsafe_loadが使われるようになりました。(https://github.com/ruby/psych/pull/487)

安全になったのですがbreaking changeでもあるため、いくつかの場所に影響がでているようです。

https://secret-garden.hatenablog.com/entry/2021/05/24/235536

上記の記事が詳しく、ruby 2.5以下でも動かす場合には引数の違いに注意が必要です。

---

### [SafeYAML](https://github.com/dtao/safe_yaml)

`safe_yaml` のgemがrequireされると、既存のYAMLのメソッドが上書きされてデフォルトで安全になります。
Psych 4.0以降を利用しているのであれば不要となりますが、挙動が少し違います。

```ruby:safe_yaml.rb
require 'yaml'
require 'safe_yaml'

class Cat
   attr_reader :color

   def initialize(color)
    @color = color
  end
end

cat = Cat.new("white")

dump_str = YAML.dump(cat)
pp dump_str
# => "--- !ruby/object:Cat\n" + "color: white\n"

cat_loaded = YAML.load(dump_str)
pp cat_loaded
# => {"color"=>"white"}

cat_loaded = YAML.load_file("./test.yaml")
pp cat_loaded
```

```
❯ bundle exec ruby safe_yaml_test.rb
"--- !ruby/object:Cat\n" + "color: white\n"
Called 'load' without the :safe option -- defaulting to safe mode.
You can avoid this warning in the future by setting the SafeYAML::OPTIONS[:default_mode] option (to :safe or :unsafe).
{"color"=>"white"}
{"color"=>"white"}
```

一方、delayed_jobではsafe_yamlが読み込まれている場合でも意図したデシリアライズを行うため、更に別メソッドを定義しています。(https://github.com/collectiveidea/delayed_job/blob/5ac5adea8d18325d0470eeebfa81227b1f5961e3/lib/delayed/syck_ext.rb#L36)

