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
cases.

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

Both endpoints send the scone_echo_supported transport parameter to indicate
support for the frame. If one endpoint does not, no SCONE_ECHO frames may be
sent on the connection in either direction.

Endpoints SHOULD also include the scone_supported transport parameter from
{{SCONE}} if an application has indicated it supports SCONE. If an endpoint
sends this parameter, it MUST NOT send SCONE_ECHO frames regardless of the
presence of a scone_echo_supported transport parameter. It might receive such
frames, however, if the peer did not send scone_supported.

Endpoints can send SCONE packets if the peer sends scone_supported, OR if both
endpoints send scone_echo_supported. If the peer only sent scone_echo_supported,
an endpoint MUST NOT send SCONE packets if its application cannot accept the
resulting Throughput Advice.

An endpoint responds to a SCONE packet based on the transport parameters it
sent, assuming the SCONE packet is coalesced with a succesfully decrypted QUIC
packet.

* If it sent scone_supported, it passes the result to its application and
it takes no further action at the QUIC layer without explicit application
instruction.

* If it sent scone_echo_supported but not scone_supported, it responds with a
SCONE_ECHO frame.

* If it sent neither, it ignores the SCONE packet.

When an endpoint receives a SCONE_ECHO frame and the transport parameters do
not indicate such a frame can be sent, it terminates the connection with
PROTOCOL_VIOLATION. Otherwise, if the frame passes validity checks, it passes
the information up to the application for further action.

There are no changes to SCONE Network Element behavior or the SCONE packet 
format from {{SCONE}}.

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

Support for this extension is indicated by the scone_echo_supported transport
parameter (0xff00219f). Both endpoints must send this parameter to allow
SCONE_ECHO frames in either direction. SCONE_ECHO frames MUST NOT be sent by
an endpoint that also sends the scone_supported transport parameter.

scone_echo_supported MUST be empty. If it is not empty, it MUST be treated as
a connection error of type TRANSPORT_PARAMETER_ERROR.

This transport parameter is valid for QUIC Version 1 {{?RFC9000}}, QUIC
Version 2 {{?RFC9369}}, and any other version that supports SCONE as outlined
in Section 6 of {{SCONE}}.

This transport parameter MUST NOT be stored for 0-RTT purposes.

## The SCONE indicator

A client that sends the scone_echo_supported transport parameter MUST send the
SCONE Indicator as described in Section 6.1 of {{SCONE}}, whether or not it also
sends scone_supported. Its semantic meaning remains unchanged.

# Security Considerations

The security considerations in Section 9 of {{SCONE}} apply.

Both sides have to send the transport parameter. If only the receiver has to
send it, and an attacker injects a SCONE packet, that will trigger a
SCONE_ECHO frame that will terminate the connection.

# Privacy Considerations

Section 10 of {{SCONE}} describes the potential privacy exposure of using SCONE.
Requiring application-layer engagement provides an additional layer of consent
to this exposure, although such engagement may not extend to the actual user.

This document envisions SCONE Echo being enabled by default in some QUIC
implementations. This might actually obscure application fingerprinting, but it
also further distances consent from the user.

# IANA Considerations

## scone_echo_supported Transport Parameter

The document registers the scone_echo_supported transport parameter in the "QUIC
Transport Parameters" registry maintained at
https://www.iana.org/assignments/quic, following the guidance from Section 22.3
of {{QUIC}}.

Value:
0xff00219f

Parameter Name:
scone_echo_supported

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
