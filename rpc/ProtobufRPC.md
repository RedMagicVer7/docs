Protobuf RPC是一种基于TCP协议的二进制RPC通信协议。它以Protobuf作为基本的数据交换格式，并基于Protobuf内置的RPC Service形式，规定了通信双方之间的数据交换协议，以实现完整的RPC调用。

Protobuf RPC不考虑跨TCP连接的情况。

# 基本协议 #

##服务 ##

所有RPC服务都在某个IP地址上通过某个端口发布。

一个端口可以同时发布多个服务。服务以名字标识。服务名必须是UpperCamelCase，由大小写字母和数字组成，长度不超过64个字符。

一个服务可以包含一个或多个方法。每个方法以名字标识，由大小写字母、数字和下划线组成，长度不超过64个字符。考虑到不同语言风格差异较大，这里对方法命名格式不做强制规定。

四元组唯一地标识了一个RPC方法。

调用方法所需参数应放在一个Protobuf消息内。如果方法有返回结果，也同样应放在一个Protobuf消息内。具体定义由通信双方自行约定。特别地，可以使用空的Protobuf消息来表示请求/响应为空的情况。

##包##

包是Protobuf RPC的基本数据交换单位。每个包由包头和包体组成，其中包体又分为元数据、数据、附件三部分。具体的参数和返回结果放在数据部分。

包分为请求包和响应包两种。它们的包头格式一致，但元数据部分的定义不同。

##包头##

包头长度固定为12字节。前四字节为协议标识PRPC，中间四字节是一个32位整数，表示包体长度，最后四字节是一个32位整数，表示包体中的元数据包长度。整数均采用网络字节序表示。

##元数据##

元数据用于描述请求/响应。

    message RpcMeta {
        optional RpcRequestMeta request = 1;
        optional RpcResponseMeta response = 2;
        optional int32 compress_type = 3;
        optional int64 correlation_id = 4;
        optional int32 attachment_size = 5;
        optional ChunkInfo chuck_info = 6;
        optional bytes authentication_data = 7;
    };

请求包中只有request，响应包中只有response。实现可以根据域的存在性来区分请求包和响应包。

<table><thead>
<tr>
<th>参数</th>
<th>说明</th>
</tr>
</thead><tbody>
<tr>
<td>request</td>
<td>请求包元数据</td>
</tr>
<tr>
<td>response</td>
<td>响应包元数据</td>
</tr>
<tr>
<td>compress_type</td>
<td>详见附录<a href="#compress_algorithm">压缩算法</a></td>
</tr>
<tr>
<td>correlation_id</td>
<td>请求包中的该域由请求方设置，用于唯一标识一个RPC请求。请求方有义务保证其唯一性，协议本身对此不做任何检查。响应方需要在对应的响应包里面将correlation_id设为同样的值。</td>
</tr>
<tr>
<td>attachment_size</td>
<td>附件大小，详见<a href="#attachment">附件</a></td>
</tr>
<tr>
<td>chuck_info</td>
<td>详见<a href="#chunk_mode">Chunk模式</a></td>
</tr>
<tr>
<td>authentication_data</td>
<td>用于存放身份认证相关信息</td>
</tr>
</tbody></table>

###请求包元数据###

请求包的元数据主要描述了需要调用的RPC方法信息，Protobuf如下

    message RpcRequestMeta {
        required string service_name = 1; 
        required string method_name = 2;
        optional int64 log_id = 3; 
    };

<table><thead>
<tr>
<th>参数</th>
<th>说明</th>
</tr>
</thead><tbody>
<tr>
<td>service_name</td>
<td>服务名，约束见上文</td>
</tr>
<tr>
<td>method_name</td>
<td>方法名，约束见上文</td>
</tr>
<tr>
<td>log_id</td>
<td>用于打印日志。可用于存放BFE_LOGID。该参数可选。</td>
</tr>
</tbody></table>


###响应包元数据###

