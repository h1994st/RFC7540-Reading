方便挑刺

## Section 3.2

- Such an HTTP/1.1 request MUST include exactly one HTTP2-Settings (Section 3.2.1) header field.
- Requests that contain a payload body MUST be sent in their entirety before the client can send HTTP/2 frames.
- A server MUST ignore an "h2" token in an Upgrade header field. Presence of a token with "h2" implies HTTP/2 over TLS, which is instead negotiated as described in Section 3.3.
- After the empty line that terminates the 101 response, the server can begin sending HTTP/2 frames.  These frames MUST include a response to the request that initiated the upgrade.
- The first HTTP/2 frame sent by the server MUST be a server connection preface (Section 3.5) consisting of a SETTINGS frame (Section 6.5).
- Upon receiving the 101 response, the client MUST send a connection preface (Section 3.5), which includes a SETTINGS frame.

## Section 3.2.1

- A request that upgrades from HTTP/1.1 to HTTP/2 MUST include exactly one "HTTP2-Settings" header field.
- A server MUST NOT upgrade the connection to HTTP/2 if this header field is not present or if more than one is present. A server MUST NOT send this header field.
- Since the upgrade is only intended to apply to the immediate connection, a client sending the HTTP2-Settings header field MUST also send "HTTP2-Settings" as a connection option in the Connection header field to prevent it from being forwarded (see Section 6.1 of [RFC7230]).

## Section 3.3

- HTTP/2 over TLS uses the "h2" protocol identifier. The "h2c" protocol identifier MUST NOT be sent by a client or selected by a server; the "h2c" protocol identifier describes a protocol that does not use TLS.
- Once TLS negotiation is complete, both the client and the server MUST send a connection preface (Section 3.5).

## Section 3.4

- A client MUST send the connection preface (Section 3.5) and then MAY immediately send HTTP/2 frames to such a server; servers can identify these connections by the presence of the connection preface. (此句中还有个MAY)
- implementations that support HTTP/2 over TLS MUST use protocol negotiation in TLS [TLS-ALPN].
- Likewise, the server MUST send a connection preface (Section 3.5).

## Section 3.5

- That is, the connection preface starts with the string "PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n"). This sequence MUST be followed by a SETTINGS frame (Section 6.5), which MAY be empty. (这句也有MAY)
- The server connection preface consists of a potentially empty SETTINGS frame (Section 6.5) that MUST be the first frame the server sends in the HTTP/2 connection.
- The SETTINGS frames received from a peer as part of the connection preface MUST be acknowledged (see Section 6.5.3) after sending the connection preface.
- Clients and servers MUST treat an invalid connection preface as a connection error (Section 5.4.1) of type PROTOCOL_ERROR. A GOAWAY frame (Section 6.8) MAY be omitted in this case, since an invalid preface indicates that the peer is not using HTTP/2.

## Section 4.1 Frame Format

- Length: ... Values greater than 2^14 (16,384) MUST NOT be sent unless the receiver has set a larger value for SETTINGS\_MAX\_FRAME\_SIZE.
- Type: ... Implementations MUST ignore and discard any frame that has a type that is unknown.
- Flags: ... Flags are assigned semantics specific to the indicated frame type. Flags that have no defined semantics for a particular frame type MUST be ignored and MUST be left unset (0x0) when sending.
- R: The semantics of this bit are undefined, and the bit MUST remain unset (0x0) when sending and MUST be ignored when receiving.

## Section 4.2 Frame Size

- All implementations MUST be capable of receiving and minimally processing frames up to 2^14 octets in length, plus the 9-octet frame header (Section 4.1).
- An endpoint MUST send an error code of FRAME\_SIZE\_ERROR if a frame exceeds the size defined in SETTINGS\_MAX\_FRAME_SIZE, exceeds any limit defined for the frame type, or is too small to contain mandatory frame data.
- A frame size error in a frame that could alter the state of the entire connection MUST be treated as a connection error (Section 5.4.1); this includes any frame carrying a header block (Section 4.3) (that is, HEADERS, PUSH_PROMISE, and CONTINUATION), SETTINGS, and any frame with a stream identifier of 0.

## Section 4.3 Header Compression and Decompression

