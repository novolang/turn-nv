# turn-nv

TURN, Traversal Using Relays around NAT, lets a host that cannot be reached
directly use a server on the public internet as a relay. The host asks a TURN
server for an address on the server, the relayed transport address, and the
server forwards the packets that arrive there to the host, and the packets the
host sends to it onward to the peer. TURN is specified in
[RFC 8656](https://www.rfc-editor.org/rfc/rfc8656) and is the fallback of ICE,
the way WebRTC and SIP phones find a path between two hosts. This package is
the TURN client for novo-lang. It is built on the STUN messages of
[stun-nv](https://novo-lang.org/packages/stun-nv).

## What it is

A TURN client holds four kinds of state at the server.

- The **allocation** is the relayed transport address. The client asks for it
  with an Allocate request and keeps it with Refresh requests. The server holds
  it for a lifetime, 600 seconds unless the client asks for another. A Refresh
  with a lifetime of 0 releases it.
- A **permission** says that a peer's IP address may send to the relayed
  address. The client installs it with a CreatePermission request. It lasts 300
  seconds. The server drops what a peer without a permission sends.
- A **channel binding** gives a peer's transport address a channel number from
  0x4000 to 0x4FFF. The client installs it with a ChannelBind request. It lasts
  600 seconds, and it installs the permission for the peer too.
- **Data** goes to and from a peer in one of two forms. A Send indication, and
  its counterpart the Data indication, carry the peer's address in a STUN
  message. Once a channel is bound, ChannelData messages carry the data after
  four bytes: the channel number and the length.

Every request except the first Allocate is authenticated with the long-term
credential mechanism of
[RFC 8489](https://www.rfc-editor.org/rfc/rfc8489) section 9.2. The first
Allocate carries no credentials. The server answers 401 with a realm and a
nonce, and the client sends the request again, signed with a key derived from
its username, the realm and its password. A server may later answer 438, Stale
Nonce, with a new nonce.

The package has no socket and no clock. A program that uses it owns the UDP or
TCP socket and the timer. The session builds the datagrams to send, reads the
ones that come back, and says what to do and when.

**Nothing is allocated.** A message is a `Vec[u8; 1280]` in the caller's
frame. A session is a `@value` struct of about 2,700 bytes that holds six
requests in flight, eight permissions and eight channels. The whole package
builds for a Cortex-M with no heap.

## Install

```
novo pkg add turn-nv
```

## Example

A client allocates a relay, and the server's part is played in the program: it
answers the first request with a 401, and the second with the relayed address.

```novo
use stunwire
use stunmac
use turnwire
use turnsess

fn main() [io]
    var s = turnsess.session("alice", "secret", 0, false)
    var out: stunwire.StunMsg = Vec.new()
    var data: stunwire.StunMsg = Vec.new()

    // The first Allocate carries only REQUESTED-TRANSPORT.
    let _ = turnsess.allocate(var s, var out, stunwire.txid(1, 1))

    // The server asks for credentials.
    let h = stunwire.parse(var out)
    var r1: stunwire.StunMsg = Vec.new()
    let _ = stunwire.begin(var r1, stunwire.STUN_ERROR, turnwire.TURN_ALLOCATE, h.txid)
    let _ = stunwire.add_error_code(var r1, 401, "Unauthenticated")
    let _ = stunwire.add_str(var r1, stunwire.STUN_ATTR_REALM, "example.org")
    let _ = stunwire.add_str(var r1, stunwire.STUN_ATTR_NONCE, "n1")
    let again = turnsess.on_datagram(var s, var r1, var out, var data, stunwire.txid(2, 2))
    println("send again ${again.event == turnsess.TURN_EV_SEND}")   // send again true

    // The server answers the signed request with the relayed address.
    let h2 = stunwire.parse(var out)
    var r2: stunwire.StunMsg = Vec.new()
    let _ = stunwire.begin(var r2, stunwire.STUN_SUCCESS, turnwire.TURN_ALLOCATE, h2.txid)
    let relayed = stunwire.addr_v4(198, 51, 100, 7, 49152)
    let _ = stunwire.add_addr(var r2, turnwire.TURN_ATTR_XOR_RELAYED_ADDRESS, relayed, h2.txid, true)
    let _ = turnwire.add_lifetime(var r2, 600)
    var key: stunmac.StunKey = Vec.new()
    let _ = stunmac.long_term_key(stunmac.STUN_HASH_MD5, "alice", "example.org", "secret", var key)
    let _ = stunmac.add_integrity(var r2, var key, false)
    let done = turnsess.on_datagram(var s, var r2, var out, var data, stunwire.txid(3, 3))
    println("allocated ${done.event == turnsess.TURN_EV_ALLOCATED} port ${s.relayed.port}")   // allocated true port 49152
```

Build and test with `novo pkg build` and `novo test tests/turnsess_tests.nv`;
`tests/coverage.sh` merges the coverage of all the suites.

## What the package contains

| Module | Contents |
| --- | --- |
| `turnsess` | The session: Allocate, Refresh and release, CreatePermission, ChannelBind, data sent in Send indications and ChannelData, data received, the 401 and 438 retries, the retransmission of every request in flight, and the lifetimes of the allocation, the permissions and the channels counted from the time the caller reports. |
| `turnattr` | The attributes of RFC 8656 section 18 that stun-nv does not build or read: REQUESTED-ADDRESS-FAMILY, ADDITIONAL-ADDRESS-FAMILY, ADDRESS-ERROR-CODE and ICMP, and readers for CHANNEL-NUMBER, EVEN-PORT, DONT-FRAGMENT, RESERVATION-TOKEN and a second relayed address. |
| `turnframe` | A reader that cuts a TCP or TLS byte stream into STUN and ChannelData messages, with the padding of ChannelData, and the padding of a message to send. |

The STUN message, the long-term credential key, the integrity check, the
retransmission timer, the TURN method and attribute numbers and the ChannelData
frame are those of stun-nv's `stunwire`, `stunmac`, `stunclient` and
`turnwire`. stun-nv also holds `turnclient`, which runs one allocation. This
package's session replaces it where a program needs permissions, channels and
data.

Every `pub` declaration is documented where it is declared, and
[the package's page](https://novo-lang.org/packages/turn-nv) is that
reference.

## The rules a user needs

1. **The session never waits.** `tick` reports elapsed milliseconds and answers
   what has fallen due. Several timers can fall due together, and each call
   answers one thing, so a caller asks again with an elapsed time of 0 until the
   answer is `TURN_EV_NONE`.
2. **The caller supplies transaction IDs.** Each call that may build a request
   takes a fresh random 96-bit ID. The package chooses none.
3. **A retransmission is the same bytes.** A request is sent again at 0, 500,
   1500, 3500, 7500, 15500 and 31500 ms and fails 8000 ms after the last, at
   39500 ms (RFC 8489 section 6.2.1). Over a reliable transport it is sent once.
   The retransmitted request has the transaction ID and the nonce of the
   original.
4. **Six requests are in flight at most.** A seventh answers `TURN_ERR_FULL`.
5. **A permission is for an IP address.** The port of the peer does not matter,
   and a ChannelBind installs the permission of its peer.
6. **The session renews what it holds.** The allocation is refreshed when half
   its lifetime has passed. A permission and a channel are asked for again when
   60 seconds of them are left. A refresh that fails is tried again 30 seconds
   later, and a row that nothing renewed ends with `TURN_EV_EXPIRED`.
7. **A channel number is held by one peer, and a peer by one channel.** After a
   binding expires the number and the peer stay reserved for 5 minutes, and a
   binding to anything else in that time answers `TURN_ERR_CONFLICT` (RFC 8656
   section 12).
8. **Data uses the channel once it is bound.** Until the server confirmed the
   binding, and for a peer with no channel, a Send indication carries the data.
   A Send indication needs a permission, since the server drops the data
   without one. Over a stream, ChannelData is padded to a multiple of four.
9. **Data is received only from a peer that has a permission.** A Data
   indication from any other peer is ignored, and so is ChannelData on a channel
   the session does not hold. RFC 8656 section 12.6 recommends it.
10. **A 400 to a ChannelBind ends the allocation.** RFC 8656 section 12.3 says
    the two ends then disagree about the channels and advises starting afresh.
11. **A success is believed only if it is signed.** After the first 401, a
    success response whose MESSAGE-INTEGRITY does not verify is ignored and its
    timer goes on (RFC 8489 section 9.2.5).
12. **Text is held to 128 bytes.** A username, password, realm or nonce longer
    than `STUN_TEXT_MAX` is refused rather than cut.
13. **The options of an Allocate follow RFC 8656 section 7.1.** EVEN-PORT and
    RESERVATION-TOKEN are exclusive, and so are REQUESTED-ADDRESS-FAMILY and
    ADDITIONAL-ADDRESS-FAMILY. `set_options` refuses the combinations.

## Running on a microcontroller

All three modules are `@tier(embedded)` throughout and allocate nothing, and
the package declares no effects. `tests/embedded_probe.nv` is a firmware
program for a device that streams media through a relay. It builds the TURN
attributes and reads them back, cuts two padded ChannelData messages out of a
stream, and runs a session through the 401 challenge to a relayed address, a
permission, a channel, data both ways, a refresh and the release. It is
compiled for a Cortex-M as part of this package's checks, so the claim is built
rather than asserted, and under QEMU it prints `PASS: turn-embedded`.

The image is 107,192 bytes of flash and 520 bytes of static RAM, including the
runtime and the board's startup code. Most of the flash is the hash functions of
stun-nv. The largest values the probe holds are a session of 2,664 bytes, a
stream reader of 1,312 bytes and a message of 1,288 bytes, all in the frame.

The registry shows the embedded, rt, system, wasm and app tiers for this
package.

## What is not included

- **A TURN server: NOT IMPLEMENTED.** This package is the client of RFC 8656.
  It has no allocation table, no relay and no permission checks on the server's
  side.
- **TURN over TCP and TLS allocations.** The framing of a stream is here
  (`turnframe`). Allocations that relay TCP, RFC 6062, and the transport
  security of TLS and DTLS are not.
- **A requested transport other than UDP**, which RFC 8656 allows only.
- **Preparing the strings.** Applying OpaqueString (RFC 8265) to a username or a
  password is the caller's.
- **The SHA-256 key and the other password algorithms of RFC 8489 section 9.2.**
  The session derives the MD5 key.
- **Finding the server.** The DNS discovery of RFC 8656 section 4 and the
  ALTERNATE-SERVER redirect are the caller's.
- **Sockets and timers.**

## Related packages

- [stun-nv](https://novo-lang.org/packages/stun-nv) holds the STUN message and
  the credential checks this package is built on.
- [rtp-nv](https://novo-lang.org/packages/rtp-nv) reads and writes the RTP
  packets that travel as the data of a relay, and
  [srtp-nv](https://novo-lang.org/packages/srtp-nv) protects them.
- [ice-nv](https://novo-lang.org/packages/ice-nv) takes the relayed address as a
  candidate and decides which path the media takes.

## Tests

The suite is three files and a probe under `tests/`. `tests/coverage.sh` reports the
coverage of `src/` merged across them.

`tests/turnattr_tests.nv` holds the attributes of RFC 8656 section 18 to bytes
worked out by hand from its figures, and the derivation is in the test. The
RFC prints no complete TURN message in hexadecimal, so the messages are built
and read against their figures.

`tests/turnframe_tests.nv` holds ChannelData to the format of section 12.4,
bare and padded, cut out of a byte stream with a STUN message and the faults of
a stream that has lost its place.

`tests/turnsess_tests.nv` runs the session against a server written in the
test. It follows the allocation through the 401 challenge, the retransmission
schedule send by send, a 438 answered three times, the options of an Allocate,
both address families and the ADDRESS-ERROR-CODE, the permission and channel
tables, the refresh times of the allocation, the permissions and the channels,
the quarantine of 5 minutes, data in both forms, and every error. It verifies
the integrity of the session's requests under the key of
`alice:example.org:secret` with stun-nv's own check.

`tests/embedded_probe.nv` is the microcontroller program described above.

## Licence

Apache-2.0. See `LICENSE`.
