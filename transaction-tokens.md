---
stand_alone: true
ipr: trust200902
cat: info # Check
submissiontype: IETF
area: sec
wg: oauth

docname: draft-transaction-tokens-00

title: Transaction Tokens
abbrev: trat
lang: en
kw:
  - Microservices
  - OAuth
  - JWT
  - token exchange
# date: 2022-02-02 -- date is filled in automatically by xml2rfc if not given
author:
- ins: A. Tulshibagwale
  name: Atul Tulshibagwale
  org: SGNL
  email: atul@sgnl.ai
  
- ins: G. Fletcher
  name: George Fletcher
  org: Capital One
  email: george.fletcher@capitalone.com
  
contributor:
- ins: P. Kasselman
  name: Pieter Kasselman
  org: Microsoft
  email: pieter.kasselman@microsoft.com
  
- ins: E. Gilman
  name: Evan Gilman
  org: SPIRL
  email: evan@spirl.com

normative:
  RFC2119: # Keywords
  RFC6749: #OAuth
  RFC7519: #JWT
  RFC7515: #JWS
  RFC8174: # Ambiguity in Keywords
  RFC8693: # OAuth 2.0 Token Exchange
  
  OpenIdConnect:
    title: OpenID Connect Core 1.0 incorporating errata set 1
    target: https://openid.net/specs/openid-connect-core-1_0.html
    author:
    - name: Nat Sakimura
      org: NRI
    - name: John Bradley
      org: Ping Identity
    - name: Mike Jones
      org: Microsoft
    - name: B. de Medeiros
      org: Google
    - name: Chuck Mortimore
      org: Salesforce
    date: 2014-11

  SubjectIdentifiers:
    title: Subject Identifiers for Security Event Tokens
    target: https://datatracker.ietf.org/doc/html/draft-ietf-secevent-subject-identifiers
    author:
    - name: Annabelle Backman
      org: Amazon
    - name: Martin Scurtescu
      org: Coinbase
    - name: Prachi Jain
      org: Fastly
  
informative:
  Spiffe:
    title: Secure Production Identity Framework for Everyone
    target: https://spiffe.io/docs/latest/spiffe-about/overview/
    author:
    - org: Cloud Native Computing Foundation

--- abstract

Transaction Tokens (TraTs) enable workloads in a trusted domain to ensure that identity and authorization context of an external invocation is preserved throughout any subsequent workloads that are invoked as a result of such external invocation. TraTs also enable workloads within the trusted domain to optinally assert their processing of the request to downstream workloads.

--- middle

# Introduction

Modern computing architectures often use multiple independently running components called workloads. In many cases, external invocations through externally visible interfaces such as APIs result in a number of internal workloads being invoked in order to respond to the external invocation. These workloads often run in virtually or physically isolated networks. These networks may be compromised by attackers through software supply chain or privileged user compromise attacks. Such attackers, malicious insiders or software errors within workloads could result in unauthorized invocations of any workloads, user impersonation and illegal parameter modification or augmentation.

## What are Transaction Tokens
Transaction Tokens (TraTs) are short-lived, signed JWTs {{RFC7519}} that assert the identity of a user or robotic principal and assert an authorization context. The authorization context provides information expected to remain constant during the execution of a call as it passes through multiple workloads.

### Nesting
Alternatively, a TraT MAY be a signed JWT that has a nested TraT in its body. This nesting enables workloads in a call chain to assert their invocation during the call chain to downstream workloads.

## Creating TraTs
TraTs are typically created when a workload is invoked using an endpoint that is externally visible, and is authorized using a separate mechanism, such as an OAuth {{RFC6749}} token or an OpenID Connect {{OpenIdConnect}} token. This workload then invokes a special Token Service (TraT Service) and provides context that is sufficient for itto generate a TraT. This context MAY include:

* The external authorization token (e.g. the OAuth token)
* Parameters that are required to be bound for the duration of this call
* Additional context such as incoming IP Address, User Agent, or other information that ca nhelp the TraT Service to issue the TraT

The TraT Service responds to a successful invocation by generating a TraT. The calling workload then uses the TraT to authorize its calls to subsequent workloads. A workload within the call chain of such an external call MAY sign the TraT itself so that subsequent workloads know that the signing workload was in the path of the call chain.

TraTs help prevent spurious invocations by ensuring that a workload receiving an invocation can independently verify the user or robotic principal on whose behalf an external call was made, a subset of the parameters of the call, and a subset of the intermediate workloads that were invoked during the subsequent processing of the call. Through the presence of additional signatures on the TraT, a workload receiving an invocation can also independently verify that specific workloads were within the path of the call before it was invoked.

## TraT Lifetime
TraTs are expected to be short-lived (order of minutes, e.g. 5 minutes), and as a result MAY be used only for the expected duration of an external invocation. If a long-running process such as an batch or offline task is involved, it can use a separate mechanism to perform the external invocation, but the resulting Transaction Token SHALL still be short-lived.

## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED",
"MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when,
they appear in all capitals, as shown here.

# Terminology

Workload
: An independent computational unit that can autonomously receive and process invocations, and can generate invocations of other workloads. Examples of workloads include containerized microservices, monolithic services and infrastructure services such as managed databases.