响应包的元数据是对返回结果的描述。如果出现任何异常，错误也会放在元数据中。其Protobuf描述如下

    message RpcResponseMeta { 
        optional int32 error_code = 1; 
        optional string error_text = 2; 
    };

<table><thead>
<tr>
<th>参数</th>
<th>说明</th>
</tr>
</thead><tbody>
<tr>
<td>error_code</td>
<td>发生错误时的错误号，0表示正常，非0表示错误。具体含义由应用方自行定义。</td>
</tr>
<tr>
<td>error_text</td>
<td>错误的文本描述</td>
</tr>
</tbody></table>

某些实现需要在元数据中增加自己专有的字段。为了避免冲突，并保证不同实现之间相互调用的兼容性，所有实现都需要向接口规范委员会申请一个专用的序号用于存放自己的扩展字段。

以Hulu为例，被分配的序号为100。因此Hulu可以使用这样的proto定义：

    message RpcMeta {
        optional RpcRequestMeta request = 1;
        optional RpcResponseMeta response = 2;
        optional int32 compress_type = 3;
        optional int64 correlation_id = 4;
        optional int32 attachment_size = 5;
        optional ChunkInfo chuck_info = 6;
        optional HuluMeta hulu_meta = 100;
    };
    message RpcRequestMeta {
        required string service_name = 1; 
        required string method_name = 2;
        optional int64 log_id = 3; 
        optional HuluRequestMeta hulu_request_meta = 100;
    };
    message RpcResponseMeta { 
        optional int32 error_code = 1; 
        optional string error_text = 2; 
        optional HuluResponseMeta hulu_response_meta = 100;
    };

因为只是将100这个序号保留给Hulu使用，因此Hulu可以自由决定是否添加这些字段，以及使用什么样的名字。其余实现使用的proto中不存在这些定义，会直接作为Unknown字段忽略。

当前分配的序号如下

    序号  实现
    100 Hulu
    101 Sofa

## 数据 ##

自定义的Protobuf Message。用于存放参数或返回结果。

## 附件 ##

某些场景下需要通过RPC来传递二进制数据，例如文件上传下载，多媒体转码等等。将这些二进制数据打包在Protobuf内会增加不必要的内存拷贝。因此协议允许使用附件的方式直接传送二进制数据。

附件总是放在包体的最后，紧跟数据部分。消息包需要携带附件时，应将RpcMeta中的attachment_size设为附件的实际字节数。

# Chunk模式 #

Protobuf不适合传输超大数据，也不适合需要流式传输的场景。针对这些情况，Protobuf RPC定义了Chunk模式来处理。

Chunk模式本质上是将一个大的数据流拆分成一个个小的Chunk包按序进行发送。如何拆分还原由通信双方确定。由于需要同时传输多个数据流，因此需要对不同的数据流加以标识。

元数据中带有chuck_info字段的包即为Chunk包。ChunkInfo的定义如下：

    messsage ChunkInfo {
            required int64 stream_id = 1; 
            required int64 chunk_id = 2; 
    }; 

stream_id用于唯一标识一个数据流，由发送方保证其唯一性，协议不对此进行任何检查。chunk_id从0开始严格递增。发送方需保证按序发送Chunk包。数据流的最后一个包chunk_id为-1。由于Protobuf RPC基于TCP协议，因此包之间的顺序可以保证。

协议只关注同一个TCP连接下的Chunk包有序性，应用方如需要在连接断开之后恢复数据流，需要设计其他的解决方案。

# 附录 #

## 压缩算法 ##

可以使用指定的压缩算法来压缩消息包中的数据部分。

<table><thead>
<tr>
<th>值</th>
<th>含义</th>
</tr>
</thead><tbody>
<tr>
<td>0</td>
<td>不压缩</td>
</tr>
<tr>
<td>1</td>
<td>使用Snappy 1.0.5</td>
</tr>
<tr>
<td>2</td>
<td>使用gzip</td>
</tr>
</tbody></table>

[http://gollum.baidu.com/ProtobufRPC](http://gollum.baidu.com/ProtobufRPC)