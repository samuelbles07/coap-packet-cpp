# CoapPacket Library

A lightweight, platform-agnostic C++ library for building and parsing CoAP (Constrained Application Protocol) UDP packets.

## Overview

CoapPacket is a minimal CoAP implementation focused solely on packet construction and parsing. Unlike full CoAP stacks, this library:

- **Only handles packet building and parsing** - no networking, no state management
- **Platform agnostic** - pure C++ with no hardware dependencies
- **Builder pattern API** - fluent, easy-to-use interface
- **Zero external dependencies** - only uses C++ standard library (STL)
- **Lightweight** - extracted from lobaro-coap, keeping only the essentials

Perfect for embedded systems or applications that need CoAP packet handling without the full protocol stack overhead.

## Features

- ✅ Build CoAP packets using fluent builder pattern
- ✅ Parse CoAP packets from UDP datagrams
- ✅ Support for all CoAP message types (CON, NON, ACK, RST)
- ✅ Support for all standard CoAP codes (GET, POST, PUT, DELETE, response codes)
- ✅ CoAP option handling with automatic delta encoding/decoding
- ✅ Automatic option sorting
- ✅ Convenience methods for common operations (Uri-Path, Uri-Query, Content-Format)
- ✅ Comprehensive error handling without exceptions or templates
- ✅ RFC 7252 compliant

## Requirements

- **C++11 or later**
- **No external dependencies** (uses only STL)

## Installation

Simply copy the `lib/coap-packet` directory into your project and include the headers.

### Directory Structure

```
lib/coap-packet/
├── include/
│   └── coap-packet/
│       ├── CoapBuilder.h      # Builder pattern API
│       ├── CoapParser.h       # Parser API
│       ├── CoapPacket.h       # Packet structure
│       ├── CoapTypes.h        # Enums and constants
│       └── CoapError.h        # Error codes
├── src/
│   ├── CoapBuilder.cpp
│   └── CoapParser.cpp
└── examples/
    └── basic_usage.cpp
```

## Quick Start

### Building a CoAP GET Request

```cpp
#include "coap-packet/CoapBuilder.h"

using namespace CoapPacket;

CoapBuilder builder;
std::vector<uint8_t> udpBuffer;

uint8_t token[] = {0x12, 0x34};

CoapError err = builder
    .setType(CoapType::CON)
    .setCode(CoapCode::GET)
    .setMessageId(1234)
    .setToken(token, 2)
    .setUriPath("/sensors/temp")
    .buildBuffer(udpBuffer);

if (err == CoapError::OK) {
    // Send udpBuffer via UDP socket
}
```

### Building a POST Request with Payload

```cpp
CoapBuilder builder;
std::vector<uint8_t> udpBuffer;

uint8_t token[] = {0xAB, 0xCD};
std::string jsonData = "{\"temperature\":22.5}";

CoapError err = builder
    .setType(CoapType::CON)
    .setCode(CoapCode::POST)
    .setMessageId(5678)
    .setToken(token, 2)
    .setUriPath("/sensors/data")
    .setContentFormat(CoapContentFormat::JSON)
    .setPayload(jsonData)
    .buildBuffer(udpBuffer);
```

### Parsing a CoAP Response

```cpp
#include "coap-packet/CoapParser.h"

using namespace CoapPacket;

// Received UDP datagram
uint8_t* udpData = ...; // from socket
size_t udpLength = ...;

CoapPacket packet;
CoapError err = CoapParser::parse(udpData, udpLength, packet);

if (err == CoapError::OK) {
    // Access packet data
    uint16_t msgId = packet.message_id;
    CoapCode code = packet.code;

    // Check if it's a successful response
    if (code == CoapCode::CONTENT_2_05) {
        std::string payload(packet.payload.begin(), packet.payload.end());
        std::cout << "Received: " << payload << std::endl;
    }
}
```

## API Reference

### CoapBuilder

Fluent builder for constructing CoAP packets.

#### Core Methods

```cpp
CoapBuilder& setType(CoapType type);
CoapBuilder& setCode(CoapCode code);
CoapBuilder& setMessageId(uint16_t id);
CoapBuilder& setToken(const uint8_t* token, uint8_t length);
```

#### Option Methods

```cpp
// Add raw option
CoapBuilder& addOption(CoapOptionNumber optionNum, const std::vector<uint8_t>& value);

// Add string option
CoapBuilder& addOption(CoapOptionNumber optionNum, const std::string& value);

// Add uint option
CoapBuilder& addOption(CoapOptionNumber optionNum, uint32_t value);
```

