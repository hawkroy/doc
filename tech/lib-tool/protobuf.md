## Protobuf使用

Protobuf的文件的语法结构

```protobuf
// {options} set
message {
	// fields declarations
	// each field like below
	// 		{field_rule}  {field_type} {field_name} = {unique_number} {options};
}
```

{field_type}可以是scalar type，也可以是复合类型 (protobuf支持message的嵌套定义)。对于scalar type，protobuf中定义了如下的scalar类型

| .proto Type | Note                                                     | c++ Type | python Type             |
| ----------- | -------------------------------------------------------- | -------- | ----------------------- |
| double      |                                                          | double   | float                   |
| float       |                                                          | float    | float                   |
| int32       | 使用varint的变长编码，对于负数，性能不好，推荐使用sint32 | int32    | int                     |
| int64       | 使用varint的变长编码，对于负数，性能不好，推荐使用sint64 | int64    | int/long[1]             |
| uint32      | 使用varint的变长编码                                     | uint32   | int/long[1]             |
| uint64      | 使用varint的变长编码                                     | uint64   | int/long[1]             |
| sint32      | 使用varint的变长编码，对于负数性能较好                   | int32    | int                     |
| sint64      | 使用varint的变长编码，对于负数性能较好                   | int64    | int/long[1]             |
| fixed32     | 固定使用4Byte编码，当值大于2<sup>28</sup>时性能更好      | uint32   | int/long[1]             |
| fixed64     | 固定使用8Byte编码，当值大于2<sup>56</sup>时性能更好      | uint64   | int/long[1]             |
| sfixed32    | 支持负数，固定4Byte编码                                  | int32    | int                     |
| sfixed64    | 支持负数，固定8Byte编码                                  | int64    | int/long[1]             |
| bool        |                                                          | bool     | bool                    |
| string      | 使用UTF-8的unicode或是7bit的ASCII编码                    | string   | unicode(py2) / str(py3) |
| bytes       | 包含任意byte序列的编码                                   | string   | bytes                   |

[1],  decode时按照long进行decode, 可以使用int进行设置

对于{unique_number}每个field必须有一个单独的number，这个number在protobuf进行serialize和deserialize的时候会被用到，且在使用过程中不变。其中

- ​	1-15: 1Byte encode					对于常用的消息域尽量使用1-15进行编码
- 16-2047: 2Byte encode
- 最大的消息编码值2<sup>29</sup>-1(536,870,911)
- 19000(FieldDescriptor::kFirstReservedNumber)-19999(FieldDescriptor::kLastReservedNumber)的field number不能使用，被protobuf用于内部实现使用
- 同时，可以在message中定义`reserved`关键字来保留某些不允许使用的field number

对于{field_rule}，