- A decoding error in a header block MUST be treated as a connection error (Section 5.4.1) of type COMPRESSION_ERROR.
- Each header block is processed as a discrete unit. Header blocks MUST be transmitted as a contiguous sequence of frames, with no interleaved frames of any other type or from any other stream.
- A receiver MUST terminate the connection with a connection error (Section 5.4.1) of type COMPRESSION_ERROR if it does not decompress a header block.

## Section 5.1 Stream States

- idle state:
    - Receiving any frame other than HEADERS or PRIORITY on a stream in this state MUST be treated as a connection error (Section 5.4.1) of type PROTOCOL_ERROR.
- reserved (local) state:
    - An endpoint MUST NOT send any type of frame other than HEADERS, RST_STREAM, or PRIORITY in this state.
    - Receiving any type of frame other than RST\_STREAM, PRIORITY, or WINDOW\_UPDATE on a stream in this state MUST be treated as a connection error (Section 5.4.1) of type PROTOCOL\_ERROR.
- reserved (remote) state:
    - An endpoint MUST NOT send any type of frame other than RST\_STREAM, WINDOW\_UPDATE, or PRIORITY in this state.
    - Receiving any type of frame other than HEADERS, RST\_STREAM, or PRIORITY on a stream in this state MUST be treated as a connection error (Section 5.4.1) of type PROTOCOL\_ERROR.
- half-closed (remote) state:
    - If an endpoint receives additional frames, other than WINDOW\_UPDATE, PRIORITY, or RST\_STREAM, for a stream that is in this state, it MUST respond with a stream error (Section 5.4.2) of type STREAM\_CLOSED.
- closed state:
    - An endpoint MUST NOT send frames other than PRIORITY on a closed stream.
    - An endpoint that receives any frame other than PRIORITY after receiving a RST\_STREAM MUST treat that as a stream error (Section 5.4.2) of type STREAM\_CLOSED.
    - Similarly, an endpoint that receives any frames after receiving a frame with the END\_STREAM flag set MUST treat that as a connection error (Section 5.4.1) of type STREAM\_CLOSED, unless the frame is permitted as described below.
    - Endpoints MUST ignore WINDOW\_UPDATE or RST\_STREAM frames received in this state, though endpoints MAY choose to treat frames that arrive a significant time after sending END\_STREAM as a connection error (Section 5.4.1) of type PROTOCOL\_ERROR.
    - An endpoint MUST ignore frames that it receives on closed streams after it has sent a RST_STREAM frame.

## Section 5.1.1 Stream Identifiers

- Streams initiated by a client MUST use odd-numbered stream identifiers; those initiated by the server MUST use even-numbered stream identifiers.
- The identifier of a newly established stream MUST be numerically greater than all streams that the initiating endpoint has opened or reserved.
- An endpoint that receives an unexpected stream identifier MUST respond with a connection error (Section 5.4.1) of type PROTOCOL\_ERROR.

## Section 5.1.2 Stream Concurrency

- Endpoints MUST NOT exceed the limit set by their peer.
- An endpoint that receives a HEADERS frame that causes its advertised concurrent stream limit to be exceeded MUST treat this as a stream error (Section 5.4.2) of type PROTOCOL\_ERROR or REFUSED\_STREAM.

## Section 5.2.1 Flow-Control Principles

- A sender MUST respect flow-control limits imposed by a receiver.

## Section 5.2.2 Appropriate Use of Flow Control

- When using flow control, the receiver MUST read from the TCP receive buffer in a timely fashion.

## Section 5.3.1 Stream Dependencies

- A stream cannot depend on itself. An endpoint MUST treat this as a stream error (Section 5.4.2) of type PROTOCOL_ERROR.

## Section 5.4.1 Connection Error Handling

- After sending the GOAWAY frame for an error condition, the endpoint MUST close the TCP connection.

## 5.4.2 Stream Error Handling

- A RST\_STREAM is the last frame that an endpoint can send on a stream. The peer that sends the RST\_STREAM frame MUST be prepared to receive any frames that were sent or enqueued for sending by the remote peer.
- To avoid looping, an endpoint MUST NOT send a RST\_STREAM in response to a RST\_STREAM frame.

## 5.5 Extending HTTP/2

- Implementations MUST ignore unknown or unsupported values in all extensible protocol elements.
- Implementations MUST discard frames that have unknown or unsupported types.
- However, extension frames that appear in the middle of a header block (Section 4.3) are not permitted; these MUST be treated as a connection error (Section 5.4.1) of type PROTOCOL_ERROR.
- Extensions that could change the semantics of existing protocol components MUST be negotiated before being used.
- If a setting is used for extension negotiation, the initial value MUST be defined in such a fashion that the extension is initially disabled.

