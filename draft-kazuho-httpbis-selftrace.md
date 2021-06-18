---
title: Self-Tracing for HTTP
docname: draft-kazuho-httpbis-selftrace-latest
category: info

ipr: trust200902
area: Applications and Real-Time
workgroup: HTTP
keyword: Internet-Draft

stand_alone: yes
pi: [toc, docindent, sortrefs, symrefs, strict, compact, comments, inline]

author:
  -
    ins: K. Oku
    name: Kazuho Oku
    org: Fastly
    email: kazuhooku@gmail.com
  -
    ins: J. Iyengar
    name: Jana Iyengar
    org: Fastly
    email: jri.ietf@gmail.com

normative:

informative:
  FETCH:
    target: https://fetch.spec.whatwg.org
    title: Fetch - Living Standard
    author:
     -
        org: WHATWG


--- abstract

This document registers a "Well-Known URI" for exposing state of an HTTP
connection to the peer using formats such as qlog schema
{{?QLOG=I-D.ietf-quic-qlog-main-schema-00}}.

--- middle

# Introduction

One challenge regarding HTTP ({{!HTTP=I-D.ietf-httpbis-messaging}}) performance
or stability analysis is obtaining the sender-side trace of connections.
End-users of HTTP who face issues do not have access to the server-side traces.
It is also difficult for server-side operators to retain enough amount of
fine-grained traces that they can consult when their end-users report issues.
Also, there are privacy concerns regarding retaining fine-grained traces.

This challenge can be overcome if the server exposes the trace of each HTTP
connection on that same connection. When users experience issues, they can
report to the server operators with the traces that they obtained on the HTTP
connections that suffered. The privacy concern is mitigated as the users will be
submitting the trace actively.


## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in {{!RFC2119}}.


# The Self-Trace Well-Known URI

When a server receives a GET request at the Self-Trace Well-Known URI, it starts
streaming event information that occur on the connection on which that request
was received.

Note however that, depending on the type of the trace being collected, sending
chunks of a trace might cause additonal events that in turn generate more chunks
that have to be sent. To prevent this infinite feedback loop, a server SHOULD
suspend the transmission of self-trace when the self-trace becomes the only HTTP
request being inflight on an HTTP connection. When a new HTTP request is opened
on the same HTTP connection, the server can resume the transmission of the
trace.


# Security Considerations

## Cross-Origin Attacks

To prevent cross-origin attacks, web browser access to the self-trace MUST be
resticted to the same origin {{FETCH}}.


## Coalescing Proxy Acting as Client

When a forward proxy that coalesces HTTP requests from multiple end-clients
connect to an HTTP server that can serve the self-trace, and if one of the
end-clients request the self-trace, the provided trace might contain information
regarding requests bein issued by other end-clients.

To prevent this attack, servers SHOULD serve self-trace only when HTTPS is being
used. The assumption here is that when HTTPS is being used, end-clients are
directly connected to the server.


## Connections Serving Multiple Origins

Sometimes, reverse proxies are configured as such that one HTTP connection can
be used for serving multiple origins maintained by different entities (e.g., CDN
using an X.509 certificate that contains multiple customers). In such
deployments, a malicious origin might use a script running on a web browser to
fetch the self-trace that conains traffic information related to other origins
colocated, then upload the fetched trace to extract information.

To prevent such an attack, self-trace SHOULD only be provided from an origin
that is maintained by the operator of the reverse proxy.


# IANA Considerations

This specification registers the following value in the "Well-Known URIs"
registry established by {{!RFC5785}}:

URI suffix: self-trace

Change controller: IETF

Specification document(s): this document

Related information: N/A

--- back

# Acknowledgements

In {{?I-D.benfield-http2-debug-state}}, Cory Benfield presented the idea of
exposing a snapshot of an HTTP connection to the client. The key difference from
that proposal is that this specification defines a way to "stream" the states as
they change.
