
###基本帧协议

####每个帧的结构:

      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-------+-+-------------+-------------------------------+
     |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
     |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
     |N|V|V|V|       |S|             |   (if payload len==126/127)   |
     | |1|2|3|       |K|             |                               |
     +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
     |     Extended payload length continued, if payload len == 127  |
     + - - - - - - - - - - - - - - - +-------------------------------+
     |                               |Masking-key, if MASK set to 1  |
     +-------------------------------+-------------------------------+
     | Masking-key (continued)       |          Payload Data         |
     +-------------------------------- - - - - - - - - - - - - - - - +
     :                     Payload Data continued ...                :
     + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
     |                     Payload Data continued ...                |
     +---------------------------------------------------------------+

+ FIN： 1 bit
		表示这是一个消息的最后的帧。第一个帧也可能是最后一个。

+ RSV1、2、3： 1 bit each
		除非一个扩展经过协商赋予了非零值以某种含义，否则必须为0
		如果没有定义非零值，并且收到了非零的RSV，则websocket链接会失败

+ Opcode： 4 bit
		解释说明 “Payload data” 的用途/功能
		如果收到了未知的opcode，最后会断开链接
		定义了以下几个opcode值:
		    %x0 : 代表连续的帧
		    %x1 : text帧
		    %x2 ： binary帧
		    %x3-7 ： 为进一步的非控制帧而保留的
		    %x8 ： 链接关闭
		    %x9 ： ping包
            %xA : pong包
            %xB-F ： 为进一步的非控制帧而保留的

+ Mask： 1 bit
        定义“payload data”是否被标记了
        如果置1， “Masking-key”就会被赋值
        所有从客户端发往服务器的帧都会被置1
        
+ Payload length： 7 bit | 7+16 bit | 7+64 bit
		“payload data” 的长度如果在0~125 bytes范围内，它就是“payload length”，
		如果是126 bytes， 紧随其后的被表示为16 bits的2 bytes无符号整型就是“payload length”，
		如果是127 bytes， 紧随其后的被表示为64 bits的8 bytes无符号整型就是“payload length”
		
		注意在所有情况下，最小字节数必须被用来编码长度
		
+ Masking-key： 0 or 4 bytes
		所有从客户端发送到服务器的帧都包含一个32 bits的值如果“mask bit”被设置成1，否则为0 bit
		
+ Payload data:  (x+y) bytes
		它是"Extension data"和"Application data"的总和.
		
+ Extension data:  x bytes
		除非扩展被定义，否则就是0
		任何扩展必须指定其Extension data的长度
		
+ Application data:  y bytes
		占据"Extension data"之后的剩余帧的空间
		
* 注意：这些数据都是以二进制形式表示的，而非ascii编码字符串


