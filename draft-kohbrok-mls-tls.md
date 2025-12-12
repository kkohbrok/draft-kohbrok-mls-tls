---
title: "Using MLS as Handshake in TLS 1.3"
abbrev: "MLS-TLS"
category: info

docname: draft-kohbrok-mls-tls-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Messaging Layer Security"
keyword:
 - TLS
 - Handshake
venue:
  group: "Messaging Layer Security"
  type: "Working Group"
  mail: "mls@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/mls/"
  github: "kkohbrok/draft-kohbrok-mls-tls"
  latest: "https://kkohbrok.github.io/draft-kohbrok-mls-tls/draft-kohbrok-mls-tls.html"

author:
 -
    fullname: Konrad Kohbrok
    organization: Phoenix R&D
    email: konrad@ratchet.ing
 -
    fullname: Raphael Robert
    organization: Phoenix R&D
    email: ietf@raphaelrobert.com

normative:

informative:

...

--- abstract

This document details how TLS handshake can be replaced by MLS.


--- middle

# Introduction

TODO Introduction

# MLS Overview

TODO: Give an overview over the MLS protocol.

# Protocol Overview

The TLS variant with MLS as handshake is not interoperable with vanilla TLS 1.3.
As such, a responder that wants to serve both variants needs to listen on
individual ports.

General message flow:

- Initiator sends a KeyPackage to the responder
- Responder creates MLS group and adds initiator using KeyPackage
- Responder responds with Welcome message
- Initiator receives Welcome and uses it to join group
- Both derive key material from the group which is passed on to the record
  protection layer
- Later, recipient and responder can send a commit in-band (i.e. protected by
  the record layer) to achieve PCS
- Recipient and responder can still use the TLS key update to achieve cheap FS

# Interface between MLS and TLS

The main difference between the MLS handshake and the regular TLS handshake is
that the MLS handshake introduces a new type of key update that provides
post-compromise security (PCS) in addition to forward secrecy (FS).

However, since the PCS update affects both the sending as well as the receiving
stream of each party, the resulting state machine is slightly more complex than
that of the regular TLS 1.3 handshake.

## Handshake messages

~~~ tls
struct {
  MLSMessage key_package;
} ClientHello

struct {
  MLSMessage welcome;
} ServerHello

struct {
  MLSMessage commit;
} Resumption

struct {
  MLSMessage update
} ConnectionUpdate

struct {
  uint32 epoch;
} EpochKeyUpdate
~~~

## Initial handshake

- Initiator creates KeyPackage
  - Initiator can use any ciphersuite and extension they want
  - Initiator advertises its support for extensions in Capabilities
- Responder inspects KeyPackage 
  - Check ciphersuite support
  - Check extension support
  - Check validity of credential
- Responder creates group and uses KeyPackage to add initiator
  - Group can potentially contain any extensions both participants support
- Responder sends the resulting Welcome to initiator
  - Welcome can include potential extensions as long as both participants support them
- Initiator receives Welcome
  - Check credential of responder
  - Check any extensions

## Keying the record protection layer

After applying a commit, initiator and responder export a key from the MLS key
schedule, which then acts as Master Secret in the context of the TLS 1.3 key
schedule. In some cases, initiator or responder may update the sending and
receiving stream keys individually. In that case, both keys are derived and the
one not applied is stored for later use.

## Connection updates

After the initial handshake is completed (or a session was resumed), both the
initiator and the responder can issue PCS updates. A PCS update is simply an MLS
commit. Since each PCS update yields new keys for the sending and the receiving
stream, the state machine is slightly more complex than that of TLS's regular
key update.

### Initiator behaviour

The initiator is in one of the following states:

1) Default 
2) Waiting for confirmation of own update
3) Waiting for the server to update its own key material

After the initial handshake, the initiator is in state 1.

In state 1, the initiator can create and send a ConnectionUpdate, upon which it
updates its sending stream keys, stores the ConnetionUpdate locally and
transitions to state 2.

In state 2, the initiator expects an EpochKeyUpdate message with the epoch the
initiator is currently in as confirmation that the responder has received the
ConnectionUpdate. If it receives that message, deletes the stored
ConnectionUpdate, updates its receiving stream keys and transitions to state 1.
If it receives any other handshake message, it drops that message. This is to
account for concurrently sent ConnectionUpdates, in which case the initiator's
update has priority.

In state 1, the initiator can also receive a ConnectionUpdate from the
responder. If that is the case, it sends an EpochKeyUpdate with the new epoch,
updates its sending stream and transitions to state 3.

In state 3, the initiator waits for an EpochKeyUpdate from the responder with
the current epoch. If it receives that message, it updates its receiving stream
keys and transitions to state 1.

### Responder behaviour

The responder is in one of the following states:

1) Default
2) Waiting for confirmation of own update

In state 1, the responder can create and send a ConnectionUpdate, but it doesn't
apply the commit yet and transitions to state 2.

In state 2, the responder waits for either an EpochKeyUpdate with the new epoch,
or a ConnectionUpdate from the initiator. If it receives the former, it applies
the commit and sends an EpochKeyUpdate with the new epoch. It then updates its
sending and receiving stream keys. If it receives the latter, it discards the
pending commits, transitions to state 1 and processes the update as described
below.

In state 1, the responder can also recieve a ConnectionUpdate from the
initiator. If that happens, the responder applies the initiator's update, sends
an EpochKeyUpdate with the new epoch and updates its sending and receiving
stream keys.

## Resumption

The initiator can resume a connection by sending a resumption message, which
consists of a simple MLS commit. If the connection was previously terminated
while the initiator was in state 2, it uses the commit from the ConnectionUpdate
for the Resumption message. After sending a resumption message, the initiator
enters state 2.

If the responder receives a resumption message, it does the following

- If the epoch is equal to its local epoch, it sends an EpochKeyUpdate with that
  epoch and discards the commit. This happens in case the connection broke off
  when the recipient had not receives an EpochKeyUpdate in the previous session.
- If the epoch is one higher than the current epoch, it it applies the commit
  and sends an EpochKeyUpdate.
- If the epoch is two higher than the current epoch, the connection must have
  broken off while the responder was in state 2. In that case, the responder
  frist applies the pending commit and then the commit from the
  ConnectionUpdate.


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
