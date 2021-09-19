# Reqres

A minimalistic request-response specification.

## What, Why, and How?

The specification deals with some design issues of request-response protocols over a reliable, ordered, byte oriented channel, namely the mapping from responses to concurrent requests, and flow control. Flow control is often neglected in such protocols, leading either to exhaustion of computational resources, or delay in transmitting some responses just because a completely unrelated to resource is currently unavailable.

Reqres is an asymmetric protocol; the *client* endpoint issues *requests*, and the *server* endpoint serves *responses*. A symmetric exchange can be emulated by running two reqres sessions in parallel. When issuing a request, the client annotates it with a 64-bit integer, the *request id*. Each response is annotated with the id of the request to which it pertains.

Flow control is based on a credit-based scheme, very similar to that of [minmux](https://github.com/AljoschaMeyer/minmux), the remainder of this text assumes familiarity with the minmux specification. Like minmux, a reqres consists of exchanging different packet, and each consisting of a light header followed by packet-specific data.

The specifics of the flow control mechanism employed by reqres depend on whether requests and/or responses are of a static size, or whether any (and which) of them include streaming data of arbitrary length. Reqres is actually a family of four related protocols, one protocol for each combination of static or streaming requests or responses.

## Static Requests and Responses

We first describe the simple case of both requests and responses being a finite size, for example a request might consists of a fixed-width user identifier, and the response might be a 64-bit timestamp of that user's last activity. More generally, we fix a `Request` type, and a `Response` type, both inhabited by finitely many values (i.e., the encoding of any value has a well-known, maximum size).

In such a setting, the endpoints maintain a logical *request channel* from the client to the server on which requests are sent. Flow control on this channel is done in units of complete requests. The channel is maintained similar to a minmux data stream, with the client sending `Write` and `ForgoCredit` packets, and the server sending `GiveCredit` and `Oops` packets. This channel is never closed, so (unlike with [minmux](https://github.com/AljoschaMeyer/minmux)) there is no dedicated last item type, and no `StopWrite` or `StopRead` packets are available. Furthermore there are no `RequestCredit` and `RequestItems` packets, because we assume an unbounded sequence of requests and corresponding responses anyways.

A similar, logical *response channel* from the server to the client handles responses. Flow control is done in units of complete responses, with the server sending `Write` and `ForgoCredit` packets, and the client sending `GiveCredit` and `Oops` packets.

In addition to these two channels, clients may cancel a request by sending a `CancelRequest` packet containing the corresponding request id. There is no flow control for cancellations, the server must allocate the necessary state for dealing with a request's cancellation together with the state for responding to the request. The server must ignore cancellations with an id that does not match any currently active request (this is the easiest way of preventing problems when a request is concurrently canceled by the client and responded to by the server). Otherwise, the server should send a response for the indicated request as quickly as possible. Usually, the response type would be a [sum type](https://en.wikipedia.org/wiki/Tagged_union) with a variant for cancellation that carries no further information, which would be sent when cancellation is requested.

When issuing new request, clients should use fresh ids. Initially, all ids are considered fresh, using an id for a request makes it non-fresh, until a response of that id has been received (in particular, canceling a request does not make its id fresh). Servers may assume incoming requests to have fresh ids, they do not need to check whether the id is indeed fresh. It is the client's responsibility to not mess things up.

## Streaming Data

We now consider the case where responses consist of streaming data. One example is access to a content-addressed storage, where requests consist of a hash of fixed size, but responses are higher contents of arbitrary length. More generally, instead of a single `Response` type, there are three finite types: `ResponseFirst`, `ResponseRepeated` and `ResponseLast`. In the preceding example, both `ResponseFirst` and `ResponseLast` are the [unit type](https://en.wikipedia.org/wiki/Unit_type), and `ResponseRepeated` is the type of individual bytes.

The first and last items of a request are sent over the response channel. Only the first item consumes credit. Canceling a request is now asking the server to send a last item for that request as soon as possible (which still may only be done after having transmitted a first item for that request). After sending the first but before sending the last item, an endpoint can send an arbitrary number of repeated items. These items need a different form of flow control, because they should not block the concurrent transmission of new request.

Maintaining separate credit the repeated items for every individual response would be inefficient, because the client would have to arbitrarily divide its available resources amongst its request. Instead, reqres manages flow control of repeated items for all responses collectively, the server gets to decide which responses to prioritize. This writer-driven, unified flow control is the main service provided by reqres, and the main distinguishing feature from [minmux](https://github.com/AljoschaMeyer/minmux).

There is a logical *streaming response channel* from the server to the client; the repeated items of all responses are sent over that channel. It is never closed, so it is maintained via `Write` and `ForgoCredit` packets from the server, and `GiveCredit` and `Oops` from the client. The data written over this channel is not explicitly tagged with a response id, instead there is at any point exactly one *active response id*, and all items written on the streaming response channel pertains to the request with the currently active response id. The server can set the active response id with a `SetActiveResponse` packet, only ids for which a first item but no last item has been transmitted on the response shall channel are allowed.

Both the sending of repeated response items and of `SetActiveResponse` are limited through the credit given on the streaming response channel, the unit of credit are the individual bytes of the `Write` and `SetActiveResponse` packets respectively. Thus, the client can maintain a single byte buffer into which such incoming packets can be copied for later processing.

Similarly to streaming responses, there can also be streaming request. In those settings, instead of a single `Request` type, there are three finite types: `RequestFirst`, `RequestRepeated` and `RequestLast`. `RequestFirst` and `RequestLast` are sent over the request channel, flow control works analogously as for the response counterparts. Also analogously, there is a *streaming request channel* from the client to the server, over which the repeated items of all request are sent. Repeated items are routed to the current *active request id*, which can be set via `SetActiveRequest` packet. Repeated request items and `SetActiveRequest` packets share byte-wise credit on the streaming request channel.

Additionally, the server can cancel a streaming request via a `CancelResponse` packet, to which the client should respond by sending a last request item for the indicated request as soon as possible.

## Protocols

There are four combinations of static or streaming requests and/or responses, each leading to a particular protocol, according to the proceeding introduction. We now give for each such protocol a self-contained overview over its parameters, a summary of the different packets it uses, and finally specific encodings for sending these packets over a reliable, ordered, byte-oriented channel.
