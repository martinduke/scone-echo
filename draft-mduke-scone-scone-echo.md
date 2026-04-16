---
title: In-Band SCONE Reporting over QUIC
abbrev: scone-echo
category: std

docname: draft-mduke-scone-scone-echo-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Standard Communication with Network Elements"
keyword:
venue:
  group: "Standard Communication with Network Elements"
  type: "Working Group"
  mail: "scone@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/scone"
  github: "martinduke/scone-echo"
  latest: "https://martinduke.github.io/scone-echo/draft-mduke-scone-scone-echo.html"

author:
 -
    fullname: "Martin Duke"
    organization: Google
    email: "martin.h.duke@gmail.com"

normative:
  SCONE: I-D.ietf-scone-protocol
  QUIC: RFC8999

informative:

...

--- abstract

The SCONE protocol relies on the receiver of SCONE packets to send bandwidth
estimates back to the sender via unspecified application-layer messages. In some
cases, a peer might have SCONE receive capability at the QUIC layer but not
implement the necessary application level functionality. A new QUIC frame that
directly reports the contents of received SCONE packets can address these use
cases. There are no changes in the interaction with SCONE Network Elements.

--- middle

# Introduction

The SCONE protocol ({{SCONE}}) allows networks to provide bandwidth guidance to
endpoints. Senders prepend a SCONE header to QUIC ({{QUIC}}) packets that
include a 6-bit bandwidth field. Network elements can update this field. The
receiver of SCONE packets reports the received value to the application. The
application can use this information to adjust the bit rate, either by directly
reporting the value back to the sender at the application layer, or by using it
to make some other adjustment to the incoming traffic.

This architecture requires cooperation from the application layer: a
receiver cannot usefully process a SCONE packet without application involvement
to take action on the result. In principle, a QUIC implementation could *send*
SCONE packets solely based on the receiver's advertised ability to receive, but
it might have difficulty determining the correct rate to send such packets. The
receiver would need to effectuate any behavior changes without SCONE-aware
cooperation from the sender. The authors are not aware of any deployments that
send SCONE packets without explicit confirmations from the application layer.

There are some use cases where it would be useful to not require cooperation
from the receiving application, instead returning feedback directly at the QUIC
layer. There are fewer QUIC implementations than applications. A QUIC
implementation might support SCONE, but the intervening layers do not provide
SCONE APIs. For example, a browser could use a third-party QUIC implementation
that supports SCONE, but not provide the JavaScript APIs to enable and process
SCONE. If the QUIC implementation could directly return feedback to the sender,
then only application support at the sender is required.

