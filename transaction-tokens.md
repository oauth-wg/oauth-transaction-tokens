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
  
- ins: P. Kasselman
  name: Pieter Kasselman
  org: Microsoft
  email: pieter.kasselman@microsoft.com
  
contributor:
- ins: E. Gilman
  name: Evan Gilman
  org: SPIRL
  email: evan@spirl.com

normative:
  RFC2119: # Keywords
  RFC6749: #OAuth
  RFC7519: #JWT
  RFC7515: #JWS
  RFC8141: # URN
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

Transaction Tokens (TraTs) enable workloads in a trusted domain to ensure that user identity and authorization context of an external programmatic request, such as an API invocation, is preserved and available to all workloads that are invoked as part of processing such a request. TraTs also enable workloads within the trusted domain to optionally immutably assert to downstream workloads that they were invoked in the call chain of the request.

--- middle

# Introduction

Modern computing architectures often use multiple independently running components called workloads. In many cases, external invocations through externally visible interfaces such as APIs result in a number of internal workloads being invoked in order to process the external invocation. These workloads often run in virtually or physically isolated networks. These networks and the workloads running within their perimeter may be compromised by attackers through software supply chain, privileged user compromise or other attacks. Workloads compromised through external attacks, malicious insiders or software errors can cause any or all of the following unauthorized actions:

* Invocations of workloads in the network without any external invocation being present
* Arbitrary user impersonation 
* Parameter modification or augmentation

The results of these actions are unauthorised access to resources. 

# Overview
Transaction Tokens are a means to mitigate damage from such attacks or spurious invocations. A valid TraT indicates a valid external invocation; 
They ensure that the identity of the user or robotic principal (e.g. workload) that made the external request is preserved throughout subsequent workload invocations; And they preserve any context such as:

* Parameters of the original call
* Environmental factors such as IP address of the original caller
* Any computed context that needs to be preserved in the call chain

All this ensures that downstream workloads cannot make unauthorized modifications to such information, and cannot make spurious calls without the presence of an external trigger.

## What are Transaction Tokens
Transaction Tokens (TraTs) are short-lived, signed JWTs {{RFC7519}} that assert the identity of a user or robotic principal (e.g. workload) and assert an authorization context. The authorization context provides information expected to remain constant during the execution of a call as it passes through multiple workloads.

### Nesting
Alternatively, a TraT MAY be a signed JWT that has a nested TraT in its body. This nesting enables workloads in a call chain to assert their invocation during the call chain to downstream workloads.

## Creating TraTs

### Leaf TraTs
Leaf TraTs are typically created when a workload is invoked using an endpoint that is externally visible, and is authorized using a separate mechanism, such as an OAuth {{RFC6749}} token or an OpenID Connect {{OpenIdConnect}} token. This workload then performs an OAuth 2.0 Token Exchange {{RFC8693}} to obtain a TraT. To do this, it invokes a special Token Service (the TraT Service) and provides context that is sufficient for it to generate a TraT. This context MAY include:

* The external authorization token (e.g. the OAuth token)
* Parameters that are required to be bound for the duration of this call
* Additional context such as incoming IP Address, User Agent, or other information that can help the TraT Service to issue the TraT

The TraT Service responds to a successful invocation by generating a TraT. The calling workload then uses the TraT to authorize its calls to subsequent workloads. Subsequent workloads may obtain TraTs of their own

Figure 1 below shows how TraTs are used in an a multi-workload environment.

~~~ ascii-art
                              
             +--------------+   (B) TraT Request    +---------------+
(A)User  +---|  Resource    |---------------------->|               |
   Start |   |   Server     |   (C) TraT Response   |  Transaction  |
   Flow  +-->| (Workload 1) |<----------------------|     Token     |
             +--------------+                       |    Server     |
                    |                               |               |
                    | (D) Send Request              |               |
                    |     with Leaf TraT            |               |
                    |     for Workload 1            |               |
                    v                               |               |
             +--------------+                       |               |
             |  Workload 2  |                       |               |
             |              |                       |               |
             |              |                       |               |
             +--------------+                       |               |
                    |  (E) Use unmodified           |               |
                    |      Leaf TraT                |               |
                    |      for Workload 1           |               |
                    v                               |               |
             +--------------+                       |               |
             |  Workload 3  |---+ (F) Create        |               |
             |              |   |     Nested Trat   |               |
             |              |<--+                   |               |
             +--------------+                       |               |
                    |  (G) Send request with        |               |
                    |      Nested TraT for          |               |
                    |      Workload 3               |               |
                    v                               |               |
             +--------------+   (H) TraT Request    |               |
             |  Workload 4  |---------------------->|               |
             |              |   (I) TraT Response   |  Transaction  |
             |              |<----------------------|     Token     |
             +--------------+                       |    Server     |
                    |  (J) Send request with        |               |
                    |      Leaf TraT for            |               |
                    |      Workload 4               |               | 
                    :                               |               |
                    :                               |               |
                    |                               |               |
                    |                               |               |
                    |                               |               |
                    v                               |               |
             +--------------+                       |               |
             |  Workload n  |                       |               |
             |              |                       |               |
             |              |                       |               |
             +--------------+                       +---------------+

