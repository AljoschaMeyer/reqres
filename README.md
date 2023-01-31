# Reqres

A minimalistic request-response specification.

**Status: freshly specified, needs more time before it can be declared stable.**

## What, Why, and How?

This specification deals with some design issues of request-response protocols over a reliable, ordered, byte oriented channel, namely the mapping from responses to concurrent requests, and flow control. Many request-response-based protocols neglect flow control, leading either to exhaustion of computational resources, or delay in transmitting some responses just because a completely unrelated resource is currently unavailable.

Reqres is an asymmetric protocol; the *client* endpoint issues *requests*, and the *server* endpoint serves *responses* - you can emulate a symmetric session by running two reqres sessions in parallel. When issuing a request, the client annotates it with a 64-bit integer, the *request id*. Each response is annotated with the id of the request to which it pertains.

Flow control is based on a credit-based scheme, very similar to that of [minmux](https://github.com/AljoschaMeyer/minmux). The remainder of this text assumes familiarity with the [minmux](https://github.com/AljoschaMeyer/minmux) specification. Like [minmux](https://github.com/AljoschaMeyer/minmux), a reqres session consists of exchanging different packets, each consisting of a light header followed by packet-specific data.

The specifics of the flow control mechanism employed by reqres depend on whether requests and/or responses are of a static size, or whether any (and which) of them include streaming data of arbitrary length. Reqres is actually a family of four related protocols, one protocol for each combination of static or streaming requests or responses.

## Static Requests and Responses

We first describe the simple case of both requests and responses being of finite size. For example, a request might consist of a fixed-width user identifier, and the response might be a 64-bit timestamp of that user's last activity. More generally, we fix a `Request` type, and a `Response` type, both inhabited by only finitely many values (i.e., the encoding of any value has a well-known maximum size).

In such a setting, the endpoints maintain a logical *request channel* from the client to the server for sending request. Flow control on this channel is done in units of complete requests. Endpoints maintain the channel similarly to a [minmux](https://github.com/AljoschaMeyer/minmux) data stream, with the client sending `Write` and `ForgoCredit` packets, and the server sending `GiveCredit` and `Oops` packets. This channel is never closed, so (unlike with [minmux](https://github.com/AljoschaMeyer/minmux)) there is no dedicated last item type, and no `StopWrite` or `StopRead` packets are needed. Furthermore, there are no `RequestCredit` and `RequestItems` packets, because we assume an unbounded sequence of requests and corresponding responses.

A similar logical *response channel* from the server to the client handles responses. Flow control is done in units of complete responses, with the server sending `Write` and `ForgoCredit` packets, and the client sending `GiveCredit` and `Oops` packets.

In addition to these two channels, clients may cancel request by sending `CancelRequest` packets containing a request id. There is no flow control for cancellations, the server must allocate the necessary state for dealing with a request's cancellation together with the state for responding to the request. The server must ignore cancellations with an id that does not match any currently active request (this is the easiest way of preventing problems when the client cancels a request concurrently to the server responding to it). Otherwise, the server should send a response for the indicated request as quickly as possible. Usually, the response type is a [sum type](https://en.wikipedia.org/wiki/Tagged_union) with a variant for cancellation that carries no further information; the server would send this variant when receiving a cancellation.

When issuing new request, clients should use *fresh* ids. Initially, the client considers all ids to be fresh. Using an id for a request turns it non-fresh, until the client receives a response of that id. In particular, canceling a request does not make its id freshDaddy). Servers may assume incoming requests to have fresh ids, they do not need to check whether the id is indeed fresh. It is the client's responsibility to not mess things up.

## Streaming Data

We now consider the case where responses consist of streaming data. One example is access to a content-addressed storage, where requests consist of a hash of fixed size, but responses are byte strings of arbitrary length. More generally, instead of a single `Response` type, we know define responses via three finite types: `ResponseFirst`, the first piece of data belonging to a response; `ResponseRepeated`, the parts of a response that can be repeated an arbitrary number of times; and `ResponseLast`, the last piece of data belonging to a response. In the preceding example, both `ResponseFirst` and `ResponseLast` are the [unit type](https://en.wikipedia.org/wiki/Unit_type), and `ResponseRepeated` is the type of individual bytes.

The first and last items of a response are sent over the response channel. Only the first item consumes credit. Canceling a request now has the semantics of asking the server to send a last item for the corresponding response as soon as possible (which still may only be done after having transmitted a first item for that response). After sending the first but before sending the last item, an endpoint can send an arbitrary number of repeated items. These items need a different form of flow control, because they should not block the concurrent transmission of new request.

Maintaining separate credit for the repeated items of every individual response would be inefficient, because the client would have to divide its available resources amongst its request. Instead, reqres manages flow control of repeated items for all responses collectively, the server gets to decide which responses to prioritize. This writer-driven, unified flow control is the main service provided by reqres, and the main distinguishing feature from [minmux](https://github.com/AljoschaMeyer/minmux).

There is a logical *streaming response channel* from the server to the client; the server sends the repeated items of all responses over that channel. This channel is never closed, so it is maintained via `Write` and `ForgoCredit` packets from the server, and `GiveCredit` and `Oops` from the client. Data written over this channel is not explicitly tagged with a response id, instead, there is at any point in time exactly one *active response id*, and all items written on the streaming response channel pertain to the request with the currently active response id. The server can set the active response id with a `SetActiveResponse` packet. Only ids for which a first item but no last item has been transmitted on the response shall channel are allowed.

Both the sending of repeated response items and of `SetActiveResponse` are limited through the credit given on the streaming response channel, the unit of credit are the individual bytes of the `Write` and `SetActiveResponse` packets respectively. Thus, the client can maintain a single byte buffer into which such incoming packets can be copied for later processing.

Similarly to streaming responses, minmux also supports streaming request. In those settings, instead of a single `Request` type, there are three finite types: `RequestFirst`, `RequestRepeated` and `RequestLast`. `RequestFirst` and `RequestLast` are sent over the request channel, flow control works analogously to the response counterparts. Also analogously, there is a *streaming request channel* from the client to the server, over which the repeated items of all request are sent. Repeated items are routed to the current *active request id*, which can be set via `SetActiveRequest` packets. Repeated request items and `SetActiveRequest` packets share byte-wise credit on the streaming request channel.

Additionally, the server can cancel a streaming request via a `CancelResponse` packet, to which the client should respond by sending a last request item for the indicated request as soon as possible.

## Protocols

There are four possible combinations of static or streaming requests and/or responses, each leading to a particular protocol, according to the preceeding introduction. We now give for each such protocol a self-contained overview over its parameters, a summary of the different packets it uses, and specific encodings for sending these packets over a reliable, ordered, byte-oriented channel.

Packets always use the same format: they start with a single byte header whose most significant bits (the exact number depends on the protocol variant and the packet type) act as a *tag* that indicates the packet type. The remaining header bits are used for encoding a 64-bit integer, whose meaning differs from packet to packet. If all non-tag bits are set to 1, the header is followed by a [VarGtX64](https://github.com/AljoschaMeyer/varu64#greater-than-x-unsigned-integers) encoding the integer, where `X := (2^k) - 2` and `k` is the number of non-tag bits. If they are not all set to 1, the non-tag bits encode the integer directly. For some packet types, the integer is a nonzero integer. In those cases, the bits encode the predecessor of the actual integer (`1` is encoded as a `0`, `2` as a `1`, and so on), and if they are all set to 1, the header is followed by a [VarGtX64](https://github.com/AljoschaMeyer/varu64#greater-than-x-unsigned-integers) encoding the integer, with `X := (2^k) - 1` and `k` is the number of non-tag bits. The header (or integer if applicable) is then followed directly by the encoding of packet-specific data, if there is any.

### Static Requests, Static Responses

In order to specify an instance of reqres with static requests and static responses, the following information must be given:

- What is the type of requests?
  - How are they encoded?
- What is the type of responses?
  - How are they encoded?

#### Client Packets

| Packet | Tag | Integer | Remaining data |
|---|---|---|---|
| RequestWrite | 00 | The id of the request. | The encoding of the request. |
| RequestForgoCredit | 01 | The amount of credit to forgo (nonzero). | - |
| ResponseGiveCredit | 10 | The amount of credit to give (nonzero). | - |
| ResponseOops | 110 | The maximum amount of credit to retain. | - |
| CancelRequest | 111 | The id of the request. | - |

#### Server Packets

| Packet | Tag | Integer | Remaining data |
|---|---|---|---|
| ResponseWrite | 00 | The id of the corresponding request. | The encoding of the response. |
| ResponseForgoCredit | 01 | The amount of credit to forgo (nonzero). | - |
| RequestGiveCredit | 10 | The amount of credit to give (nonzero). | - |
| RequestOops | 11 | The maximum amount of credit to retain. | - |

### Static Requests, Streaming Responses

In order to specify an instance of reqres with static requests and streaming responses, the following information must be given:

- What is the type of requests?
  - How are they encoded?
- What is the type of first response items?
  - How are they encoded?
- What is the type of repeated response items?
  - How are they encoded?
- What is the type of last response items?
  - How are they encoded?

#### Client Packets

| Packet | Tag | Integer | Remaining data |
|---|---|---|---|
| RequestWrite | 000 | The id of the request. | The encoding of the request. |
| RequestForgoCredit | 001 | The amount of credit to forgo (nonzero). | - |
| ResponseGiveCredit | 010 | The amount of credit to give (nonzero). | - |
| ResponseOops | 011 | The maximum amount of credit to retain. | - |
| CancelRequest | 100 | The id of the request. | - |
| ResponseRepeatedGiveCredit | 101 | The amount of credit to give (nonzero). | - |
| ResponseRepeatedOops | 110 | The maximum amount of credit to retain. | - |

#### Server Packets

| Packet | Tag | Integer | Remaining data |
|---|---|---|---|
| ResponseWrite | 00 | The id of the corresponding request. | The encoding of the response. |
| ResponseForgoCredit | 001 | The amount of credit to forgo (nonzero). | - |
| RequestGiveCredit | 010 | The amount of credit to give (nonzero). | - |
| RequestOops | 011 | The maximum amount of credit to retain. | - |
| ResponseRepeatedWrite | 100 | The number of repeated items (nonzero) | The concatenation of the encodings of each item. |
| ResponseRepeatedForgoCredit | 101 | The amount of credit to forgo (nonzero). | - |
| ResponseSetActive | 110 | The request id. | - |

### Streaming Requests, Static Responses

In order to specify an instance of reqres with streaming requests and static responses, the following information must be given:

- What is the type of first request items?
  - How are they encoded?
- What is the type of repeated request items?
  - How are they encoded?
- What is the type of last request items?
  - How are they encoded?
- What is the type of responses?
  - How are they encoded?

#### Client Packets

| Packet | Tag | Integer | Remaining data |
|---|---|---|---|
| RequestWrite | 000 | The id of the request. | The encoding of the request. |
| RequestForgoCredit | 001 | The amount of credit to forgo (nonzero). | - |
| ResponseGiveCredit | 010 | The amount of credit to give (nonzero). | - |
| ResponseOops | 011 | The maximum amount of credit to retain. | - |
| CancelRequest | 100 | The id of the request. | - |
| RequestRepeatedWrite | 101 | The number of repeated items (nonzero) | The concatenation of the encodings of each item. |
| RequestRepeatedForgoCredit | 110 | The amount of credit to forgo (nonzero). | - |
| RequestSetActive | 111 | The request id. | - |

#### Server Packets

| Packet | Tag | Integer | Remaining data |
|---|---|---|---|
| ResponseWrite | 00 | The id of the corresponding request. | The encoding of the response. |
| ResponseForgoCredit | 001 | The amount of credit to forgo (nonzero). | - |
| RequestGiveCredit | 010 | The amount of credit to give (nonzero). | - |
| RequestOops | 011 | The maximum amount of credit to retain. | - |
| CancelResponse | 100 | The id of the request. | - |
| RequestRepeatedGiveCredit | 101 | The amount of credit to give (nonzero). | - |
| RequestRepeatedOops | 110 | The maximum amount of credit to retain. | - |

### Streaming Requests, Streaming Responses

In order to specify an instance of reqres with streaming requests and streaming responses, the following information must be given:

- What is the type of first request items?
  - How are they encoded?
- What is the type of repeated request items?
  - How are they encoded?
- What is the type of last request items?
  - How are they encoded?
- What is the type of first response items?
  - How are they encoded?
- What is the type of repeated response items?
  - How are they encoded?
- What is the type of last response items?
  - How are they encoded?

#### Client Packets

| Packet | Tag | Integer | Remaining data |
|---|---|---|---|
| RequestWrite | 000 | The id of the request. | The encoding of the request. |
| RequestForgoCredit | 001 | The amount of credit to forgo (nonzero). | - |
| ResponseGiveCredit | 010 | The amount of credit to give (nonzero). | - |
| ResponseOops | 0110 | The maximum amount of credit to retain. | - |
| CancelRequest | 0111 | The id of the request. | - |
| RequestRepeatedWrite | 100 | The number of repeated items (nonzero) | The concatenation of the encodings of each item. |
| RequestRepeatedForgoCredit | 1010 | The amount of credit to forgo (nonzero). | - |
| ResponseRepeatedOops | 1011 | The maximum amount of credit to retain. | - |
| RequestSetActive | 110 | The request id. | - |
| ResponseRepeatedGiveCredit | 111 | The amount of credit to give (nonzero). | - |

#### Server Packets

| Packet | Tag | Integer | Remaining data |
|---|---|---|---|
| ResponseWrite | 00 | The id of the corresponding request. | The encoding of the response. |
| ResponseForgoCredit | 001 | The amount of credit to forgo (nonzero). | - |
| RequestGiveCredit | 010 | The amount of credit to give (nonzero). | - |
| RequestOops | 0110 | The maximum amount of credit to retain. | - |
| CancelResponse | 0111 | The id of the request. | - |
| RequestRepeatedGiveCredit | 100 | The amount of credit to give (nonzero). | - |
| RequestRepeatedOops | 1010 | The maximum amount of credit to retain. | - |
| ResponseRepeatedForgoCredit | 1011 | The amount of credit to forgo (nonzero). | - |
| ResponseRepeatedWrite | 110 | The number of repeated items (nonzero) | The concatenation of the encodings of each item. |
| ResponseSetActive | 111 | The request id. | - |
