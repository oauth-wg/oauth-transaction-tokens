---
stand_alone: true
ipr: trust200902
cat: info # Check
submissiontype: IETF
area: sec
wg: oauth

docname: draft-tulshibagwale-oauth-transaction-tokens-00

title: Transaction Tokens
abbrev: Tx-Tokens
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
      
  JWTEmbeddedTokens:
    title: JSON Web Token (JWT) Embedded Tokens
    target: https://www.ietf.org/archive/id/draft-yusef-oauth-nested-jwt-06.html
    author:
    - name: Rifaat Shekh-Yusef
      org: E&Y
    - name: Dick Hardt
      org: Hellō
    - name: Giuseppe De Marco
      org: Dipartimento per la trasformazione digitale della Presidenza del Consiglio dei Ministri Italy
  
informative:
  Spiffe:
    title: Secure Production Identity Framework for Everyone
    target: https://spiffe.io/docs/latest/spiffe-about/overview/
    author:
    - org: Cloud Native Computing Foundation

--- abstract

Transaction Tokens (Tx-Tokens) enable workloads in a trusted domain to ensure that user identity and authorization context of an external programmatic request, such as an API invocation, is preserved and available to all workloads that are invoked as part of processing such a request. Tx-Tokens also enable workloads within the trusted domain to optionally immutably assert to downstream workloads that they were invoked in the call chain of the request.

--- middle

# Introduction

Modern computing architectures often use multiple independently running components called workloads. In many cases, external invocations through externally visible interfaces such as APIs result in a number of internal workloads being invoked in order to process the external invocation. These workloads often run in virtually or physically isolated networks. These networks and the workloads running within their perimeter may be compromised by attackers through software supply chain, privileged user compromise or other attacks. Workloads compromised through external attacks, malicious insiders or software errors can cause any or all of the following unauthorized actions:

* Invocations of workloads in the network without any external invocation being present
* Arbitrary user impersonation 
* Parameter modification or augmentation

The results of these actions are unauthorised access to resources.

# Overview
Transaction Tokens (Tx-Token) are a means to mitigate damage from such attacks or spurious invocations. A valid Tx-Token indicates a valid external invocation.
They ensure that the identity of the user or a workload that made the external request is preserved throughout subsequent workload invocations.
They preserve any context such as:

* Parameters of the original call
* Environmental factors, such as IP address of the original caller
* Any computed context that needs to be preserved in the call chain. This includes information that was not in the original request to the external endpoint.

Cryptographically protected Tx-Token ensure that downstream workloads cannot make unauthorized modifications to such information, and cannot make spurious calls without the presence of an external trigger.

## What are Transaction Tokens?
Tx-Tokens are short-lived, signed JWTs {{RFC7519}} that assert the identity of a user or a workload and assert an authorization context. The authorization context provides information expected to remain constant during the execution of a call as it passes through multiple workloads.

When necessary, a Tx-Token may include embedded tokens, as described in {{JWTEmbeddedTokens}}. This is called Nested Tx-Token. This nesting enables workloads in a call chain to assert their invocation during the call chain to downstream workloads.

## Creating Tx-Tokens

### Leaf Tx-Tokens
Tx-Tokens are typically created when a workload is invoked using an endpoint that is externally visible, and is authorized using a separate mechanism, such as an OAuth {{RFC6749}} token or an OpenID Connect {{OpenIdConnect}} token. We call this token "Leaf Tx-Token". This workload then performs an OAuth 2.0 Token Exchange {{RFC8693}} to obtain a Tx-Token. To do this, it invokes a special Token Service (the Tx-Token Service) and provides context that is sufficient for it to generate a Tx-Token. This context MAY include:

* The external authorization token (e.g. the OAuth access token)
* Parameters that are required to be bound for the duration of this call
* Additional context, such as the incoming IP address, User Agent information, or other context that can help the Tx-Token Service to issue the Tx-Token

The Tx-Token Service responds to a successful invocation by generating a Tx-Token. The calling workload then uses the Tx-Token to authorize its calls to subsequent workloads. Subsequent workloads may obtain Tx-Tokens of their own.

### Nested Tx-Tokens
A workload within the call chain of such an external call MAY generate a new Nested Tx-Token. To generate the Nested Tx-Token, it creates a self-signed JWT Embedded Token {{JWTEmbeddedTokens}} that includes the received Tx-Token by value. Subsequent workloads can therefore know that the signing workload was in the path of the call chain.

## Tx-Token Lifetime
Tx-Tokens are expected to be short-lived (order of minutes, e.g. 5 minutes), and as a result MAY be used only for the expected duration of an external invocation. If a long-running process such as an batch or offline task is involved, it can use a separate mechanism to perform the external invocation, but the resulting Tx-Token SHALL still be short-lived.