Trust Domain
: A virtually or physically separated network which contains two or more workloads. The workloads within an Trust Domain MAY be invoked only through published interfaces. A Trust Domain MUST have a name that is used as the `aud` (audience) value in TraTs

External Endpoint
: A published interface to an Trust Domain that results in the invocation of a workload within the Trust Domain

Call Chain
: A sequence of invocations that results from the invocation of an external endpoint.

Transaction Token (TraT)
: A signed JWT that has a short lifetime, which provides immutable information about the user or robotic principal, certain parameters of the call and certain contextual attributes of the call. A TraT MAY contain a nested TraT

Leaf TraT
: A TraT that does not contain a nested TraT

Authorization Context
: A JSON object containing a set of claims that represent the immutable context of a call chain

Transaction Token Service (TraT Service)
: A special service within the Trust Domain which issues Transaction Tokens to requesting workloads

# TraT Format
A TraT is a JSON Web Token {{RFC7519}} that has a JSON Web Signature {{RFC7515}}. The following is true in a TraT:

## JWT Header {#trat-header}
In the JWT Header:

* The `typ` claim MUST be present and MUST have the value `trat`
* Key rotation of the signing key SHOULD be supported through the use of `kid` claim

## JWT Body {#trat-body}

### Common Claims
The JWT Body MUST have the following claims regardless of whether the TraT contains a nested TraT or not

* The `iss` claim MUST be present. Its value MUST uniquely identify the workload or the TraT Service that created the TraT
* The `iat` claim MUST be present. Its value MUST be the time at which the TraT was created
* The `exp` claim MUST be present. Its value MUST be the time at which the TraT expires 

### Nested TraT Claim
If the TraT contains a nested TraT, then it MUST have only one claim named `trat`. The value of this claim is a JWT representing a TraT.


### Leaf TraT Claims
The following claims MUST be present in the JWT Body of a Leaf TraT

* A `tid` claim MUST be present. The value of this claim is a unique identifier of the top-level invocation
* A `sub_id` claim MUST be present. The value of this claim is the unique identifier of the user or robotic principal on whose behalf the call chain is being executed. The format of this claim MAY be a Subject Identifier as specified in {{SubjectIdentifiers}}
* A `azc` claim MUST be present. The value of this claim is a JSON object that contains parameters and context values that remain constant in the call chain

# Requesting TraTs
A workload may request a TraT from a TraT Service using the OAuth 2.0 Token Exchange {{RFC8693}}. The request to obtain a TraT using this method is called a TraT Request, and the response is called a TraT Response. A TraT Request is a Token Exchange Request as described in {{Section 2.1 of RFC8693}} with additional parameters. A TraT Response is a successful Token Response is a OAuth 2.0 token endpoint response as described in {{Section 5 of RFC6749}}, where the `token_type` in the response has the value `trat`.

The TraT Service acts as an OAuth 2.0 {{RFC6749}} Authorization Server. The requesting workload acts as the OAuth 2.0 Client. It authenticates itself to the TraT Service through mechanisms defined in OAuth 2.0.

## TraT Request
A TraT Request is a OAuth 2.0 Token Exchange Request as described in {{Section 2.1 of RFC8693}}, with an additional parameter in the request. The following is true of the TraT Request:

* The `audience` value MUST be set to the Trust Domain name.
* The `requested_token_type` value MUST be `urn:ietf:params:oauth:token-type:trat`
* The `subject_token` value MUST be the external token received by the workload that authorized the call
* The `subject_token_type` value MUST be present and indicate the type of the authorization token present in the `subject_token` parameter

The following additional parameters MUST be present in a TraT Request:
* A parameter named `azc` , whose value is a JSON object. This object contains any information the TraT Service needs to understand the context of the incoming request.

## TraT Response
A success response to a TraT Request by a TraT Service is called a TraT Response. If the TraT Service responds with an error, the error response is as described in {{Section 5.2 of RFC6749}}. The following is true of a TraT Response:

* The `token_type` value MUST be set to `trat`
* The `access_token` value MUST be the TraT
* The response MUST NOT include the values `expires_in`, `refresh_token` and `scope`

# Creating Nested TraTs
A workload within a call chain MAY create a nested TraT. It does so by creating a new JWT that has all the header fields as described in {{trat-header}} and only a JWT Body that includes the `trat` claim as described in {{trat-body}}

# IANA Considerations {#IANA}

This memo includes no request to IANA.

# Security Considerations {#Security}

## Mutual Authentication of the TraT Request
A TraT Service MUST ensure that it authenticates any workloads requesting TraTs. In order to do so:

* It MUST name a limited, pre-configured set of workloads that MAY request TraTs
* It MUST individually authenticate the requester as being one of the name requesters
* It SHOULD rely on mechanisms such as {{Spiffe}} to securely authenticate the requester
* It SHOULD NOT rely on insecure mechanisms such as long-lived shared secrets to authenticate the requesters

The requesting workload MUST have a pre-configured location for the TraT Service. It SHOULD rely on mechanisms such as {{Spiffe}} to securely authenticate the TraT Service before making a TraT Request.

--- back

# Acknowledgements {#Acknowledgements}
{: numbered="false"}


