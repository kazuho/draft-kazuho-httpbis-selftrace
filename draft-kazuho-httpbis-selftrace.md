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

This document defines, registers, and describes the use of a well-known URI
{{!RFC8615}} for streaming a server's view of an HTTP connection to the
requesting client.

--- middle

# Introduction

An endpoint's view of an HTTP connection is limited to its own state; neither
endpoint is privy to the peer's view of the connection. This limitation has some
familiar and common consequences, at both clients and servers.

A client that experiences functional or performance issues with a server has no
visibility into the server's connection state. A user who wishes to report such
performance issues cannot provide more information than the time of the
connection and perhaps the IP address of the client.

Server operators often do not proactively retain detailed logs of all client
connections, and as a result, they may not have adequate information to
meaningfully investigate a client's report. Proactively maintaining detailed
logs has challenges for a server operator: storage or CPU considerations can
cause servers to retain only sampled logs or limit the extent of logging. If
retained, server logs are often sanitized of client-specific
personally-identifiable information, making it difficult to identify specific
connections reported as problematic by users.

In this document, we describe a client-driven server reporting mechanism, called
"self tracing", to overcome these limitations.

## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in {{!RFC2119}}.


# Self Tracing

Self tracing is a simple mechanism: a client requests a special URI (see
{{self-trace-uri}}, which the server interprets as a request to trace the
connection on which the request was received and responds by continuously
streaming the server's view of the connection.

Our key insight is that this mechanism eliminates the aforementioned issues of
resource scaling and client privacy: a server chooses what information it shares
with a client, and a user makes an active choice to share that information along
with any issue reports to the server operator.

We note that this mechanism does not eliminate the need for server-side logging,
since not all errors will be reproducible and recordable by a user, and some
classes of issues, such as connection establishment issues, are simply not
possible to log using this mechanism.

While this mechanism is usable by all HTTP versions, it is most useful to the
ones that support request multiplexing. In HTTP/2 and HTTP/3, a self-trace
request results in a stream becoming dedicated for the streaming response from
the server.


# The Self-Trace Well-Known URI {#self-trace-uri}

This specification registers the '.well-known/self-trace' well-known URI to be
used for self tracing.

For example, the self-tracing URI on 'http://www.example.com/' would be
'http://www.example.com/.well-known/self-trace'.

When a server receives a GET request for the self-tracing URI, it SHOULD start
streaming the server's connection and streams view for the connection on which
that request was received.

A server MAY respond with the following errors.
XXX Error responses

A self-trace response is terminated when either endpoint terminates the stream
or connection on which the request or response is being sent.


# Implementation Considerations

Sending log lines in a self trace might cause additional events that in turn
generate more log lines to be sent, resulting in an infinite feedback loop of
sending. For example, this can happen when a server's self trace is a streaming
event log, such as a streaming qlog trace
{{?QLOG=I-D.ietf-quic-qlog-main-schema}}. 

To prevent this infinite feedback loop, a server SHOULD suspend transmission of
a self-trace response when this response is the only one in flight in the
connection. The server SHOULD resume transmission of the self trace When a
subsequent HTTP request is received on the same HTTP connection.

Multiple concurrent self-trace requests can similarly result in an infinite
feedback loop. A server SHOULD handle at most one concurrent self-trace request
on a connection.


# Security Considerations

## Cross-Origin Attacks

To prevent cross-origin attacks, Web browser access to self tracing MUST be
resticted to the same origin {{FETCH}}.


## Coalescing Proxy Acting as Client

A forward proxy can coalesce HTTP requests from multiple clients. When such a
proxy connects to an HTTP server that can self-trace and if one of the proxied
clients requests a self-trace, the server's trace to the client might contain
connection and stream information on requests and responses issued by and for
other proxied clients.

To prevent this information leak, servers SHOULD restrict use of self tracing to
only HTTPS connections, since the use of HTTPS through a forward proxy results
in each proxied client establishing a separate connection to the server.


## Connections Serving Multiple Origins

Reverse proxies are sometimes configured to allow one HTTP connection to be used
for serving multiple origins maintained by different entities. For example, this
can happen when a multi-tenant CDN server uses a single X.509 certificate for
multiple customers.

Under such circumstances, a malicious origin can use a script running on a web
browser to fetch a self trace that contains information about the other
co-located origins and upload the collected trace back to the malicious origin.

To address this attack, reverse proxies that forward HTTP requests to multiple
origins belonging to different entities MUST do one of the following:

* serve self trace from an origin maintained by the operator of the reverse
  proxy,

* serve self-trace only when requests for one origin is inflight on a given
  connection, or

* disable self tracing.


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
exposing a snapshot of an HTTP connection to the client. A key difference from
that proposal is that this specification defines a way for a server to
continuously stream whatever state it deems useful through the life of a
connection.