## Benefits of Tx-Tokens
Tx-Tokens help prevent spurious invocations by ensuring that a workload receiving an invocation can independently verify the user or workload on whose behalf an external call was made and any context relevant to the processing of the call. Through the presence of additional signatures on the Tx-Token, a workload receiving an invocation can also independently verify that specific workloads were within the path of the call before it was invoked.

## Tx-Token Issuance and Usage Flows

{{fig-arch}} shows how Tx-Tokens are used in an a multi-workload environment.

~~~ ascii-art
                    | (A) Access Token
                    | via external API
                    v
             +--------------+ (B) Tx-Token Request  +---------------+
             |  Resource    |---------------------->|               |
             |   Server     | (C) Leaf Tx-Token     |  Transaction  |
             | (Workload 1) |<----------------------|     Token     |
             +--------------+                       |    Server     |
                    |                               |               |
                    | (D) Send Request              |               |
                    |     with Leaf Tx-Token        |               |
                    |     for Workload 1            |               |
                    v                               |               |
             +--------------+                       |               |
             |  Workload 2  |                       |               |
             |              |                       |               |
             |              |                       |               |
             +--------------+                       |               |
                    |  (E) Use unmodified           |               |
                    |      Leaf Tx-Token            |               |
                    |      for Workload 1           |               |
                    v                               |               |
             +--------------+                       |               |
             |  Workload 3  |---+ (F) Create        |               |
             |              |   |     Nested        |               |
             |              |<--+     Tx-Token      |               |
             +--------------+                       |               |
                    |  (G) Send request with        |               |
                    |      Nested Tx-Token for      |               |
                    |      Workload 3               |               |
                    v                               |               |
             +--------------+ (H) Tx-Token Request  |               |
             |  Workload 4  |---------------------->|               |
             |              | (I) Tx-Token Response |  Transaction  |
             |              |<----------------------|     Token     |
             +--------------+                       |    Server     |
                    |  (J) Send request with        |               |
                    |      Leaf Tx-Token for        |               |
                    |      Workload 5               |               |
                    :                               |               |
                    :                               |               |
                    |                               |               |
                    |                               |               |
                    |                               |               |
                    v                               |               |
             +--------------+                       |               |
             |  Workload n  | (K) Tx-Token verified |               |
             |              |     by last workload  |               |
             |              |     in call chain     |               |
             +--------------+                       +---------------+
