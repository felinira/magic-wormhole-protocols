# Persistent transit

Applications may exchange Wormhole messages in order to build a Transit connection.
They are expected to use that for most traffic instead, because Wormhole messages
are not efficient (they are hex-encoded, and also the rendezvous server has too
keep the entire communication history in memory).

Alternatively, the may use "persistent" transit to get a channel that may survive
connection interruptions. This works by using the Wormhole messages exclusively
for making out new transit connections. There is a thin abstraction wrapper around
transits that gives a "persistent channel". This is then exclusively used for
the actual application traffic.

## Application version

Clients must specify that they support persistent transit using their `app_version`
message. We recommend doing so by using a `persistent-transit-v1` ability.
That message must also contain `transit-abilities` information (the
transit abilities of a client are fixed over the entire session and won't be re-
negotiated after each transit breakdown). Persistent transit will be used if and
only if both sides support it.

## The persistent channel

The persistent transit offers the same abstraction as a transit connection:
clients may send "records" of arbitrary length, transport and encryption will
be handled for them. The difference is that this one keeps the messages around
until they are guaranteed to be received, and re-sends them over a different
connection if necessary.

Wrapper messages:

- ID, ACK, PAYLOAD

TODO specify how to derive the key (use `transit` version for uniqueness)

## Messages exchanged over the Wormhole rendezvous server