## 6.1 DATA

DATA Frame

- Padding: Padding octets MUST be set to zero when sending.
- DATA frames MUST be associated with a stream.
- If a DATA frame is received whose stream identifier field is 0x0, the recipient MUST respond with a connection error (Section 5.4.1) of type PROTOCOL_ERROR.
- If a DATA frame is received whose stream is not in "open" or "half-closed (local)" state, the recipient MUST respond with a stream error (Section 5.4.2) of type STREAM\_CLOSED.
- If the length of the padding is the length of the frame payload or greater, the recipient MUST treat this as a connection error (Section 5.4.1) of type PROTOCOL_ERROR.

## 6.2 HEADERS

HEADERS Frame

- END\_HEADERS (0x4) flag:
    - A HEADERS frame without the END\_HEADERS flag set MUST be followed by a CONTINUATION frame for the same stream.
    - A receiver MUST treat the receipt of any other type of frame or a frame on a different stream as a connection error (Section 5.4.1) of type PROTOCOL_ERROR.
- HEADERS frames MUST be associated with a stream.
- If a HEADERS frame is received whose stream identifier field is 0x0, the recipient MUST respond with a connection error (Section 5.4.1) of type PROTOCOL_ERROR.
- Padding that exceeds the size remaining for the header block fragment MUST be treated as a PROTOCOL_ERROR.

## 6.3 PRIORITY

PRIORITY Frame

- If a PRIORITY frame is received with a stream identifier of 0x0, the recipient MUST respond with a connection error (Section 5.4.1) of type PROTOCOL_ERROR.
- A PRIORITY frame with a length other than 5 octets MUST be treated as a stream error (Section 5.4.2) of type FRAME\_SIZE\_ERROR.

## 6.4 RST_STREAM

RST_STREAM Frame

- After receiving a RST_STREAM on a stream, the receiver MUST NOT send additional frames for that stream, with the exception of PRIORITY.
- However, after sending the RST\_STREAM, the sending endpoint MUST be prepared to receive and process additional frames sent on the stream that might have been sent by the peer prior to the arrival of the RST\_STREAM.
- RST_STREAM frames MUST be associated with a stream.
- If a RST\_STREAM frame is received with a stream identifier of 0x0, the recipient MUST treat this as a connection error (Section 5.4.1) of type PROTOCOL\_ERROR.
- RST_STREAM frames MUST NOT be sent for a stream in the "idle" state.
- If a RST\_STREAM frame identifying an idle stream is received, the recipient MUST treat this as a connection error (Section 5.4.1) of type PROTOCOL\_ERROR.
- A RST\_STREAM frame with a length other than 4 octets MUST be treated as a connection error (Section 5.4.1) of type FRAME\_SIZE\_ERROR.

## 6.5 SETTINGS

SETTINGS Frame

- A SETTINGS frame MUST be sent by both endpoints at the start of a connection and MAY be sent at any other time by either endpoint over the lifetime of the connection. (此句有MAY)
- Implementations MUST support all of the parameters defined by this specification.
- ACK (0x1) flag:
    - When this bit is set, the payload of the SETTINGS frame MUST be empty.
    - Receipt of a SETTINGS frame with the ACK flag set and a length field value other than 0 MUST be treated as a connection error (Section 5.4.1) of type FRAME\_SIZE\_ERROR.
- The stream identifier for a SETTINGS frame MUST be zero (0x0).
- If an endpoint receives a SETTINGS frame whose stream identifier field is anything other than 0x0, the endpoint MUST respond with a connection error (Section 5.4.1) of type PROTOCOL_ERROR.
- A badly formed or incomplete SETTINGS frame MUST be treated as a connection error (Section 5.4.1) of type PROTOCOL_ERROR.
- A SETTINGS frame with a length other than a multiple of 6 octets MUST be treated as a connection error (Section 5.4.1) of type FRAME\_SIZE\_ERROR.

## 6.5.2 Defined SETTINGS Parameters