~~~
{: #fig-arch title="Use of Tx-Tokens in Multi-Workload Environments"}

- (A) The user accesses a resource server and present an Access Token obtained from an Authorization Server using an OAuth 2.0 or an OpenID Connect flow.
- (B) The resource server is implemented as a workload (Workload 1) and requests a Leaf Tx-Token from the Transaction Token Server using the Token Exchange protocol {{RFC8693}}.
- (C) The Transaction Token Service returns a Leaf Tx-Token containing the requested claims that establish the identity of the original caller as well as additional claims that can be used to make authorization decisions and establish the call chain.
- (D) The Resource Server (Workload 1) calls Workload 2 and passes the Leaf Tx-Token for Workload 1. Workload 2 validates the Tx-Token and makes an authorization decision by combining contextual information at its disposal with information in the Tx-Token to make an authorization decision to accept or reject the call.
- (E) Workload 2 is not required to add aditional information to the Tx-Token and passes the unmodified Tx-Token for Workload 1 to Workload 3. Workload 3 validates the Tx-Token and makes an authorization decision by combining contextual information at its disposal with information in the Tx-Token to make an authorization decision to accept or reject the call.
- (F) Workload 3 generates a Nested Tx-Token that includes additional call chain information.
- (G) Workload 3 sends the Nested Tx-Token to Workload 4. Workload 4 validates the Nested Tx-Token and makes an authorization decision by combining contextual information at its disposal with information in the Nested Tx-Token to make an authorization decision to accept or reject the call.
- (H) Workload 4 needs a Tx-Token containing information from the Authroization Server and requests a new Leaf Transaction Token (Leaf Tx-Token) from the Transaction Token Server using the Token Exchange protocol {{RFC8693}}.
- (I) The Transaction Token Service returns a Leaf Transaction Token (Leaf Tx-Token) containing the requested claims that include the call chain information included in the Tx-Token as well as additional claims needed.
- (J) Workload 4 sends the Tx-Token to the Workload 5, who verifies it and extracts claims and combine it with contextual information for use in authroization decisions. Other workloads continue to pass Tx-Tokens, generate Nested Tx-Tokens or request new Tx-Tokens.
- (K) Workload n is the final workload in the call chain. It verifies the received Tx-Token, extracts claims and combine it with contextual information for use in authroization decisions. 

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
: A virtually or physically separated network, which contains two or more workloads. The workloads within an Trust Domain may be invoked only through published interfaces. A Trust Domain must have a name that is used as the `aud` (audience) value in Tx-Tokens.

External Endpoint:
: A published interface to an Trust Domain that results in the invocation of a workload within the Trust Domain.

Call Chain:
: A sequence of invocations that results from the invocation of an external endpoint.

Transaction Token (Tx-Token):
: A signed JWT that has a short lifetime, which provides immutable information about the user or workload, certain parameters of the call and certain contextual attributes of the call. A Tx-Token may contain a nested Tx-Token.

Leaf Tx-Token:
: A Tx-Token that does not contain a `tx_token` claim in its JWT body.

Nested Tx-Token:
: A JWT Embedded Token {{JWTEmbeddedTokens}} that embeds a Tx-Token by value.

Authorization Context:
: A JSON object containing a set of claims that represent the immutable context of a call chain.

Transaction Token Service (Tx-Token Service):
: A special service within the Trust Domain, which issues Tx-Tokens to requesting workloads.

# Tx-Token Format
A Tx-Token is a JSON Web Token {{RFC7519}} protected by a JSON Web Signature {{RFC7515}}. The following is true in a Tx-Token:

## JWT Header {#tx-token-header}
In the JWT Header:

* The `typ` claim MUST be present and MUST have the value `tx_token`.
* Key rotation of the signing key SHOULD be supported through the use of a `kid` claim.

{{figtxtokenheader}} is a non-normative example of the JWT Header of a Tx-Token

~~~ json
{
    "typ": "tx_token",
    "alg": "RS256",
    "kid": "identifier-to-key"
}
~~~
{: #figtxtokenheader title="Example: Tx-Token Header"}

## JWT Body {#tx-token-body}

### Common Claims
The JWT body MUST have the following claims regardless of whether the Tx-Token is a Leaf Tx-Token or a Nested Tx-Token:

* An `iss` claim, whose value is a URN {{RFC8141}} that uniquely identifies the workload or the Tx-Token Service that created the Tx-Token.
* An `iat` claim, whose value is the time at which the Tx-Token was created.
* An `exp` claim, whose value is the time at which the Tx-Token expires. Note that if this claim is in a Nested Tx-Token, then this `exp` value MUST NOT exceed the `exp` value of the Tx-Token included in the JWT Body.

### Leaf Tx-Token Claims
The following claims MUST be present in the JWT body of a Leaf Tx-Token:

* A `tid` claim, whose value is the unique identifier of entire call chain.
* A `sub_id` claim, whose value is the unique identifier of the user or workload on whose behalf the call chain is being executed. The format of this claim MAY be a Subject Identifier as specified in {{SubjectIdentifiers}}.
* An `azc` claim, whose value is a JSON object that contains values that remain constant in the call chain.

{{figleaftxtokenbody}} shows a non-normative example of the JWT body of a Leaf Tx-Token:

~~~ json
{
    "iss": "https://trust-domain.com/tx-token-service",
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
{: #figleaftxtokenbody title="Example: Leaf Tx-Token Body"}


### Nested Tx-Token Claim
A Nested Tx-Token is a JWT Embedded Token {{JWTEmbeddedTokens}}, which embeds a Tx-Token by value. The following claims MUST be present in a Nested Tx-Token:

* A `type` claim, whose value is `urn:ietf:params:oauth:token-type:tx_token`.
* A `token` claim, whose value is an encoded JWT representation of a Tx-Token.

{{fignestedtxtokenbody}} shows a non-normative example the JWT body of a nested Tx-Token

~~~ json
{
    "iss": "https://trust-domain.com/fraud-detection",
    "iat": "1686536236000",
    "exp": "1686536526000",
    "type": "urn:ietf:params:oauth:token-type:tx_token",
    "token": "eyJ0eXAiOiJ0cmF0Iiwi...thwd8"
}
~~~
{: #fignestedtxtokenbody title="Example: Nested Tx-Token Body"}


# Requesting Leaf Tx-Tokens
A workload requests a Tx-Token from a Transaction Token Service using OAuth 2.0 Token Exchange {{RFC8693}}. The request to obtain a Tx-Token using this method is called a Tx-Token Request, and a success response is called a Tx-Token Response. A Tx-Token Request is a Token Exchange Request, as described in Section 2.1 of {{RFC8693}} with additional parameters. A Tx-Token Response is a OAuth 2.0 token endpoint response, as described in Section 5 of {{RFC6749}}, where the `token_type` in the response has the value `tx_token`.

The Transaction Token Service acts as an OAuth 2.0 {{RFC6749}} Authorization Server. The requesting workload acts as the OAuth 2.0 Client, which authenticates itself to the Transaction Token Service through mechanisms defined in OAuth 2.0.

## Tx-Token Request
A Tx-Token Request is an OAuth 2.0 Token Exchange Request, as described in Section 2.1 of {{RFC8693}}, with an additional parameter in the request. The following is true of the Tx-Token Request:

* The `audience` value MUST be set to the Trust Domain name.
* The `requested_token_type` value MUST be `urn:ietf:params:oauth:token-type:tx_token`.
* The `subject_token` value MUST be the external token received by the workload that authorized the call.
* The `subject_token_type` value MUST be present and indicate the type of the authorization token present in the `subject_token` parameter.

The following additional parameter MUST be present in a Tx-Token Request:

* A parameter named `azc` , whose value is a JSON object. This object contains any information the Transaction Token Service needs to understand the context of the incoming request.

{{figtxtokenrequest}} shows a non-normative example of a Tx-Token Request.

~~~ http
POST /tx-token-service/token_endpoint HTTP 1.1
Host: tx-token-service.trust-domain.com
Content-Type: application/x-www-form-urlencoded

requested_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Atx_token
&audience=http%3A%2F%2Ftrust-domain.com
&subject_token=eyJhbGciOiJFUzI1NiIsImtpZC...kdXjwhw
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Aaccess_token
&azc=%7B%22param1%22%3A%22value1%22%2C%22param2%22%3A%22value2%22%2C%22ip_address%22%3A%2269.151.72.123%22%7D
~~~
{: #figtxtokenrequest title="Example: Tx-Token Request"}


## Tx-Token Response
A successful response to a Tx-Token Request by a Transaction Token Service is called a Tx-Token Response. If the Transaction Token Service responds with an error, the error response is as described in Section 5.2 of {{RFC6749}}. The following is true of a Tx-Token Response:

* The `token_type` value MUST be set to `tx_token`.
* The `access_token` value MUST be the Tx-Token.
* The response MUST NOT include the values `expires_in`, `refresh_token` and `scope`.

{{figtxtokenresponse}} shows a non-normative example of a Tx-Token Response.

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: non-cache, no-store

{
  "issued_token_type": "urn:ietf:params:oauth:token-type:tx_token",
  "access_token": "eyJCI6IjllciJ9...Qedw6rx"
}
~~~
{: #figtxtokenresponse title="Example: Tx-Token Response"}


# Creating Nested Tx-Tokens
A workload within a call chain MAY create a Nested Tx-Token. It does so by embedding the Tx-Token it receives by value in a JWT Embedded Token {{JWTEmbeddedTokens}}. Nested Tx-Tokens are self-signed and not requested from a separate service. 

The expiration time of a enclosing Tx-Token MAY NOT exceed the expiration time of an embedded Tx-Token.

# IANA Considerations {#IANA}

This memo includes no request to IANA.

# Security Considerations {#Security}

## Mutual Authentication of the Tx-Token Request
A Transaction Token Service MUST ensure that it authenticates any workloads requesting Tx-Tokens. In order to do so:

* It MUST name a limited, pre-configured set of workloads that MAY request Tx-Tokens.
* It MUST individually authenticate the requester as being one of the name requesters.
* It SHOULD rely on mechanisms, such as {{Spiffe}}, to securely authenticate the requester.
* It SHOULD NOT rely on insecure mechanisms, such as long-lived shared secrets to authenticate the requesters.

The requesting workload MUST have a pre-configured location for the Transaction Token Service. It SHOULD rely on mechanisms, such as {{Spiffe}}, to securely authenticate the Transaction Token Service before making a Tx-Token Request.

## Sender Constrained Tokens
Although Tx-Tokens are short-lived, they may be sender constrained as an additional layer of defence to prevent them from being re-used by a compromised or malicious workload under the control of a hostile actor. 

## Access Tokens
When creating Tx-Tokens, the Tx-Token MUST NOT contain the Access Token presented to the resource server. If an Access Token is included in a Tx-Token, an attacker may obtain a Tx-Token, extract the Access Token, and replay it to the Resource Server. Tx-Token expiry does not protect against this attack since the Access Token may remain valid even after the Tx-Token has expired.

--- back

# Acknowledgements {#Acknowledgements}
{: numbered="false"}



