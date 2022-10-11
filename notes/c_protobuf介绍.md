- [先关注一下GPB怎么序列化和反序列化](#先关注一下gpb怎么序列化和反序列化)
  - [Parsing and Serialization(C++)](#parsing-and-serializationc)
    - [举例](#举例)
- [encoding介绍](#encoding介绍)

官方手册: [https://developers.google.com/protocol-buffers](https://developers.google.com/protocol-buffers)

# 先关注一下GPB怎么序列化和反序列化

## Parsing and Serialization(C++)

Finally, each protocol buffer class has methods for writing and reading messages of your chosen type using the protocol buffer [binary format](https://developers.google.com/protocol-buffers/docs/encoding). These include:

* `bool SerializeToString(string* output) const;`: serializes the message and stores the bytes in the given string. Note that the bytes are binary, not text; we only use the `string` class as a convenient container.
* `bool ParseFromString(const string& data);`: parses a message from the given string.
* `bool ParseFromArray(const void * data, int size)`: Parse a protocol buffer contained in an array of bytes.
* `bool SerializeToOstream(ostream* output) const;`: writes the message to the given C++ `ostream`.
* `bool ParseFromIstream(istream* input);`: parses a message from the given C++ `istream`.

These are just a couple of the options provided for parsing and serialization. Again, see the [`Message` API reference](https://developers.google.com/protocol-buffers/docs/reference/cpp/google.protobuf.message#Message) for a complete list.

### 举例
比如定义了一个proto是AddressBook, protoc生成了对应的类.
```
# 声明这个类的对象
tutorial::AddressBook address_book;

# 从os.in读一个对象出来
fstream input(argv[1], ios::in | ios::binary);
address_book.ParseFromIstream(&input)

# 写到os.out
fstream output(argv[1], ios::out | ios::trunc | ios::binary);
address_book.SerializeToOstream(&output)
```

# encoding介绍
是binary模式的编码, int是变长的, 叫`varints`

简明列表:[https://developers.google.com/protocol-buffers/docs/encoding](https://developers.google.com/protocol-buffers/docs/encoding)

| Type | Meaning | Used For |
| --- | --- | --- |
| 0 | Varint | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
| 1 | 64-bit | fixed64, sfixed64, double |
| 2 | Length-delimited | string, bytes, embedded messages, packed repeated fields |
| 3 | Start group | groups (deprecated) |
| 4 | End group | groups (deprecated) |
| 5 | 32-bit | fixed32, sfixed32, float |