#### Convenience Methods

```cpp
// Automatically splits path on '/' and creates multiple URI_PATH options
CoapBuilder& setUriPath(const std::string& path);

// Add single URI path segment
CoapBuilder& addUriPathSegment(const std::string& segment);

// Add URI query parameter (creates "key=value")
CoapBuilder& addUriQuery(const std::string& key, const std::string& value);

// Set content format option
CoapBuilder& setContentFormat(CoapContentFormat format);
```

#### Payload Methods

```cpp
CoapBuilder& setPayload(const std::vector<uint8_t>& data);
CoapBuilder& setPayload(const std::string& data);
CoapBuilder& setPayload(const uint8_t* data, size_t length);
```

#### Build Methods

```cpp
// Build packet structure
CoapError build(CoapPacket& packet);

// Build UDP buffer directly (ready to send)
CoapError buildBuffer(std::vector<uint8_t>& buffer);

// Check last error
CoapError getLastError() const;

// Reset builder to initial state
void reset();
```

### CoapParser

Static parser for CoAP packets.

```cpp
// Parse from raw buffer
static CoapError parse(const uint8_t* buffer, size_t length, CoapPacket& packet);

// Parse from vector
static CoapError parse(const std::vector<uint8_t>& buffer, CoapPacket& packet);
```

### CoapPacket

Represents a complete CoAP packet.

```cpp
struct CoapPacket {
    uint8_t version;              // Always 1
    CoapType type;                // CON, NON, ACK, RST
    uint8_t token_length;         // 0-8
    uint8_t token[8];             // Token bytes
    CoapCode code;                // Request/response code
    uint16_t message_id;          // Message ID
    std::vector<CoapOption> options;  // Options list
    std::vector<uint8_t> payload;     // Payload data

    // Helper methods
    const uint8_t* getTokenPtr() const;
    const uint8_t* getPayloadPtr() const;
    size_t getPayloadSize() const;
    void setToken(const uint8_t* tokenData, uint8_t length);
    void clear();
};
```

### CoapOption

Represents a single CoAP option.

```cpp
struct CoapOption {
    uint16_t number;              // Option number
    std::vector<uint8_t> value;   // Option value
};
```

### Error Codes

```cpp
enum class CoapError {
    OK = 0,
    DATAGRAM_TOO_SHORT,
    INVALID_VERSION,
    INVALID_TOKEN_LENGTH,
    INVALID_CODE_CLASS,
    INVALID_FORMAT,
    TOO_MANY_OPTIONS,
    OPTION_TOO_LONG,
    PAYLOAD_TOO_LARGE,
    MISSING_REQUIRED_FIELD,
    INVALID_OPTION_NUMBER,
    BUFFER_TOO_SMALL,
    OUT_OF_MEMORY,
    INVALID_ARGUMENT
};

// Get human-readable error message
const char* getErrorMessage(CoapError error);
```

## CoAP Types and Codes

### Message Types

```cpp
enum class CoapType : uint8_t {
    CON = 0,  // Confirmable
    NON = 1,  // Non-confirmable
    ACK = 2,  // Acknowledgment
    RST = 3   // Reset
};
```

### Common Request Codes

```cpp
enum class CoapCode : uint8_t {
    GET = 1,
    POST = 2,
    PUT = 3,
    DELETE = 4,
    // ... more codes
};
```

### Common Response Codes

```cpp
CONTENT_2_05              // Success with content
CREATED_2_01              // Resource created
DELETED_2_02              // Resource deleted
CHANGED_2_04              // Resource changed
BAD_REQUEST_4_00          // Client error
NOT_FOUND_4_04            // Resource not found
INTERNAL_SERVER_ERROR_5_00 // Server error
```

### Option Numbers

```cpp
enum class CoapOptionNumber : uint16_t {
    URI_HOST = 3,
    URI_PORT = 7,
    URI_PATH = 11,
    CONTENT_FORMAT = 12,
    URI_QUERY = 15,
    ACCEPT = 17,
    // ... more options
};
```

### Content Formats

```cpp
enum class CoapContentFormat : uint16_t {
    TEXT_PLAIN = 0,
    LINK_FORMAT = 40,
    XML = 41,
    OCTET_STREAM = 42,
    JSON = 50,
    CBOR = 60
};
```

## Examples

### Example 1: GET Request with Query Parameters