Initially, and when one side thinks it has lost connection, a `transit` message
containing the hints is sent. The other side must answer it by doing the same.
Then, the transit process proceeds as usual. As soon as it completes, the
previous transit connection is abandoned (if it hasn't been closed already)
and the new one takes over. The first thing the persistent channel will do is
to retransmit messages that had not been acknowledged before the connection
loss (in order).

If the process fails, it shall be repeated until success or one side gives up.

For transit to succeed, both sides need to connect to the correct targets more
or less at the same time. To facilitate this, `transit` messages have a time
stamp and are versioned. Only hints of the same version should be used for
conenction. If a newer `transit` message appears, old connection attempts should
be cancelled. `transit` messages that are older than five minutes should be ignored.

To make versioning easier, only the leader side initiates `transit` messages,
and thus that side also maintains the version counter.

### Leaders and Followers

Each side of a Wormhole has a randomly-generated dilation `side` string (this
is included in the `please-dilate` message, and is independent of the
Wormhole's mailbox "side"). When the wormhole is dilated, the side with the
lexicographically-higher "side" value is named the "Leader", and the other
side is named the "Follower". The general wormhole protocol treats both sides
identically, but the distinction matters for the dilation protocol.

Both sides send a `please-dilate` as soon as dilation is triggered. Each side
discovers whether it is the Leader or the Follower when the peer's
"please-dilate" arrives. The Leader has exclusive control over whether a
given connection is considered established or not: if there are multiple
potential connections to use, the Leader decides which one to use, and the
Leader gets to decide when the connection is no longer viable (and triggers
the establishment of a new one).

## L3 protocol

The L3 layer is responsible for connection selection, monitoring/keepalives,
and message (de)serialization. Framing is handled by L2, so the inbound L3
codepath receives single-message bytestrings, and delivers the same down to
L2 for encryption, framing, and transmission.

Connection selection takes place exclusively on the Leader side, and includes
the following:

* receipt of viable L2 connections from below (indicated by the first valid
  decrypted frame received for any given connection)
* expiration of a timer
* comparison of TBD quality/desirability/cost metrics of viable connections
* selection of winner
* instructions to losing connections to disconnect
* delivery of KCM message through winning connection
* retain reference to winning connection

On the Follower side, the L3 manager just waits for the first connection to
receive the Leader's KCM, at which point it is retained and all others are
dropped.

The L3 manager knows which "generation" of connection is being established.
Each generation uses a different dilation key (?), and is triggered by a new
set of L1 messages. Connections from one generation should not be confused
with those of a different generation.

Each time a new L3 connection is established, the L4 protocol is notified. It
will will immediately send all the L4 messages waiting in its outbound queue.
The L3 protocol simply wraps these in Noise frames and sends them to the
other side.

The L3 manager monitors the viability of the current connection, and declares
it as lost when bidirectional traffic cannot be maintained. It uses PING and
PONG messages to detect this. These also serve to keep NAT entries alive,
since many firewalls will stop forwarding packets if they don't observe any
traffic for e.g. 5 minutes.

Our goals are:

* don't allow more than 30? seconds to pass without at least *some* data
  being sent along each side of the connection
* allow the Leader to detect silent connection loss within 60? seconds
* minimize overhead

We need both sides to:

* maintain a 30-second repeating timer
* set a flag each time we write to the connection
* each time the timer fires, if the flag was clear then send a PONG,
  otherwise clear the flag

In addition, the Leader must:

* run a 60-second repeating timer (ideally somewhat offset from the other)
* set a flag each time we receive data from the connection
* each time the timer fires, if the flag was clear then drop the connection,
  otherwise clear the flag

In the future, we might have L2 links that are less connection-oriented,
which might have a unidirectional failure mode, at which point we'll need to
monitor full roundtrips. To accomplish this, the Leader will send periodic
unconditional PINGs, and the Follower will respond with PONGs. If the
Leader->Follower connection is down, the PINGs won't arrive and no PONGs will
be produced. If the Follower->Leader direction has failed, the PONGs won't
arrive. The delivery of both will be delayed by actual data, so the timeouts
should be adjusted if we see regular data arriving.

If the connection is dropped before the wormhole is closed (either the other
end explicitly dropped it, we noticed a problem and told TCP to drop it, or
TCP noticed a problem itself), the Leader-side L3 manager will initiate a
reconnection attempt. This uses L1 to send a new DILATE message through the
mailbox server, along with new connection hints. Eventually this will result
in a new L3 connection being established.

Finally, L3 is responsible for message serialization and deserialization. L2
performs decryption and delivers plaintext frames to L3. Each frame starts
with a one-byte type indicator. The rest of the message depends upon the
type:

* 0x00 PING, 4-byte ping-id
* 0x01 PONG, 4-byte ping-id
* 0x02 OPEN, 4-byte subchannel-id, 4-byte seqnum
* 0x03 DATA, 4-byte subchannel-id, 4-byte seqnum, variable-length payload
* 0x04 CLOSE, 4-byte subchannel-id, 4-byte seqnum
* 0x05 ACK, 4-byte response-seqnum

All seqnums are big-endian, and are provided by the L4 protocol. The other
fields are arbitrary and not interpreted as integers. The subchannel-ids must
be allocated by both sides without collision, but otherwise they are only
used to look up L5 objects for dispatch. The response-seqnum is always copied
from the OPEN/DATA/CLOSE packet being acknowledged.

L3 consumes the PING and PONG messages. Receiving any PING will provoke a
PONG in response, with a copy of the ping-id field. The 30-second timer will
produce unprovoked PONGs with a ping-id of all zeros. A future viability
protocol will use PINGs to test for roundtrip functionality.

All other messages (OPEN/DATA/CLOSE/ACK) are deserialized and delivered
"upstairs" to the L4 protocol handler.

The current L3 connection's `IProducer`/`IConsumer` interface is made
available to the L4 flow-control manager.

## L4 protocol

The L4 protocol manages a durable stream of OPEN/DATA/CLOSE/ACK messages.
Since each will be enclosed in a Noise frame before they pass to L3, they do
not need length fields or other framing.

Each OPEN/DATA/CLOSE has a sequence number, starting at 0, and monotonically
increasing by 1 for each message. Each direction has a separate number space.

The L4 manager maintains a double-ended queue of unacknowledged outbound
messages. Subchannel activity (opening, closing, sending data) cause messages
to be added to this queue. If an L3 connection is available, these messages
are also sent over that connection, but they remain in the queue in case the
connection is lost and they must be retransmitted on some future replacement
connection. Messages stay in the queue until they can be retired by the
receipt of an ACK with a matching response-sequence-number. This provides
reliable message delivery that survives the L3 connection being replaced.

ACKs are not acked, nor do they have seqnums of their own. Each inbound side
remembers the highest ACK it has sent, and ignores incoming OPEN/DATA/CLOSE
messages with that sequence number or higher. This ensures in-order
at-most-once processing of OPEN/DATA/CLOSE messages.

Each inbound OPEN message causes a new L5 subchannel object to be created.
Subsequent DATA/CLOSE messages for the same subchannel-id are delivered to
that object.

Each time an L3 connection is established, the side will immediately send all
L4 messages waiting in the outbound queue. A future protocol might reduce
this duplication by including the highest received sequence number in the L1
PLEASE-DILATE message, which would effectively retire queued messages before
initiating the L2 connection process. On any given L3 connection, all
messages are sent in-order. The receipt of an ACK for seqnum `N` allows all
messages with `seqnum <= N` to be retired.

The L4 layer is also responsible for managing flow control among the L3
connection and the various L5 subchannels.
