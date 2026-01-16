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
However, this comes at the risk of double the chance of ID collissions. Due to this, both sides must take care to not use the same ID as one another.  
  
# Base headers  
Base headers (unchanging accross versions) are defined as:  
    len[4], id[4], version[2], type[2], crc[4]  
len is the total lenght of the entire MFS message (Including itself). (Little endian).  
id is the message id of the message. (Little endian). (ID collissions are allowed as long as the colliding messages are at least 10 seconds apart.)  
version is the MFS version of the message. (Little Endian). (Details in a later section.)  
type is the request type. Explained in a later section. (Little endian).  
crc is the crc of the entire message. (CRC should be computed as if it was filled as zeros during computation. CRC computations beings from the start of base headers).  

# Version  
The highest 8 bits are major version.  
The lower 8 bits are minor version.  
This is MFSv2. Which makes its version:  
    `0x0200` in hexidecimal.  

# Message types
These are additional headers that are appended to the base headers, as the type directs.
MFSv2 supporting servers must implement the following message types (Numbers in decimal):  
0: noop. Adds no additional headers. No operation.  
1: get_size. Adds no additional headers.  Gets the maximum message lenght the server can support.  
2: get_async_size. Adds no additional headers.  Gets the maximum amount of messages the server can handle. (Maximum in-flight messages).  
3: get_attributes. `path_len[4], path[path_len]`. Gets the attributes of given file at the path.  path_len is a little endian uint32 defining the lenght of the path, which is a ASCII string.  
4: set_p2p_mode. Adds no additional headers. Swiches the connection into a peer-to-peer mode as defined above.  
5: seek_read. `path_len[4], path[path_len], offset[4], amount[4]`