~~~
Figure: Use of TraTs in multi-workload environments

<ToDo - Update Description to match new figure - Pieter>
- (A) The user accesses a resource server and present an Access Token obtained from an Authorization Server using an OAuth 2.0 or OpenID Connect flow.
- (B) The resource server is implemented as a workload (Workload 1) and requests a Transaction Token (TraT) from the Transaction Token Server using the Token Exchange protocol {{RFC8693}}.
- (C) The Transaction Token Service (TraT Service) returns a Transaction Token containing the requested claims that establish the identity of the original caller, and additional claims that can be used to make authroization decisions and establish the call chain.
- (D) The Resource Server (Workload 1) calls Workload 2 and passes the TraT along with other parameters. Workload 2 validates the TraT and makes an authroization decision by combining contextual information at its disposal with information in the TraT to make an Authroization Decision to accept or reject the call.
- (E) Workload 2 requests a Transaction Token (TraT) from the Transaction Token Server using the Token Exchange protocol {{RFC8693}}.
- (F) The Transaction Token Service (TraT Service) returns a Transaction Token containing the requested claims that establish the identity of the original caller, workload 1 and the call chain and additional claims that can be made to make authroization decisions.
- (G-J) The pattern of sending a TraT to the next workload where it is used as part of an Authroization Decision before obtaining a new Trat for use with the next call repeats until the call chain completes.

### Nested TraTs
A workload within the call chain of such an external call MAY generate a new Nested TraT. To generate the Nested Trat, it creates a new JWT that includes the received TraT in the body and signs the JWT itself so that subsequent workloads know that the signing workload was in the path of the call chain.

## TraT Lifetime
TraTs are expected to be short-lived (order of minutes, e.g. 5 minutes), and as a result MAY be used only for the expected duration of an external invocation. If a long-running process such as an batch or offline task is involved, it can use a separate mechanism to perform the external invocation, but the resulting TraT SHALL still be short-lived.

## Benefits of TraTs
TraTs help prevent spurious invocations by ensuring that a workload receiving an invocation can independently verify the user or robotic principal on whose behalf an external call was made and any context relevant to the processing of the call. Through the presence of additional signatures on the TraT, a workload receiving an invocation can also independently verify that specific workloads were within the path of the call before it was invoked.

# Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED",
"MAY", and "OPTIONAL" in this document are to be interpreted as
described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when,
they appear in all capitals, as shown here.

# Terminology

Workload:
: An independent computational unit that can autonomously receive and process invocations, and can generate invocations of other workloads. Examples of workloads include containerized microservices, monolithic services and infrastructure services such as managed databases.

Trust Domain:
: A virtually or physically separated network which contains two or more workloads. The workloads within an Trust Domain MAY be invoked only through published interfaces. A Trust Domain MUST have a name that is used as the `aud` (audience) value in TraTs

External Endpoint:
: A published interface to an Trust Domain that results in the invocation of a workload within the Trust Domain

Call Chain:
: A sequence of invocations that results from the invocation of an external endpoint.

Transaction Token (TraT):
: A signed JWT that has a short lifetime, which provides immutable information about the user or robotic principal, certain parameters of the call and certain contextual attributes of the call. A TraT MAY contain a nested TraT

Leaf TraT:
: A TraT that does not contain a `trat` claim in its JWT body

Nested TraT:
: A TraT that contains a nested TraT in its JWT body as the value of the `trat` claim

Authorization Context:
: A JSON object containing a set of claims that represent the immutable context of a call chain

Transaction Token Service (TraT Service):
: A special service within the Trust Domain which issues Transaction Tokens to requesting workloads

# TraT Format
A TraT is a JSON Web Token {{RFC7519}} that has a JSON Web Signature {{RFC7515}}. The following is true in a TraT:

## JWT Header {#trat-header}
In the JWT Header:

* The `typ` claim MUST be present and MUST have the value `trat`
* Key rotation of the signing key SHOULD be supported through the use of a `kid` claim

Below is a non-normative example of the JWT Header of a TraT