- SETTINGS\_ENABLE\_PUSH (0x2) parameter:
    - An endpoint MUST NOT send a PUSH_PROMISE frame if it receives this parameter set to a value of 0.
    - An endpoint that has both set this parameter to 0 and had it acknowledged MUST treat the receipt of a PUSH\_PROMISE frame as a connection error (Section 5.4.1) of type PROTOCOL\_ERROR.
    - The initial value is 1, which indicates that server push is permitted.  Any value other than 0 or 1 MUST be treated as a connection error (Section 5.4.1) of type PROTOCOL_ERROR.
- SETTINGS\_INITIAL\_WINDOW_SIZE (0x4) parameter:
    - Values above the maximum flow-control window size of 2^31-1 MUST be treated as a connection error (Section 5.4.1) of type FLOW\_CONTROL\_ERROR.
- SETTINGS\_MAX\_FRAME_SIZE (0x5) parameter:
    - The initial value is 2^14 (16,384) octets. The value advertised by an endpoint MUST be between this initial value and the maximum allowed frame size (2^24-1 or 16,777,215 octets), inclusive.
    - Values outside this range MUST be treated as a connection error (Section 5.4.1) of type PROTOCOL_ERROR.
- An endpoint that receives a SETTINGS frame with any unknown or unsupported identifier MUST ignore that setting.

## 6.5.3 Settings Synchronization

- In order to provide such synchronization timepoints, the recipient of a SETTINGS frame in which the ACK flag is not set MUST apply the updated parameters as soon as possible upon receipt.
- The values in the SETTINGS frame MUST be processed in the order they appear, with no other frame processing between values.
- Unsupported parameters MUST be ignored.
- Once all values have been processed, the recipient MUST immediately emit a SETTINGS frame with the ACK flag set.

## 6.6 PUSH_PROMISE

PUSH_PROMISE Frame

- Promised Stream ID: The promised stream identifier MUST be a valid choice for the next stream sent by the sender (see "new stream identifier" in Section 5.1.1).
- END_HEADERS (0x4) flag:
    - A PUSH\_PROMISE frame without the END\_HEADERS flag set MUST be followed by a CONTINUATION frame for the same stream.
    - A receiver MUST treat the receipt of any other type of frame or a frame on a different stream as a connection error (Section 5.4.1) of type PROTOCOL_ERROR.
- PUSH_PROMISE frames MUST only be sent on a peer-initiated stream that is in either the "open" or "half-closed (remote)" state.
- If the stream identifier field specifies the value 0x0, a recipient MUST respond with a connection error (Section 5.4.1) of type PROTOCOL_ERROR.
- PUSH\_PROMISE MUST NOT be sent if the SETTINGS\_ENABLE\_PUSH setting of the peer endpoint is set to 0. An endpoint that has set this setting and has received acknowledgement MUST treat the receipt of a PUSH\_PROMISE frame as a connection error (Section 5.4.1) of type PROTOCOL\_ERROR.
- A sender MUST NOT send a PUSH_PROMISE on a stream unless that stream is either "open" or "half-closed (remote)"; the sender MUST ensure that the promised stream is a valid choice for a new stream identifier (Section 5.1.1) (that is, the promised stream MUST be in the "idle" state).
- A receiver MUST treat the receipt of a PUSH\_PROMISE on a stream that is neither "open" nor "half-closed (local)" as a connection error (Section 5.4.1) of type PROTOCOL\_ERROR.
- However, an endpoint that has sent RST\_STREAM on the associated stream MUST handle PUSH\_PROMISE frames that might have been created before the RST\_STREAM frame is received and processed.
- A receiver MUST treat the receipt of a PUSH\_PROMISE that promises an illegal stream identifier (Section 5.1.1) as a connection error (Section 5.4.1) of type PROTOCOL\_ERROR.

## 6.7 PING

PING Frame

- In addition to the frame header, PING frames MUST contain 8 octets of 
opaque data in the payload.
- Receivers of a PING frame that does not include an ACK flag MUST send a PING frame with the ACK flag set in response, with an identical payload.
- ACK (0x1) flag:
    - When set, bit 0 indicates that this PING frame is a PING response. An endpoint MUST set this flag in PING responses.
    - An endpoint MUST NOT respond to PING frames containing this flag.
- If a PING frame is received with a stream identifier field value other than 0x0, the recipient MUST respond with a connection error (Section 5.4.1) of type PROTOCOL_ERROR.
- Receipt of a PING frame with a length field value other than 8 MUST be treated as a connection error (Section 5.4.1) of type FRAME\_SIZE\_ERROR.

























