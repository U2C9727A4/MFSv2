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
    len[4], id[4], version[2], type[2], crc[4]
len is the total lenght of the entire MFS message (Including itself). (Little endian). uint32.  
id is the message id of the message. (Little endian). (ID collissions are allowed as long as it is unique among in-flight requests.) uint32.  
version is the MFS version of the message. (Little Endian). (Details in a later section.) uint16.  
type is the request type. Explained in a later section. (Little endian). uint16.  
crc is the crc of the entire message. (CRC should be computed as if it was filled as zeros during computation. CRC computations beings from the start of base headers). CRC-32 is used.  

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
1: get_size. Adds no additional headers.  Gets the maximum message lenght the server can both recieve and send.  
2: get_async_size. Adds no additional headers.  Gets the maximum amount of messages the server can handle. (Maximum in-flight messages).  
3: get_attributes. `path_len[4], path[path_len]`. Gets the attributes of given file at the path.  path_len is a little endian uint32 defining the lenght of the path, which is a ASCII string. path_len is an uint32.  
4: set_p2p_mode. Adds no additional headers. Swiches the connection into a peer-to-peer mode as defined above.  
5: seek_read. `path_len[4], path[path_len], seek[4], amount[4]`. path_len, seek and amount are little endian. path_len is the lenght of the path, which is an ASCII string. seek is the offset to start reading from and amount is the amount of bytes to read. path_len, seek and amount is uint32.  
6: seek_write. `path_len[4], path[path_len], seek[4], data_len[4], data[data_len]`. path_len, seek and data_len is little-endian. path_len is the lenght of the path, which is an ASCII string, seek is the offset to start writing from, data_len is the lenght of the data which is the data to write to the file. Writes data_len amount of bytes at seek `seek` to the file at path. path_len, seek, data_len are uint32.  
7: get_file. `path_len[4], path[path_len]`. path_len is little endian. Gets the entire contents of the file. Does not require seeking support. path_len is an uint32. Atomic.  
8: replace_file. `path_len[4], path[path_len], data_len[4], data[data_len],` data_len and path_len is little endian. Replaces the entire contents of the file. Does not require seeking support. path_len and data_len is uint32. Atomic.  
9: list. `path_len[4], path[path_len]`. path_len is little endian. Lists all files and directories under given path. path_len is uint32.
  
## Message response types
They follow the same structure as the normal message types, expect for the fact they have the response bit set.  
0: noop. Adds no additional headers.  
1: get_size. `size[4]`. size is little-endian. size represents the maximum message size the server can handle.  
2: get_async_size. `amount[2]`. amount is little-endian. Amount represents the maximum in-flight messages the server can handle. amount is uint16.  
3: get_attributes. `attributes[1], size[4]`. size is little endian. Attributes represent the attribute bitmask of the file at path, while size represents the file size. attributes is uint8. size is uint32.  
4: set_p2p_mode. Adds no additional headers. Response only sent when the mode switching is successful. in-flight messages that are sent before the switch are invalidated and cleared when switching happens.  
5: seek_read. `data_len[4], data[data_len]`. data_len is little endian. data_len represents the data lenght and data represents the data that was read. data_len is uint32.   
6: seek_write. `bytes_written[4]` bytes_written is little endian. bytes_written represents the amount of bytes that was written to the file. bytes_written is an uint32.  
7: get_file. `data_len[4], data[data_len]`. data_len is little endian. data_len represents the data lenght and data represents the data that was read. data_len is uint32.  
8: replace_file. No additional headers. a non-error return means the file was successfully replaced.  
9: list. `files_len[4], files[files_len], dirs_len[4] , dirs[dirs_len]`. 
TODO: Finish this.

## Message errors
All errors (error type bit set) follow these headers:  
err[4]. err is little-endian. err represents the error code. err is an uint32.  
  
An error cannot be a request onto itself, due to this an error bit set asserts that the response bit is set aswell.  
  
### Error codes  
(Error codes are in decimal)  
Error codes ranging from 0 to 1000 are queue errors.  
0: Queue is full.  
Error codes ranging from 1000 to 2000 are message errors  
1000: CRC verification failure.  
1001: Message bigger than buffers.  
1002: Unsupported version.  
1003: Unsupported message type.  
1004: ID collision.  
1005: Malformed message.  
Error codes ranging from 2000 to 3000 are VRS (Virtual Resource Space) errors.  
2000: Resource at string does not exist.  
2001: Resource is not writeable.  
2002: Resource is not readable.  
2003: Resource is not seekable.  
2004: Resource at path is not a file.
Error codes ranging from 3000 to 5000 are request errors.

# File attributes
In MFSv2, there is 4 attributes:  
read-write mode,  
atomic,  
seeking support.  
appending support.  

read-write mode defines the reading and writing mode of the file.  
atomic files are whoose read and write operations happen atomically per message. 
seeking files are files that support seeking.  
appending files that can only be appended to.  
File attributes are 8 bits. (leftmost bit is highest bit)  
    x x x x x 0 0 0  
highest bit is read bit (Determines if the file can be read from)  
second highest is write bit (Determines if the file can be written to)  
third highest is atomic bit (Determines if the file is atomic)  
fourth highest bit is the seek bit (Determines if the file supports seeked operations.)  
third lowest bit is the append bit. (Determines if the file can only be appended to.)  

# Async and concurrency.  
In MFS, each client has a queue which represents the requests it has sent. a request can be removed from the query only when it's response is sent back to the user.  
There is no pre-emption. A request is fully processed and request sent when the server decides to process it.  
(For context, messages that are in the queue are called in-flight messages.)  
All messages and files obey FCFS (First come, First serve) concurrency.  

# Retries  
When endpoint A send MFS message p to endpoint B, and endpoint B responds with a CRC verification failure, the same message (deep copy) should be sent again.  
A maximum of 5 retries should be attempted before giving up.

# Paths
Paths in MFS are slash-seperated (/) and must NOT

# Resource-Based Interfaces (RBI)
RBIs are optional interaces that exist as directories and files instead of message types.
## MBRBI (Message Bundling Resource Based Interface)  
(All paths are relative to the directory this RBI is in).  
MBRBI is a type of RBI that is used to bundle critical sets of messages together before execution.  
MBRBI defines the directory structure:
/messages
/reset
/execute
messages is a appending atomic file without seeking. It contains messages that are to be executed when execute file is written to.  
reset is a atomic file without seeking. When written ASCII "1" into it, the messages in the messages file is flushed.  
execute is a atomic file without seeking. When written ASCII "1" into it, the messages in the messages file is flushed and given to the server to execute.
