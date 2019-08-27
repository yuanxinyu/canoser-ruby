# Canoser

[中文文档 Chinese document](/README-CN.md)

A ruby implementation of the canonical serialization for the Libra network.

Canonical serialization guarantees byte consistency when serializing an in-memory
data structure. It is useful for situations where two parties want to efficiently compare
data structures they independently maintain. It happens in consensus where
independent validators need to agree on the state they independently compute. A cryptographic
hash of the serialized data structure is what ultimately gets compared. In order for
this to work, the serialization of the same data structures must be identical when computed
by independent validators potentially running different implementations
of the same spec in different languages.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'canoser'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install canoser

## Usage

Canoser可以自动实现数据结构的序列化和反序列化，只是需要按要求定义数据结构。以一个实际的Libra代码中的数据结构为例：

```rust
pub struct AccountResource {
    balance: u64,
    sequence_number: u64,
    authentication_key: ByteArray,
    sent_events_count: u64,
    received_events_count: u64,
    delegated_withdrawal_capability: bool,
}
impl CanonicalSerialize for AccountResource {
    fn serialize(&self, serializer: &mut impl CanonicalSerializer) -> Result<()> {
        serializer
            .encode_struct(&self.authentication_key)?
            .encode_u64(self.balance)?
            .encode_bool(self.delegated_withdrawal_capability)?
            .encode_u64(self.received_events_count)?
            .encode_u64(self.sent_events_count)?
            .encode_u64(self.sequence_number)?;
        Ok(())
    }
}
```
在Libra使用的rust中，需要手动写代码实现数据结构的序列化，而且数据结构中的字段顺序和序列化时的顺序不一定一致。
在Canoser中，可以对应如下定义该数据结构
```ruby
  class AccountResource < Canoser::Struct
  	define_field :authentication_key, [Canoser::Uint8]
  	define_field :balance, Canoser::Uint64
  	define_field :delegated_withdrawal_capability, Canoser::Bool
  	define_field :received_events_count, Canoser::Uint64
  	define_field :sent_events_count, Canoser::Uint64
  	define_field :sequence_number, Canoser::Uint64
  end  
```
注意，ruby中的数据结构顺序要按照Libra中序列化的顺序来定义。

### 定义数据结构

定义数据结构的第一步是写一个类继承Canoser::Struct，然后通过define_field方法来定义该结构所拥有的字段。字段支持的类型有：

| 字段类型 | 可选子类型 | 说明 |
| ------ | ------ | ------ |
| Canoser::Uint8 |  | 无符号8位整数 |
| Canoser::Uint16 |  | 无符号16位整数 |
| Canoser::Uint32 |  | 无符号32位整数 |
| Canoser::Uint64 |  | 无符号64位整数 |
| Canoser::Bool |  | 布尔类型 |
| Canoser::Str |  | 字符串 |
| [] | 支持 | 数组类型 |
| {} | 支持 |  Map类型 |
| A Canoser::Struct |  | 嵌套的另外一个结构（不能循环引用） |

### 关于数组类型
数组里的数据，如果没有定义类型，那么缺省是Uint8。下面的两个定义等价：
```ruby
  class Arr1 < Canoser::Struct
    define_field :addr, []
  end
  class Arr2 < Canoser::Struct
    define_field :addr, [Canoser::Uint8]
  end  
```  
数组还可以定义长度，表示定长数据。比如Libra中的地址是256位，也就是32个字节，所以可以如下定义：
```ruby
  class Address < Canoser::Struct
    define_field :addr, [Canoser::Uint8], 32
  end  
```  
定长数据在序列化的时候，不写入长度信息。

### 关于Map类型
Map里的数据，如果没有定义类型，那么缺省是字节数组。下面的两个定义等价：
```ruby
  class Map1 < Canoser::Struct
    define_field :addr, {}
  end
  class Map2 < Canoser::Struct
    define_field :addr, {[Canoser::Uint8], [Canoser::Uint8]}
  end  
```  

下面是一个复杂的例子，包含三个数据结构：
```ruby
  class Addr < Canoser::Struct
    define_field :addr, [Canoser::Uint8], 32
  end

  class Bar < Canoser::Struct
    define_field :a, Canoser::Uint64
    define_field :b, [Canoser::Uint8]
    define_field :c, Addr
    define_field :d, Canoser::Uint32
  end

  class Foo < Canoser::Struct
    define_field :a, Canoser::Uint64
    define_field :b, [Canoser::Uint8]
    define_field :c, Bar
    define_field :d, Canoser::Bool
    define_field :e, {}
  end
```
这个例子参考自libra中canonical serialization的测试代码。

### 序列化和反序列化
在定义好Canoser::Struct后，不需要自己实现序列化和反序列化代码，直接调用基类的默认实现即可。以AccountResource结构为例：
```ruby
#序列化
obj = AccountResource.new(authentication_key:[...],...)
bytes = obj.serialize
#反序列化
obj = AccountResource.deserialize(bytes)
```


## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake test` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/yuanxinyu/canoser. This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [Contributor Covenant](http://contributor-covenant.org) code of conduct.

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).

