# Changelog

All notable changes to turn-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0

The first release.

### Added

- **`turnsess`**, a TURN client session as a step a caller drives: Allocate,
  Refresh and release, CreatePermission and ChannelBind, with up to six
  requests in flight, each with its own transaction ID and retransmission
  timer (RFC 8489 section 6.2.1); the 401 challenge and the 438 stale nonce of
  the long-term credential mechanism; the lifetimes of the allocation (renewed
  at half), of eight permissions (300 seconds, renewed with 60 left) and of
  eight channels (600 seconds, renewed with 60 left); the quarantine of 5
  minutes after a binding expires; Send indications and ChannelData for data
  out, and Data indications, ChannelData and the ICMP report for data in; the
  options of an Allocate (REQUESTED-ADDRESS-FAMILY, ADDITIONAL-ADDRESS-FAMILY,
  EVEN-PORT, RESERVATION-TOKEN, DONT-FRAGMENT and LIFETIME) with the
  combinations RFC 8656 section 7.1 refuses; and a second relayed address, the
  reservation token and the ADDRESS-ERROR-CODE read from the success.
- **`turnattr`**, the attributes of RFC 8656 section 18 that stun-nv's
  `turnwire` does not build or read: REQUESTED-ADDRESS-FAMILY,
  ADDITIONAL-ADDRESS-FAMILY, ADDRESS-ERROR-CODE and ICMP, and readers for
  CHANNEL-NUMBER, EVEN-PORT, DONT-FRAGMENT, RESERVATION-TOKEN and the nth
  address of a type.
- **`turnframe`**, a reader that cuts a stream into STUN and ChannelData
  messages, skipping the padding of ChannelData, and `pad_for_stream`.
- **`tests/embedded_probe.nv`**, a firmware program for a device that streams
  through a relay, built for `--target=nrf52-qemu`. It prints
  `PASS: turn-embedded` under QEMU. The image is 107,192 bytes of flash and 520
  bytes of static RAM.
- Three test suites.

### Not included

- A TURN server.
- TURN over TCP allocations (RFC 6062) and the transport security of TLS and
  DTLS.
- Preparing strings with OpaqueString, the SHA-256 key and the other password
  algorithms, DNS discovery of a server, and following an ALTERNATE-SERVER.

### Known

- stun-nv 0.1.0 has a `turnclient` that runs one allocation with no tables of
  permissions and channels. This package's session does not use it, so a
  program that wants only an allocation has two ways to run one.
- stun-nv 0.1.0 holds text to 128 bytes, though RFC 8489 allows a realm or a
  nonce of up to 763 bytes, so a server whose nonce is longer fails the
  allocation with `TURN_ERR_BAD_RESPONSE`.