This document proposes an extension to the QUIC protocol defining a new QUIC
frame that echoes received SCONE feedback directly to the sender at the QUIC
layer.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Overview {#overview}

The use of SCONE_ECHO frames is negotiated by new transport parameters
separately in each direction. This negotiation is an alternate means of allowing
the sending of SCONE packets, in addition to scone_supported from {{SCONE}}. In
each direction, sending SCONE is authorized by the new transport parameters or
scone_supported, never both.

When an endpoint receives a valid SCONE packet that has been authorized by the
SCONE echo parameters, it sends a SCONE_ECHO QUIC frame in response and sends no 
signal to its local application layer.

Upon receipt of a valid SCONE_ECHO packet, the SCONE sender reports the bandwidth
advice to its local application layer. 

There are no changes to SCONE Network Element behavior or the SCONE packet
format from {{SCONE}}. SCONE packets are still only valid if another QUIC packet
in the UDP datagram is successfully decrypted.

# The SCONE_ECHO Frame {#scone-echo}

An endpoint uses the SCONE_ECHO frame to return the 6-bit value encoded in a
SCONE packet. The conditions for sending it are described in {{overview}}.

~~~
SCONE_ECHO Frame {
  Type (i) = 0xff005345,
  Packet Number (i),
  Zeros (2),
  Throughput Advice (6),
}
~~~

Packet Number: the full (62-bit) packet number of the first successfully
decrypted QUIC packet in the UDP datagram that contained the SCONE header.

Zeros: These two bits MUST be zero and MUST be ignored on receipt.

Throughput Advice: The Rate Signal in the SCONE packet as encoded in Section
5 of {{SCONE}}.

A SCONE sender SHOULD keep track of the Packet Numbers to which it prepended
SCONE headers, and MUST ignore any SCONE_ECHO frames where it does not have
a record of prepending SCONE to that packet number. It might not store such
numbers when it hits storage limitations or receives duplicate SCONE_ECHO
frames.

SCONE_ECHO frames are retransmittable and MUST only appear in 1-RTT packets,
because a succesfully decrypted 1-RTT packet indicates all transport
parameters have been verified. However, the Packet Number field can refer
to a packet number in any packet number space.

# Negotiating SCONE Echo

This document specifies two new transport parameters: scone_echo_send and
scone_echo_receive.

Endpoints send scone_echo_send if and only if a local application has not
requested bandwidth advice from incoming SCONE packets (in which case an
endpoint would instead send scone_supported from SCONE). An endpoint MUST
NOT send both scone_echo_send and scone_supported; doing so is a
TRANSPORT_PARAMETER_ERROR.

Endpoints send scone_echo_receive when a local application requests sending
of SCONE packets. It indicates the ability to process SCONE_ECHO frames.

Endpoints MUST NOT send SCONE packets unless the peer has sent either
scone_supported or scone_echo_send. If scone_echo_send, the endpoint MUST
also have sent scone_echo_receive.

Endpoints MUST NOT send SCONE_ECHO frames unless the peer has sent
scone_echo_receive.

scone_echo_send and scone_echo_receive MUST be empty. If not empty, it MUST be
treated as a connection error of type TRANSPORT_PARAMETER_ERROR.

These transport parameters are valid for QUIC Version 1 {{?RFC9000}}, QUIC
Version 2 {{?RFC9369}}, and any other version that supports SCONE as outlined
in Section 6 of {{SCONE}}.

These transport parameters MUST NOT be stored for 0-RTT purposes.

## The SCONE indicator

A client that sends the scone_echo_send or scone_echo_receive transport
parameter MUST send the SCONE Indicator as described in Section 6.1 of
{{SCONE}}, whether or not it also sends scone_supported. Its semantic meaning
remains unchanged.

# Security Considerations

The security considerations in Section 9 of {{SCONE}} apply.

# Privacy Considerations

Section 10 of {{SCONE}} describes the potential privacy exposure of using SCONE.
Requiring application-layer engagement provides an additional layer of consent
to this exposure, although such engagement may not extend to the actual user.

This document envisions SCONE Echo being enabled by default in some QUIC
implementations. This might actually obscure application fingerprinting, but it
also further distances consent from the user.

# IANA Considerations

## scone_echo_send Transport Parameter

The document registers the scone_echo_supported transport parameter in the "QUIC
Transport Parameters" registry maintained at
https://www.iana.org/assignments/quic, following the guidance from Section 22.3
of {{QUIC}}.

Value:
0xff002200

Parameter Name:
scone_echo_send

Status:
Provisional

Specification:
This document

Date:
This date

Change Controller:
IETF (iesg@ietf.org)

Contact:
QUIC Working Group (quic@ietf.org)

Notes:
(none)

## scone_echo_receive Transport Parameter

The document registers the scone_echo_supported transport parameter in the "QUIC
Transport Parameters" registry maintained at
https://www.iana.org/assignments/quic, following the guidance from Section 22.3
of {{QUIC}}.

Value:
0xff002201

Parameter Name:
scone_echo_receive

Status:
Provisional

Specification:
This document

Date:
This date

Change Controller:
IETF (iesg@ietf.org)

Contact:
QUIC Working Group (quic@ietf.org)

Notes:
(none)

## SCONE_ECHO frame

This document registers the SCONE_ECHO frame in the "QUIC Frame Types" registry.

value: 0xff005345

name: SCONE_ECHO

Status: Provisional

Specification: {{scone-echo}}

Date: This date

Change Controller: IETF (iesg@ietf.org)

Contact: QUIC Working Group (quic@ietf.org)

Pkts: 1-RTT


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