~~~ json
{
    "typ": "trat",
    "alg": "RS256",
    "kid": "identifier-to-key"
}
~~~
{: #figtratheader title="Example: TraT Header"}

## JWT Body {#trat-body}

### Common Claims
The JWT Body MUST have the following claims regardless of whether the TraT is a Leaf Trat or a Nested TraT:

* An `iss` claim, whose value is a URN {{RFC8141}} that uniquely identifies the workload or the TraT Service that created the TraT
* An `iat` claim, whose value is the time at which the TraT was created
* An `exp` claim, whose value is the time at which the TraT expires. Note that if this claim is in a Nested TraT, then this `exp` value MUST NOT exceed the `exp` value of the TraT included in the JWT Body

### Leaf TraT Claims
The following claims MUST be present in the JWT Body of a Leaf TraT

* A `tid` claim, whose value is the unique identifier of entire call chain
* A `sub_id` claim, whose value is the unique identifier of the user or robotic principal on whose behalf the call chain is being executed. The format of this claim MAY be a Subject Identifier as specified in {{SubjectIdentifiers}}
* An `azc` claim, whose value is a JSON object that contains values that remain constant in the call chain

Below is a non-normative example of the JWT Body of a Leaf TraT

~~~ json
{
    "iss": "https://trust-domain.com/trat-service",
    "iat": "1686536226000",
    "exp": "1686536526000",
    "tid": "97053963-771d-49cc-a4e3-20aad399c312",
    "sub_id": {
        "format": "email",
        "email": "user@trust-domain.com"
    },
    "azc": {
        "action": "BUY", // parameter of external call
        "ticker": "MSFT", // parameter of external call
        "quantity": "100", // parameter of external call
        "user_ip": "69.151.72.123", // env context of external call
        "user_level": "vip" // computed value not present in external call
    }
}
~~~
{: #figleaftratbody title="Example: Leaf TraT Body"}


### Nested TraT Claim
The following claim MUST be present in a Nested TraT:

* A `trat` claim, whose value is an encoded JWT representation of a TraT.

Below is a non-normative example the JWT Body of a nested TraT

~~~ json
{
    "iss": "https://trust-domain.com/fraud-detection",
    "iat": "1686536236000",
    "exp": "1686536526000",
    "trat": "eyJ0eXAiOiJ0cmF0Iiwi...thwd8"
}
~~~
{: #fignestedtratbody title="Example: Nested TraT Body"}


# Requesting TraTs
A workload requests a TraT from a TraT Service using OAuth 2.0 Token Exchange {{RFC8693}}. The request to obtain a TraT using this method is called a TraT Request, and a success response is called a TraT Response. A TraT Request is a Token Exchange Request as described in {{Section 2.1 of RFC8693}} with additional parameters. A TraT Response is a successful Token Response is a OAuth 2.0 token endpoint response as described in {{Section 5 of RFC6749}}, where the `token_type` in the response has the value `trat`.

The TraT Service acts as an OAuth 2.0 {{RFC6749}} Authorization Server. The requesting workload acts as the OAuth 2.0 Client. It authenticates itself to the TraT Service through mechanisms defined in OAuth 2.0.

## TraT Request
A TraT Request is an OAuth 2.0 Token Exchange Request as described in {{Section 2.1 of RFC8693}}, with an additional parameter in the request. The following is true of the TraT Request:

* The `audience` value MUST be set to the Trust Domain name.
* The `requested_token_type` value MUST be `urn:ietf:params:oauth:token-type:trat`
* The `subject_token` value MUST be the external token received by the workload that authorized the call
* The `subject_token_type` value MUST be present and indicate the type of the authorization token present in the `subject_token` parameter

The following additional parameters MUST be present in a TraT Request:

* A parameter named `azc` , whose value is a JSON object. This object contains any information the TraT Service needs to understand the context of the incoming request.

The following is a non-normative example of a TraT Request:

~~~ http
POST /trat-service/token_endpoint HTTP 1.1
Host: trat-service.trust-domain.com
Content-Type: application/x-www-form-urlencoded

requested_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Atrat
&audience=http%3A%2F%2Ftrust-domain.com
&subject_token=eyJhbGciOiJFUzI1NiIsImtpZC...kdXjwhw
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Aaccess_token
&azc=%7B%22param1%22%3A%22value1%22%2C%22param2%22%3A%22value2%22%2C%22ip_address%22%3A%2269.151.72.123%22%7D

~~~
{: #figtratrequest title="Example: TraT Request"}


## TraT Response
A success response to a TraT Request by a TraT Service is called a TraT Response. If the TraT Service responds with an error, the error response is as described in {{Section 5.2 of RFC6749}}. The following is true of a TraT Response:

* The `token_type` value MUST be set to `trat`
* The `access_token` value MUST be the TraT
* The response MUST NOT include the values `expires_in`, `refresh_token` and `scope`

The following is a non-normative example of a TraT Response

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: non-cache, no-store

{
  "issued_token_type": "urn:ietf:params:oauth:token-type:trat",
  "access_token": "eyJCI6IjllciJ9...Qedw6rx"
}
~~~
{: #figtratresponse title="Example: TraT Response"}


# Creating Nested TraTs
A workload within a call chain MAY create a Nested TraT. It does so by creating a new JWT that has all the header fields as described in {{trat-header}} and only a JWT Body that includes the `trat` claim as described in {{trat-body}}.

The expiration time of a enclosing TraT MAY NOT exceed the expiration time of an embedded TraT.

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

## Sender Constrained Tokens
Although TraTs are short-lived, they may be sender constrained as an additional layer of defence to prevent them from being re-used by a compromised or malicious workload under the control of a hostile actor. 

## Access Tokens
When using nested TraTs, the nested TraT MUST NOT contain the Access Token presented to the resource server. If an Access Token is included in a TraT, an attacker may obtain a TraT, extract the Access Token, and replay it to the Resource Server. TraT expiry does not protect against this attack since the Access Token may remain valid even after the TraT has expired.

--- back

# Acknowledgements {#Acknowledgements}
{: numbered="false"}


