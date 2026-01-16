# WARNING  
This document is currently marked as incomplete.  

# MicroFileSystem
MFS is a lightweight, asynchronus RPC protocol with filesystem-like semantics on resources. It is intended to be used as a control plane for MCUs and other types of embedded systems, to control hardware and high-level software operations.  
  
# Transport  
Primarily, TCP is used for MFS. However, MFS can also make use of UART, CAN, SPI and similar transport protocols, as long as the underlying transport HAL can convert it into a byte-stream.  
Additionally, MFS can be used as a P2P (Peer to Peer) and a client-to-server modes. (Explained in later sections).  

# Defaults  
By default, MFS requires a minimum total message lenght buffer of 32 bytes. (Used for buffer size negotion purposes). (This size can only be increased later during the connection as the server advertises its capabilities)  
And the connection starts off with the connection initiator being a client, and the connection acceptor as the server.  

# Peer to Peer mode  
In peer to peer mode, each side can send MFS requests to one another. Making both sides a server and a client at the same time over a single link.  
However, peer to peer mode is still point-to-point. There is only two peers. No more is allowed. This simply shifts MFS from client to server to server-to-server.  
  
# Base headers  
Base headers (unchanging accross versions) are defined as:  
    len[4], id[4], version[2], type[2], crc[4], priority[1]  
len is the total lenght of the entire MFS message (Including itself). (Little endian).  
id is the message id of the message. (Little endian). (ID collissions are allowed as long as it is unique among in-flight requests.)  
version is the MFS version of the message. (Little Endian). (Details in a later section.)  
type is the request type. Explained in a later section. (Little endian).  
crc is the crc of the entire message. (CRC should be computed as if it was filled as zeros during computation. CRC computations beings from the start of base headers). CRC-32 is used.  
priority is the priority of this message. Higher priority requests are handled first when theres multiple in-flight requests.  

# Version  
The highest 8 bits are major version.  
The lower 8 bits are minor version.  
This is MFSv2. Which makes its version:  
    `0x0200` in hexidecimal.  
  
# Message types
These are additional headers that are appended to the base headers, as the type directs.  
The highest bit is the response bit (used for responses to a specific message type)  
the second highest bit is the error bit. (used for indicating the message has resulted in an error.)  
  
MFSv2 supporting servers must implement the following message types (Numbers in decimal):  
0: noop. Adds no additional headers. No operation.  
1: get_size. Adds no additional headers.  Gets the maximum message lenght the server can support.  
2: get_async_size. Adds no additional headers.  Gets the maximum amount of messages the server can handle. (Maximum in-flight messages).  
3: get_attributes. `path_len[4], path[path_len]`. Gets the attributes of given file at the path.  path_len is a little endian uint32 defining the lenght of the path, which is a ASCII string.  
4: set_p2p_mode. Adds no additional headers. Swiches the connection into a peer-to-peer mode as defined above.  
5: seek_read. `path_len[4], path[path_len], seek[4], amount[4]`. path_len, seek and amount are little endian. path_len is the lenght of the path, which is an ASCII string. seek is the offset to start reading from and amount is the amount of bytes to read.  
6: seek_write. `path_len[4], path[path_len], seek[4], data_len[4], data[data_len]`. path_len, seek and data_len is little-endian. path_len is the lenght of the path, which is an ASCII string, seek is the offset to start writing from, data_len is the lenght of the data which is the data to write to the file. Writes data_len amount of bytes at seek `seek` to the file at path.  
7: get_file. `path_len[4], path[path_len]`. path_len is little endian. Gets the entire contents of the file. Does not require seeking support.  
8: replace_file. `path_len[4], path[path_len], data_len[4], data[data_len],` data_len and path_len is little endian. Replaces the entire contents of the file. Does not require seeking support.  
9: list. `path_len[4], path[path_len]`. path_len is little endian. Lists all files and directories under given 
  
# File attributes
In MFSv2, there is 3 attributes:  
read-write mode,  
atomic,  
seeking support.  

read-write mode defines the reading and writing mode of the file.  
atomic files are whoose read and write operations happen atomically per message. 
seeking files are files that support seeking.  
File attributes are 8 bits. (leftmost bit is highest bit)  
    x x x x 0 0 0 0  
highest bit is read bit (Determines if the file can be read from)  
second highest is write bit (Determines if the file can be written to)  
third highest is atomic bit (Determines if the file is atomic)  
fourth highest bit is the seek bit (Determines if the file supports seeked operations.)
