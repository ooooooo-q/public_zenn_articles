---
title: "const_missing,method_missing"
---


### [Module#const_missing](https://docs.ruby-lang.org/ja/latest/method/Module/i/const_missing.html)


```ruby
module Bar
  BAR = 1

  def self.const_missing(name)
    puts name
  end

end

# Bar::a
#  undefined method `a' for Bar:Module (NoMethodError)

Bar::Aa
# Aa

# const_get経由でもconst_missingが呼び出される
Bar.const_get("Ab", false)
# Ab
```


定数が存在しない場合は、`const_missing`が呼ばれることもあります。実際に設定されることもすくないですし、デシリアライズでよばれるかは実装によります

marshal,yaml,constanisezで呼ばれるかどうか



### 事例

https://snyk.io/vuln/SNYK-RUBY-BINDATA-1314298

https://github.com/dmendel/bindata/blob/d99f050b88337559be2cb35906c1f8da49531323/lib/bindata/bits.rb#L166

https://github.com/rubysec/ruby-advisory-db/issues/476
https://blog.presidentbeef.com/blog/2020/09/14/another-reason-to-avoid-constantize-in-rails/


### method_missing

