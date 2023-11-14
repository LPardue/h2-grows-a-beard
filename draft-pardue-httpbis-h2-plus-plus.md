---
title: "HTTP/2 Plus Plus"
abbrev: "H2++"
category: info

docname: draft-pardue-httpbis-h2-plus-plus-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "HTTP"
keyword:
 - next generation
 - riker
 - beard
venue:
  group: "HTTP"
  type: "Working Group"
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "LPardue/h2-grows-a-beard"
  latest: "https://LPardue.github.io/h2-grows-a-beard/draft-pardue-httpbis-h2-plus-plus.html"

author:
 -
    fullname: "Lucas Pardue"
    organization: Cloudflare
    email: "lucapardue.24.7@gmail.com"

normative:
  RFC9110:
    display: HTTP
  RFC9113:
    display: HTTP/2

informative:



--- abstract

This document defines HTTP/2 Plus Plus, an HTTP mapping to TCP that is inspired
by HTTP/2. It shares many concepts but has a different feature set and wire image that is incompatible.


--- middle

# Introduction

HTTP/2 {{RFC9113}} is an optimzed expression of the semantics of HTTP {{RFC9110}} for TCP {{?TCP=RFC793}}.

Several features defined in HTTP/2 have not been widely deployed. The second
revision of {{RFC9113}} deprecated these features in a way that would avoid
interoperabilty issues. However, some of the complexity for dealing with this,
especially the possibility of speaking with a peer that tries to use deprecated
features, is pushed to implementations. Furthermore, HTTP/2 extensions that
replace features (whether deprecated or not) require careful writing to cover
all scenarios of usage.

HTTP/2 Plus Plus takes the best bits of HTTP/2 and addresses issues that are
hard to fix in a backwards-compitible and easily-deployable way. It is not wire
compatible with HTTP/2 and so uses a different APLN ID. However, the concepts
shared between HTTP/2 and HTTP/2 Plus Plus mean that it is possible to translate between versions.


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
