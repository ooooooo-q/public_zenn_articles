---
title: "secret_key_base"
---

### secret_key_base について

`secret_key_base` はcredentials.ymlなどから設定される値で、Rails内の暗号化や証明のキーの元の値として扱われます。
RAILS_ENVがproductionであればcredentials.ymlに設定された値が、developmentであり設定がなければランダムな値が使用されます。

`secret_key_base` の値は `ActiveSupport::KeyGenerator` で生成するkeyのsecretにのみ使用されています。

```ruby
# https://github.com/rails/rails/blob/v6.1.4.1/railties/lib/rails/application.rb#L173
def key_generator
  # number of iterations selected based on consultation with the google security
  # team. Details at https://github.com/rails/rails/pull/6952#issuecomment-7661220
  @caching_key_generator ||= ActiveSupport::CachingKeyGenerator.new(
    ActiveSupport::KeyGenerator.new(secret_key_base, iterations: 1000)
  )
end
```

[Rails セキュリティガイド](https://railsguides.jp/security.html)ではCookieでの利用例についてのみ言及されていますが、他にもmessage_verifierや6.1で導入された署名付きのレコードなどで利用されています。

```ruby
# https://github.com/rails/rails/blob/v6.1.4.1/railties/lib/rails/application.rb#L199
def message_verifier(verifier_name)
  @message_verifiers[verifier_name] ||= begin
    secret = key_generator.generate_key(verifier_name.to_s)
    ActiveSupport::MessageVerifier.new(secret)
  end
end
```
```ruby
# https://github.com/rails/rails/blob/v6.1.4.1/activerecord/lib/active_record/railtie.rb#L277
initializer "active_record.set_signed_id_verifier_secret" do
  ActiveSupport.on_load(:active_record) do
    self.signed_id_verifier_secret ||= -> { Rails.application.key_generator.generate_key("active_record/signed_id") }
  end
end
```

secret_key_baseの値が漏洩すると前述のActiveStorageによるRCEなどが可能になりますが、以下のように値がわかってしまう場合もあります。


### 秘密を知る開発者

アプリケーションの秘密情報を `credentials.yml.enc` などで管理している場合、開発者であればその中身も知ることができます。
その事自体は秘密情報の管理方針次第なのですが、ActiveStorageのURLを利用することでSSHのポートが開いていないサーバであっても任意の操作が可能になります。


### [CVE-2019-5420] Possible Remote Code Execution Exploit in Rails Development Mode

https://groups.google.com/forum/#!topic/rubyonrails-security/IsQKvDqZdKw

hackeroneのレポートにも書かれていますが、Rails5.2.2.1未満では開発モードの `secret_key_base` はアプリケーションの名前から計算可能であったため、アプリケーションの名前を知っている開発サーバには攻撃可能な状態でした。現在はdevelopmentモードで動かしている場合はランダムな値になるため安全なようです。（ただし、明示的にsecret_key_baseを渡している場合を除きます。）


### Directory Traversal

https://gist.github.com/mala/bdcf8681615d9b5ba7814f48dcea8d60

CVE-2019-5420と同時に公開されたCVE-2019-5418は任意のパスのファイルが閲覧できてしまうDirectory Traversalでした。そのため、CVE-2019-5418を使ってsecret_key_baseの値を取得した後Active StorageのURLに細工したリクエストを投げるとRCEできる組み合わせになってしまっていました。


### エラーメッセージ経由

https://hackerone.com/reports/460545

https://blog.harshjaiswal.com/rce-due-to-showexceptions/

エラー発生時に詳細を表示する[Rack::ShowExceptions](https://github.com/rack/rack/blob/v2.2.2/lib/rack/show_exceptions.rb)が本番環境でも読み込まれていたため、わざとエラーを起こすことで `secret_key_base` の値が見えてしまった事例です。


### サムネイル経由

これまで脆弱性を調査してきた中で、画像がアップロードされたときに作るサムネイルの中に任意のファイルの中身が表示されてしまう脆弱性を何度か報告したことがあります。imagemagickやghostscriptなどの問題です。

[ImageTragick](https://imagetragick.com/)を始めとしたこれらの問題は知識が必要なうえに色々対応が難しいです。Active StorageではサムネイルをRailsサーバ側で作ることにも注意が必要です。SaaSやCDNでサムネイルを作ってくれるサービスを使えるのであれば使ったほうが楽そうです。

