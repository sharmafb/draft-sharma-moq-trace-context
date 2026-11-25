---
title: Trace Context Propagation for Media over QUIC Transport
abbrev: MOQT Trace Context
docname: draft-sharma-moq-trace-context-latest
category: std
submissiontype: IETF
v: 3
area: Web and Internet Transport
workgroup: Media Over QUIC

venue:
  group: Media Over QUIC
  type: Working Group
  mail: moq@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/moq/
  github: sharmafb/draft-sharma-moq-trace-context
  latest: https://sharmafb.github.io/draft-sharma-moq-trace-context/draft-sharma-moq-trace-context.html

author:
  - ins: A. Sharma
    fullname: Aman Sharma
    organization: Meta
    email: amsharma@meta.com

normative:
  MOQT: I-D.ietf-moq-transport-23
  TRACE-CONTEXT:
    title: Trace Context
    target: https://www.w3.org/TR/2021/REC-trace-context-1-20211123/
    author:
      - org: W3C
    date: 2021-11-23
  TRACE-CONTEXT-2:
    title: Trace Context Level 2
    target: https://www.w3.org/TR/2024/CRD-trace-context-2-20240328/
    author:
      - org: W3C
    date: 2024-03-28

--- abstract

This document defines a Media over QUIC Transport (MOQT) extension that carries
W3C Trace Context on SUBSCRIBE and FETCH requests. Subscribers, relays, and
publishers can use it to correlate spans across MOQT sessions.

--- middle

# Introduction

A MOQT Request ID identifies a request within one session {{MOQT}}. A relay can
issue another request on a separate session to serve it. W3C Trace Context
{{TRACE-CONTEXT}} lets tracing tools connect those operations across sessions
using two fields.

- **Traceparent** identifies the overall trace and the sender's operation
  within it. It also carries flags, including a hint about whether the sender
  recorded trace data.
- **Tracestate** carries optional vendor-specific tracing information, such as
  a tracing system's own identifiers, as an ordered list of key-value pairs.

{::boilerplate bcp14-tagged}

# Negotiation {#negotiation}

This extension applies to `moqt-23`. An endpoint advertises support in SETUP
with the TRACE_CONTEXT_SUPPORT Setup Option (odd type TBD, zero-length value).
An endpoint MUST have sent and received this option on the session before
sending a TRACE_CONTEXT parameter.

# TRACE_CONTEXT Parameter {#parameter}

The optional TRACE_CONTEXT Message Parameter (type TBD) can appear once in a
SUBSCRIBE or FETCH. Its value uses the length-prefixed encoding in {{MOQT}}.

~~~
TRACE_CONTEXT Value {
  Length (vi64),
  Traceparent Length (vi64),
  Traceparent (..),
  Tracestate (..),
}
~~~

Length counts the bytes after itself. Traceparent Length counts the bytes of
Traceparent. Tracestate occupies the remaining bytes and is omitted when none
remain. Both fields use the ASCII header field values from {{TRACE-CONTEXT}}.
The `trace-flags` field follows {{TRACE-CONTEXT-2}}, including its rule for
preserving the random trace ID flag when continuing a trace.

The receiver parses the context within the validated outer Length. It MUST
ignore the context if Traceparent Length cannot be decoded, exceeds the
remaining bytes, or yields an unusable Traceparent under the W3C rules.
If only Tracestate is invalid, the receiver MUST ignore Tracestate and retain
Traceparent. MOQT framing and parameter errors follow {{MOQT}}.

# Relay Processing

A relay issuing a new upstream request SHOULD propagate usable context from
one downstream request that triggered it, if the upstream session negotiated
the extension and local policy permits it. A relay that participates in the
trace keeps the trace ID and uses its outgoing span ID as the new parent ID,
following the W3C mutation rules. A relay can also forward both fields unchanged.

A relay can trace a cache hit locally or link downstream requests to a shared
upstream operation in its telemetry. Trace context MUST NOT affect which
upstream requests the relay issues, request aggregation, or cache identity.

# Shared Subscription Example {#shared-subscription-example}

Two clients subscribe to the same Track through `relay_1`. Each starts its own
trace, and all three sessions have negotiated the extension. The arrows are
numbered in order and labeled with `(trace ID, parent ID)`.

~~~
          (1) SUBSCRIBE             (2) SUBSCRIBE
              (A, C1)                   (A, R1)
client_1 ----------------> relay_1 ----------------> publisher
                              ^
                              |
client_2 ---------------------+
          (3) SUBSCRIBE
              (B, C2)
~~~

The relay continues Trace A on its upstream SUBSCRIBE with its own outgoing
span ID, R1. When `client_2` subscribes, the relay reuses that upstream
subscription and can link Trace B to it in local telemetry. Each SUBSCRIBE
carries its context once, even if the subscription delivers many Objects.

# Security Considerations

Trace context is untrusted metadata and MUST NOT affect authorization. Endpoints
apply local sampling limits and MAY omit context or restart a trace at a trust
boundary. A restart discards inherited Tracestate. Trace parameters MUST NOT
contain personal or sensitive data or be copied into shared Track or Object
Properties.

# IANA Considerations

IANA is requested to register TRACE_CONTEXT_SUPPORT (odd type TBD) in the
"MOQT Setup Options" registry and TRACE_CONTEXT (type TBD) in the
"MOQT Message Parameters" registry, referencing {{negotiation}} and
{{parameter}}, respectively.
