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

This document details how MLS can be combined with the TLS record layer to yield
a secure channel protocol that can be configured to be post-quantum secure and
that is suitable for long-lived connection thanks to MLS key updates. In the
context of this composed protocol, MLS acts as a continuous key agreement
protocol that allows initiator and responder to update their key material not
just to achieve forward-secrecy, but also to post-compromise security.

--- middle

# Introduction

This document builds on the two-party profile for MLS
{{!I-D.draft-kohbrok-mls-two-party-profile}} and combines it with the record
layer of TLS 1.3 {{!RFC8446}} to yield a secure channel protocol. The
composition of both protocols is seamless due to the modular design of TLS 1.3
and MLS capability to act as a continuous key agreement protocol. After the
initial handshake, initiator and responder can tunnel MLS commits through the
secure channel to update their key material. Key material updates allow both
parties to achieve forward-secrecy and post-compromise security.

# Protocol Overview

The TLS variant with MLS as handshake is not interoperable with vanilla TLS 1.3.
As such, a responder that wants to serve both variants needs to listen on
individual ports.

The protocol consists of three phases.

1. Initial key agreement, where initiator and responder agree on key material
   using the MLS two party profile.
2. Secure channel phase, where initiator and responder can transfer data
   encrypted using the TLS 1.3 record layer protocol. The encryption keys are
   derived from the MLS group created during the initial key agreement phase.
   During the secure channel phase, initiator and responder can tunnel MLS
   commits through the secure channel to update their key material.
3. Resumption, where initiator or responder can resume a previously interrupted
   connection without having to repeat phase 1, including the ability to send
   data in the first flight of messages.

For more details on the initial key agreement, resumption, as well as how
message updates work, see {{!I-D.draft-kohbrok-mls-two-party-profile}}.

# Initializing the record layer

Both after the initial key agreement phase and the resumption phase, initiator
and responder derive key material from the MLS group created during the initial
key agreement phase.

The `client_application_traffic_secret` and the
`server_application_traffic_secret` required by the record layer are derived as
follows.

~~~ tls
server_application_traffic_secret =
  MLS-Exporter("MLS-TLS s ap traffic", [], Length)

client_application_traffic_secret =
  MLS-Exporter("MLS-TLS c ap traffic", [], Length)
~~~

Where MLS-Exporter is defined in {{!RFC9420}} and Length is the size of the
secret required by the TLS record layer.

# Updating record layer keys

The MLS two party profile allows both parties to update the MLS group created in
the initial key agreement phase. In the context of MLS-TLS, these
ConnectionUpdate and EpochKeyUpdate messages (as defined in
{{!I-D.draft-kohbrok-mls-two-party-profile}}) are sent in place of key update messages
defined in {{!RFC8446}}. Concretely, they serialized and sent as the `content`
of a TLSInnerPlaintext message with `type` set to `handshake`.

TODO: The two-party profile I-D should define a message similar to
{{!RFC9420}}'s MLSMessage, which contains either a ClientHello, ServerHello,
ConnectionUpdate or EpochKeyUpdate. This is to allow recipients to know what to
deserialize when they receive an in-band update message.

When a party is clear to start using the key material as specified in Section 4
of {{!I-D.draft-kohbrok-mls-two-party-profile}}, the client derives new application
traffic secrets as specified in {{initializing-the-record-layer}}.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