```cpp
CoapBuilder builder;
std::vector<uint8_t> buffer;

uint8_t token[] = {0x01};

CoapError err = builder
    .setType(CoapType::CON)
    .setCode(CoapCode::GET)
    .setMessageId(100)
    .setToken(token, 1)
    .setUriPath("/api/data")
    .addUriQuery("start", "0")
    .addUriQuery("limit", "10")
    .buildBuffer(buffer);

// buffer is ready to send via UDP
```

### Example 2: Responding to a Request

```cpp
// Parse incoming request
CoapPacket request;
CoapParser::parse(udpData, udpLength, request);

// Build response with same token and message ID
CoapBuilder respBuilder;
std::vector<uint8_t> respBuffer;

std::string responseData = "42";

CoapError err = respBuilder
    .setType(CoapType::ACK)
    .setCode(CoapCode::CONTENT_2_05)
    .setMessageId(request.message_id)  // Echo message ID
    .setToken(request.token, request.token_length)  // Echo token
    .setContentFormat(CoapContentFormat::TEXT_PLAIN)
    .setPayload(responseData)
    .buildBuffer(respBuffer);
```

### Example 3: Extracting URI Path from Parsed Packet

```cpp
CoapPacket packet;
CoapParser::parse(udpData, udpLength, packet);

// Extract all URI_PATH options
std::vector<std::string> pathSegments;
for (const auto& opt : packet.options) {
    if (opt.number == static_cast<uint16_t>(CoapOptionNumber::URI_PATH)) {
        std::string segment(opt.value.begin(), opt.value.end());
        pathSegments.push_back(segment);
    }
}

// Reconstruct full path
std::string fullPath = "/";
for (const auto& seg : pathSegments) {
    fullPath += seg + "/";
}
```

## CoAP Packet Format

The library handles the complete CoAP packet format as defined in RFC 7252:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Ver| T |  TKL  |      Code     |          Message ID           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Token (if any, TKL bytes) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Options (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1 1 1 1 1 1 1 1|    Payload (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

- **Ver**: Version (always 1)
- **T**: Type (2 bits)
- **TKL**: Token Length (4 bits, 0-8)
- **Code**: Request/Response Code (8 bits)
- **Message ID**: 16-bit identifier
- **Token**: 0-8 bytes
- **Options**: Delta-encoded options
- **0xFF**: Payload marker (if payload present)
- **Payload**: Up to 1024 bytes

## Implementation Details

### Option Delta Encoding

Options are automatically sorted by option number and encoded using delta encoding:

- **Delta < 13**: Stored directly in 4-bit field
- **Delta 13-268**: Field = 13, followed by 1-byte value (delta - 13)
- **Delta 269+**: Field = 14, followed by 2-byte value (delta - 269)

The same encoding applies to option length.

### Validation

The builder automatically validates:
- Token length ≤ 8 bytes
- Valid code class (1, 6, 7 are reserved)
- Payload size ≤ 1024 bytes
- Empty messages have no token, options, or payload

The parser validates:
- Minimum packet size (4 bytes)
- CoAP version = 1
- Token length ≤ 8 bytes
- Valid code class
- Option and payload sizes

## Error Handling

All functions return `CoapError` enum values:

```cpp
CoapError err = builder.buildBuffer(buffer);
if (err != CoapError::OK) {
    std::cerr << "Error: " << getErrorMessage(err) << std::endl;
}
```

No exceptions are thrown, making it safe for embedded systems.

## Comparison with lobaro-coap

This library is extracted from [lobaro-coap](https://github.com/Lobaro/lobaro-coap) but focuses solely on packet handling:

| Feature | lobaro-coap | CoapPacket |
|---------|-------------|------------|
| Packet building | ✅ | ✅ |
| Packet parsing | ✅ | ✅ |
| Networking | ✅ | ❌ |
| Confirmable messages | ✅ | ❌ |
| Blockwise transfer | ✅ | ❌ |
| Observe | ✅ | ❌ |
| Resource handlers | ✅ | ❌ |
| Language | C | C++ |
| API Style | C functions | Builder pattern |

**Use CoapPacket when**: You only need to build/parse CoAP packets and handle the protocol logic yourself.

**Use lobaro-coap when**: You need a complete CoAP client/server implementation.

## License

This library is derived from lobaro-coap which is licensed under the MIT License.

Copyright (c) 2015 Dipl.-Ing. Tobias Rohde, http://www.lobaro.com

## References

- [RFC 7252: The Constrained Application Protocol (CoAP)](https://tools.ietf.org/html/rfc7252)
- [lobaro-coap GitHub Repository](https://github.com/Lobaro/lobaro-coap)

## Contributing

This library is designed to be minimal and focused. If you need additional features, consider using the full lobaro-coap library or implementing them in your application layer.
