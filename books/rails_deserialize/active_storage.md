---
title: "Active Storage"
---

まずは手元にあるRailsアプリケーションで `bin/rails routes` を試してみてください。

出力結果に以下のURLは含まれていたでしょうか？

```
rails_service_blob GET  /rails/active_storage/blobs/:signed_id/*filename(.:format)                               active_storage/blobs#show
rails_blob_representation GET  /rails/active_storage/representations/:signed_blob_id/:variation_key/*filename(.:format) active_storage/representations#show
rails_disk_service GET  /rails/active_storage/disk/:encoded_key/*filename(.:format)                              active_storage/disk#show
update_rails_disk_service PUT  /rails/active_storage/disk/:encoded_token(.:format)                                      active_storage/disk#update
rails_direct_uploads POST /rails/active_storage/direct_uploads(.:format)                                           active_storage/direct_uploads#create
```

含まれていた場合はそのRailsのアプリケーションにはRCE(任意コード実行)の可能性があります。
実際には `secret_key_base` の値を知っている必要がありますし、シリアライザの指定を変更することで対策可能です。


### Active StroageのURL

これらはRails 5.2から入った[Active Storage](https://railsguides.jp/active_storage_overview.html)を利用する際に必要なURLです。Rails5.2以降であれば、`rails new` した際にActive Storageも含まれています。

Active Storageの機能を使うためには `rails active_storage:install` の実行とDBのmigrationが必要です。([Active Storage の概要 - 2 セットアップ](
https://railsguides.jp/active_storage_overview.html#%E3%82%BB%E3%83%83%E3%83%88%E3%82%A2%E3%83%83%E3%83%97))
しかし、セットアップを行なう前であってもURLにはアクセスが可能です。config/application.rbのrequireから `"active_storage/engine"` を外していなければ（もしくは `require "raisl/all"` となっていれば）、Active Storageを利用する必要がない場合であってもroutesに含まれています。その状態では、`/rails/active_storage/disk/test/test` にアクセスすると、通常の404ページではなく空のレスポンスが返されます。(https://github.com/rails/rails/blob/v6.1.4.1/activestorage/app/controllers/active_storage/disk_controller.rb#L16)


2019年の夏にはRails6.0がリリースされ、セキュリティのサポート対象となるバージョンは5.2以上となったこともあり、現在は多くのRailsアプリケーションでこれらのURLがアクセス可能になっているのではないでしょうか。([Maintenance Policy for Ruby on Rails - 3 Security Issues](https://guides.rubyonrails.org/maintenance_policy.html#security-issues))

実際に利用されるURLは、以下のようにパラメータがbase64でエンコードされています。

```
/rails/active_storage/blobs/redirect/eyJfcmFpbHMiOnsibWVzc2FnZSI6IkJBaHBCZz09IiwiZXhwIjpudWxsLCJwdXIiOiJibG9iX2lkIn19--ed4ee8109834f4dd747bfb68d7a7ddc2e43e8f69/alert.svg
```

これをデコードしていくと、ファイルのID(=1)がとりだせます。

```ruby:decode_url.rb
❯ irb
irb(main):001:0> require 'base64'
=> true

irb(main):002:0> ::Base64.strict_decode64("eyJfcmFpbHMiOnsibWVzc2FnZSI6IkJBaHBCZz09IiwiZXhwIjpudWxsLCJwdXIiOiJibG9iX2lkIn19")
=> "{\"_rails\":{\"message\":\"BAhpBg==\",\"exp\":null,\"pur\":\"blob_id\"}}"

irb(main):003:0> ::Base64.strict_decode64("BAhpBg==")
=> "\x04\bi\x06"

irb(main):004:0> Marshal.load("\x04\bi\x06")
=> 1
```

RCEが可能になる理由は、URLで渡されるパラメータのデシリアライズに `Marshal.load` が使われているためです。


### ActiveSupport::MessageVerifier と ActiveSupport::MessageEncryptor

Railsで認証や署名に使われる `ActiveSupport::MessageVerifier` と `ActiveSupport::MessageEncryptor` のシリアライザにはデフォルトとしてMarshalが指定されており、Active StorageのURLにもこれらのクラスが使われています。(https://github.com/rails/rails/blob/v6.1.4.1/activesupport/lib/active_support/message_verifier.rb#L110, https://github.com/rails/rails/blob/v6.1.4.1/activesupport/lib/active_support/message_encryptor.rb#L142)ただし、シリアライザが実行される前に `secret_key_base` の値を元にした検証がされるため、この値が知られていなければRCEはできません。

```ruby:verifier.rb
require 'active_support'

verifier = ActiveSupport::MessageVerifier.new('dummy_secret')
pp verifier
# #<ActiveSupport::MessageVerifier:0x00007f9a45813058
#  @digest="SHA1",
#  @on_rotation=nil,
#  @options={},
#  @rotations=[],
#  @secret="dummy_secret",
#  @serializer=Marshal>

token = verifier.generate({cat: "color"}, purpose: :test)
pp token
# => "eyJfcmFpbHMiOnsibWVzc2FnZSI6IkJBaDdCam9JWTJGMFNTSUtZMjlzYjNJR09nWkZWQT09IiwiZXhwIjpudWxsLCJwdXIiOiJ0ZXN0In19--649cea0bd9f62a311378b88f5785cc3ac4314244"

pp verifier.verified(token, purpose: :test) 
# => {:cat=>"color"}

# シリアライザにJSONを指定することも可能
json_verifier = ActiveSupport::MessageVerifier.new('dummy_secret', serializer: JSON)
pp json_verifier
# #<ActiveSupport::MessageVerifier:0x00007f9a442f3920
#  @digest="SHA1",
#  @on_rotation=nil,
#  @options={:serializer=>JSON},
#  @rotations=[],
#  @secret="dummy_secret",
#  @serializer=JSON>

json_token = json_verifier.generate({cat: "color"}, purpose: :test)
pp json_token
# => "eyJfcmFpbHMiOnsibWVzc2FnZSI6ImV5SmpZWFFpT2lKamIyeHZjaUo5IiwiZXhwIjpudWxsLCJwdXIiOiJ0ZXN0In19--6deaf5555f0d3afc3f23baf3b00887c6302abef1"
```

値を知る方法があるとRCEが可能であるという話を、Ruby on Railsに報告したのが以下のhackeroneのレポートです。([hackerone](https://hackerone.com/)は脆弱性報告のハンドリングを行うサービスです。)

https://hackerone.com/reports/473888


Ruby 2.7以降ではERBに対してMarshalのデシリアライズが禁止されたため、レポートの内容そのままでは再現できなくなりましたが、他の方法によりRCEは可能です。




