# csrmesh
Reverse engineered bearer/bridge implementation of the CSRMesh BTLE protocol including all necessary cryptographic and packing routines required to send valid packets over a CSRMesh BTLE network. This particular implementation currently only supports the CSRMesh lighting API and has been succssfully tested with a pair of Feit HomeBrite smart LED bulbs. However, the implemtation of other CSRMesh APIs should be quite straightformard if needed. This implementation now supports sending packets to multi-device mesh networks as well as addressing devices by either device ID or group ID.

# Tested Devices
 * Feit HomeBrite A19 Household, Model AOM800/827/LED/HBR with a max of 254 bulbs(theoretically) per network PIN in a given area

# Requirements
 * Python 2.7.x and 3.x (preferred)
 * Recent version of bluez (requires gatttool binary)
 * pycrypto

# Usage
    csrmesh-cli [-h] --pin (4 digit pin) --dest (destination BTLE address) 
    [--level LEVEL] [--red RED] [--green GREEN] [--blue BLUE] [--objid OBJID]

All levels are represented using numbers from 0 (off) to 255 (maximum)

Object ID's are specified via the following syntax:
 * An object ID of zero results in a broadcast message. This is the default.
 * Positive object ID's will be interpreted as an individual device ID. Device indicies begin at 1.
 * Negative object ID's will be interpreted as group numbers.
 
Addressing examples:

    Bulb/Device 5 -> --objid 5 
    Group 3       -> --objid -3
    Broadcast     -> --objid 0 (or omit the --objid option)

# Protocol Documentation
## Network Key Derivation
The 128 bit network key used in CSRMesh networks is derived by concatenating the ASCII representation of the PIN with a null byte and the string 'MCP', computing the SHA256 hash of the string, reversing the order of the bytes in the resulting hash, and taking the first 16 bytes as the key.

## Forming Authenticated Packets
Packets sent to CSRMesh devices require authentication as well as encryption. All multibyte types are represented in little endian format. To form a valid packet, the sequence number, the constant 0x0080, and N null bytes where N is equal to the size of the payload to be sent are concatenated together and encrypted using the network key using AES-128 in ECB mode. The result of such encryption is then truncated to the length of the payload to be sent. The payload is XOR'ed with the result of the previous encryption to form the encrypted payload. Next, the message authentication code is computed using HMAC-SHA256 using the network key as the secret of the following data: 8 null bytes, sequence number, constant 0x80 and encrypted payload. The order of the bytes in the resulting hash are then reversed and the hash truncated to 8 bytes. Finally, the packet is formed by contatenating the sequence number, constant 0x80, encrypted payload, truncated HMAC, and the constant 0xff.

## Sending packets
Packets are then sent via the Bluetooth LE GATT protocol. To send a packet, the data must be written to two write only handles, 0x0011 and 0x0014. The first 20 bytes of the packet are first written to handle 0x0011 and then the remainder of the packet is written to handle 0x0014.
