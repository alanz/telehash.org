Reliable Channels
=================

Telehash packets are by default only as reliable as UDP itself is, which means they may be dropped or arrive out of order.  Most of the time applications want to transfer content in a durable way, so reliable channels replicate TCP features such as ordering, retransmission, and buffering/backpressure mechanisms. The primary method of any application interfacing with a switch library is going to be through starting or receiving reliable channels.

Reliability is requested on a channel with the very first packet (that contains the `type`) by including a `"seq":0` with it, and a switch must respond with an `err` if it cannot handle reliable channels.  Reliability must not be requested for channel types that are expected to be unreliable (like `seek` and `peer`, etc).

## `seq` - Sequenced Data

The principle requirement of a reliable channel is always including a simple `"seq":0` integer value on every packet that contains any content (including the `end`). All `seq` values start at 0 and increment per packet sent when that packet contains any data to be processed, with a maximum value of 4,294,967,295 (a 32-bit unsigned integer)

A buffer of these packets must be kept keyed by the seq value until the receiving switch has responded
confirming them in a `ack` (below). The maximum size of this buffer and the number of outgoing packets that can be sent before being acknowledged is currently 100, but this is very much just temporary to ease early implementations and handling it's size will be defined in it's own document.

Upon receiving `seq` values, the switch must only process the packets and their contents in order, so any packets received with a sequence value out of order or with any gaps must either be buffered or dropped.  It is much more efficient to buffer these when a switch has the resources to do so, but if buffering it must have it's own internal maximum to limit it.

## `ack` - Acknowledgements

The `"ack":0` integer is always included on any outgoing packets as the highest known `seq` value confirmed as *processed by the app*. What this means is that any switch library must provide a way to send data/packets to the app using it in a serialized way, and be told when the app is done processing one packet so that it can both confirm that `seq` as well as give the app the next one in order. Any outgoing `ack` must represent only the latest app-processed data `seq` so that the sender can confirm that the data was completely received/handled by the recipient app.

An `ack` may also be sent in it's own packet ad-hoc at any point without any content data, and these ad-hoc acks must not include a `seq` value as they are not part of the content stream.

When receiving an `ack` the switch may then discard any buffered packets up to and including that matching `seq` id, and also confirm to the app that the included content data was received and processed by the other side.

An `ack` must be sent a minimum of once per second after receiving any packet including a `seq` value, either included with response content packets or their own ad-hoc packets.  Allowing up to one second gives a safe default for the application to generate any response content, as well as receive a larger number of content packets before acknowleding them all.

## `miss` - Misssing Sequences

The `"miss":[1,2,4]` is an array of integers and must be sent along with any `ack` if in the process of receiving packets there were any missing sequences, containing in any order the known missing sequence values.  Because the `ack` is confirmed processed packets, all of the `miss` ids will be higher than the accompanying `ack`.

This is only applicable when a receiving switch is buffering incoming sequences and is missing specific packets in the buffer that it requires before it can continue processing the content in them.

A `miss` should be no larger than 100 entries, any array larger than that should be discarded, as well as any included sequence values lower than the accompanying `ack` or higher than any outgoing sent `seq` values.

Upon receiving a `miss` the switch should resend those specific matching `seq` packets in it's buffer, but never more than once per second. So if an id in a `miss` is repeated or shows up in multiple incoming packets quickly (happens often), the matching packet is only resent once until at least one second has passed.

A switch MAY opportunistically remove packets from it's outgoing buffer that are higher than the `ack` and lower than the highest value in a `miss` array, and are not included in the array as that's a signal that they've been received.

# Reliability Notes

Here's some summary notes for implementors:

* send an ack with every outgoing packet, of the highest seq you've received and processed
* only send a miss if you've discovered missing packets in the incoming seq ordering/buffering
* if an app is receiving packets and hasn't generated response packets, send an ack after 1s
* when an `end` is sent, don't close the channel until it's acked
* when an `end` is received, process it in order like any other content packet, and close only after acking + timeout wait to allow re-acking if needed
* automatically resend the last-sent un-acked sequence packet every 2 seconds until the channel timeout
