---
title: "XML"
---

XMLを扱う場合、ruby標準の[rexml](https://docs.ruby-lang.org/ja/latest/library/rexml.html)などは安全ですが、一部のgemでは安全ではないケースがあります。


### [Hash.from_xml](https://api.rubyonrails.org/v6.1.0/classes/Hash.html#method-c-from_xml)

`Hash.from_xml` はActiveSupportによるHashクラスの拡張で、XMLの文字列からHashを作成する機能です。

```ruby:from_xml.rb
require 'active_support'
require 'active_support/core_ext'

xml = <<-XML
  <?xml version="1.0" encoding="UTF-8"?>
    <hash>
      <blue type="file">aaa</blue>
      <green type="integer">200</green>
      <red type="date">2021-12-21</red>
    </hash>
XML

puts Hash.from_xml(xml)
# => {"hash"=>{"blue"=>#<StringIO:0x00007fd22624b300 @original_filename=nil, @content_type=nil>, "green"=>200, "red"=>Tue, 21 Dec 2021}}
```

`type` の値によって、パース方法を指定できます。(https://github.com/rails/rails/blob/v6.1.4.1/activesupport/lib/active_support/xml_mini.rb#L64
)


#### 過去の事例

3.2.11以前のバージョンのRailsでは `type` にYAMLも指定可能でした。さらに、4.0以前のバージョンではRailsへのリクエストボディのフォーマットとしてXMLを指定でき、パースには `Hash.form_xml` が使用されていました。

そのため、リクエストボディのXMLの中に細工したYAMLをいれることでRCEが可能な状態でした。([Multiple vulnerabilities in parameter parsing in Action Pack (CVE-2013-0156)](https://groups.google.com/g/rubyonrails-security/c/61bkgvnSGTQ),[Rails 3.2.10 Remote Code Execution](https://github.com/charliesome/charlie.bz/blob/master/posts/rails-3.2.10-remote-code-execution.md))

現在でも `Hash.from_trusted_xml(xml)` や `Hash.from_xml(xml, [])` が利用された場合はYAMLを指定できるためRCEが可能です。しかし、これらが実際に利用されていることは少なそうです。


### [Ox.load](https://github.com/ohler55/ox)

OxはOjと同じ作者によるXMLのパーサであり、こちらもクラスのシリアライズに対応しています。
(https://www.ohler.com/dev/ruby_object_xml_serialization/ruby_object_xml_serialization.html)

Ojと同じ様に `:mode => :object` が指定されている場合にデシリアライズがされます。Ojと違い、modeの指定がない場合にはデシリアライズされないのですが、読み込まれるXMLの中にmodeの指定があるとデシリアライズされるようです。

```ruby:ox.rb
require 'ox'

class Cat
  def initialize(color)
    @color = color
  end
end

cat = Cat.new('white')

xml = Ox.dump(cat, :indent => 2)
puts xml
# <o c="Cat">
#   <s a="@color">white</s>
# </o>

puts Ox.load(xml)
# => <Ox::Element:0x00007f8b850f3328>

puts Ox.load(xml, mode: :hash_no_attrs)
# => {:o=>{:s=>"white"}}

puts Ox.load(xml, :mode => :object)
# => #<Cat:0x00007f8b850f2db0>

puts Ox.parse_obj(xml)
# => #<Cat:0x00007fd7500b7910>

xml = <<XML
<?ox mode="object" effort="strict" circular="false" xsd_date="false" ?>
<o c="Cat">
  <s a="@color">white</s>
</o>
XML

puts Ox.load(xml)
# => #<Cat:0x00007fae549c8d58>
# XML側のmodeの指定が反映されている

puts Ox.load(xml, mode: :hash_no_attrs)
# => {:o=>{:s=>"white"}}

puts Ox.load(xml, :mode => :object)
# => #<Cat:0x00007f8b850f2db0>

puts Ox.parse_obj(xml)
# => #<Cat:0x00007fd7500b7910>
```


### [Plist.parse_xml](https://github.com/patsplat/plist/tree/master)

Property Listを扱うgem、PlistではXMLのなかにMarshalでdumpされたデータを含むことができます。これが危険であることはReadmeでも書かれています。

[Security considerations](https://github.com/patsplat/plist/tree/master#security-considerations-)

> Plist.parse_xml uses Marshal.load for <data/> attributes. If the <data/> attribute contains malicious data, an attacker can gain code execution. You should never use Plist.parse_xml with untrusted plists!

```ruby:plist.rb
require 'plist' 

class Cat
  def initialize(color)
    @color = color
  end
end

cat = Cat.new('white')

xml = Plist::Emit.dump(cat, true)
puts xml
# <?xml version="1.0" encoding="UTF-8"?>
# <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
# <plist version="1.0">
# <!-- The <data> element below contains a Ruby object which has been serialized with Marshal.dump. -->
# <data>
# BAhvOghDYXQGOgtAY29sb3JJIgp3aGl0ZQY6BkVU
# </data>
# </plist>

data = Plist.parse_xml(xml);
puts data
# => #<Cat:0x000000014725b4b0>
```


### [XMLRPC](https://github.com/ruby/xmlrpc)

Ruby3.0.0未満では、rubyでXMLを使ったRPCを行うライブラリが標準添付ライブラリに含まれていました。

https://hackerone.com/reports/1189419

XMLRPCでは通信時に送信するデータをシリアライズする際の対象となるクラスに制限がありましたが、デシリアライズの際には制限されていないためリクエストに細工すると想定外のクラスを含めることができました。
0.3.3で[修正](https://github.com/ruby/xmlrpc/pull/36)されましたが、CVEは発行されていません。
