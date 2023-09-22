---
stand_alone: true
ipr: trust200902
cat: info # Check
submissiontype: IETF
area: sec
wg: oauth

docname: draft-tulshibagwale-oauth-transaction-tokens-03

title: Transaction Tokens
abbrev: Txn-Tokens
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
- ins: K. Burgin
  name: Dr. Kelley W. Burgin, PhD.
  org: MITRE Corporation
  email: kburgin@mitre.org

- ins: H. Tschofenig
  name: Hannes Tschofenig
  org: Arm Ltd.
  email: Hannes.Tschofenig@arm.com

- ins: E. Gilman
  name: Evan Gilman
  org: SPIRL
  email: evan@spirl.com

normative:
  RFC2119: # Keywords
  RFC8446: # TLS
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

Transaction Tokens (Txn-Tokens) enable workloads in a trusted domain to ensure that user identity and authorization context of an external programmatic request, such as an API invocation, are preserved and available to all workloads that are invoked as part of processing such a request. Txn-Tokens also enable workloads within the trusted domain to optionally immutably assert to downstream workloads that they were invoked in the call chain of the request.

--- middle

# Introduction

Modern computing architectures often use multiple independently running components called workloads. In many cases, external invocations through externally visible interfaces such as APIs result in a number of internal workloads being invoked in order to process the external invocation. These workloads often run in virtually or physically isolated networks. These networks and the workloads running within their perimeter may be compromised by attackers through software supply chain, privileged user compromise or other attacks. Workloads compromised through external attacks, malicious insiders or software errors can cause any or all of the following unauthorized actions:

* Invocations of workloads in the network without any external invocation being present
* Arbitrary user impersonation 
* Parameter modification or augmentation

The results of these actions are unauthorised access to resources.

# Overview
Transaction Tokens (Txn-Token) are a means to mitigate damage from such attacks or spurious invocations. A valid Txn-Token indicates a valid external invocation.
They ensure that the identity of the user or a workload that made the external request is preserved throughout subsequent workload invocations.
They preserve any context such as:

* Parameters of the original call
* Environmental factors, such as IP address of the original caller
* Any computed context that needs to be preserved in the call chain. This includes information that was not in the original request to the external endpoint.

Cryptographically protected Txn-Tokens ensure that downstream workloads cannot make unauthorized modifications to such information, and cannot make spurious calls without the presence of an external trigger.

## What are Transaction Tokens?
Txn-Tokens are short-lived, signed JWTs {{RFC7519}} that assert the identity of a user or a workload and assert an authorization context. The authorization context provides information expected to remain constant during the execution of a call as it passes through multiple workloads.

When necessary, a Txn-Token may include embedded tokens, as described in {{JWTEmbeddedTokens}}. This is called a Nested Txn-Token. This nesting enables workloads in a call chain to assert their invocation during the call chain to downstream workloads.

## Creating Txn-Tokens

### Leaf Txn-Tokens
Txn-Tokens are typically created when a workload is invoked using an endpoint that is externally visible, and is authorized using a separate mechanism, such as an OAuth {{RFC6749}} access token or an OpenID Connect {{OpenIdConnect}} ID token. We call this token a "Leaf Txn-Token". This workload then performs an OAuth 2.0 Token Exchange {{RFC8693}} to obtain a Txn-Token. To do this, it invokes a special Token Service (the Txn-Token Service) and provides context that is sufficient for it to generate a Txn-Token. This context MAY include:

* The external authorization token (e.g., the OAuth access token)
* A previously created Txn-Token (leaf or nested)
* Parameters that are required to be bound for the duration of this call
* Additional context, such as the incoming IP address, User Agent information, or other context that can help the Txn-Token Service to issue the Txn-Token

The Txn-Token Service responds to a successful invocation by generating a Txn-Token. The calling workload then uses the Txn-Token to authorize its calls to subsequent workloads. Subsequent workloads may obtain Txn-Tokens of their own.

### Nested Txn-Tokens
A Nested Txn-Token is a means for a workloads to record their processing of a Txn-Token and for downstream workloads to verify that a certain upstream workload has been invoked in the call chain.

A workload within the call chain of such an external call MAY generate a new Nested Txn-Token. To generate the Nested Txn-Token, it creates a self-signed JWT Embedded Token {{JWTEmbeddedTokens}} that includes the received Txn-Token by value. Subsequent workloads can therefore know that the signing workload was in the path of the call chain.

### Replacement Txn-Tokens
A service within a call chain may choose to replace the Txn-Token. This can typically happen due to the following reasons:

* The current Txn-Token has become bloated (due to lots of nesting)
* The service wants to add to the context of the current Txn-Token
* The service wants to remove some intermediate signers' information to avoid leaking information about internal systems

To get a replacement Txn-Token, a service will request a new Txn-Token from the Txn-Token Service and provide the current Txn-Token and other parameters in the request. The Txn-Token service must exercise caution in what kinds of replacement requests it supports so as to not negate the entire value of Txn-Tokens.

## Txn-Token Lifetime
Txn-Tokens are expected to be short-lived (order of minutes, e.g., 5 minutes), and as a result MAY be used only for the expected duration of an external invocation. If a long-running process such as an batch or offline task is involved, it can use a separate mechanism to perform the external invocation, but the resulting Txn-Token is still short-lived.

## Benefits of Txn-Tokens
Txn-Tokens help prevent spurious invocations by ensuring that a workload receiving an invocation can independently verify the user or workload on whose behalf an external call was made and any context relevant to the processing of the call. Through the presence of additional signatures on the Txn-Token, a workload receiving an invocation can also independently verify that specific workloads were within the path of the call before it was invoked.

## Txn-Token Issuance and Usage Flows

### Basic Flow {#basic-flow}
{{fig-arch-basic}} shows the basic flow of how Txn-Tokens are used in an a multi-workload environment.

~~~ ascii-art
                                                     
     1    ┌──────────────┐    2      ┌──────────────┐
─────────▶│              ├───────────▶              │
          │   External   │           │  Txn-Token   │
     7    │   Endpoint   │    3      │   Service    │
◀─────────┤              ◀───────────│              │
          └────┬───▲─────┘           └──────────────┘
               │   │                                 
             4 │   │ 6                               
          ┌────▼───┴─────┐                           
          │              │                           
          │   Internal   │                           
          │  µ-service   │                           
          │              │                           
          └────┬───▲─────┘                           
               │   │                                 
               ▼   │                                 
                 o                                   
             5   o    6                              
                 o                                   
               │   ▲                                 
               │   │                                 
          ┌────▼───┴─────┐                           
          │              │                           
          │   Internal   │                           
          │  µ-service   │                           
          │              │                           
          └──────────────┘                                        
~~~
{: #fig-arch-basic title="Basic Transaction Tokens Architecture"}

1. External endpoint is invoked using conventional authorization mechanism such as an OAuth 2.0 Access token
2. External endpoint provides context and incoming authorization (e.g., access token) to the Txn-Token Service
3. Txn-Token Service mints a Txn-Token that provides immutable context for the transaction and returns it to the requester
4. The external endpoint initiates a call to an internal microservice and provides the Txn-Token as authorization
5. Subsequent calls to other internal microservices use the same Txn-Token to authorize calls
6. Responses are provided to callers based on successful authorization by the invoked microservices
7. External client is provided a response to the external invocation

### Nested Txn-Token Flow

{{fig-arch-nested}} shows an internal microservice generating a Nested Txn-Token in the flow

~~~ ascii-art
                                                     
     1    ┌──────────────┐    2      ┌──────────────┐
─────────▶│              ├───────────▶              │
          │   External   │           │  Txn-Token   │
     9    │   Endpoint   │    3      │   Service    │
◀─────────┤              ◀───────────│              │
          └────┬───▲─────┘           └──────────────┘
               │   │                                 
             4 │   │ 8                               
          ┌────▼───┴─────┐                           
          │              │                           
          │   Internal   │                           
          │  µ-service   │                           
          │              │                           
          └────┬───▲─────┘                           
               │   │                                 
               ▼   │                                 
                 o                                   
             5   o    8                              
               │ o ▲                                 
               │   │                                 
               │   │                                 
          ╔════▼═══╩═════╗                           
          ║              ╠──────┐                    
          ║   Internal   ║      │ 6                  
          ║  µ-service   ║      │                    
          ║              ◀──────┘                    
          ╚════╦═══▲═════╝                           
               │   │                                 
               ▼   │                                 
                 o                                   
             7   o    8                              
                 o                                   
               │   ▲                                 
               │   │                                 
          ┌────▼───┴─────┐                           
          │              │                           
          │   Internal   │                           
          │  µ-service   │                           
          │              │                           
          └──────────────┘                                          
~~~
{: #fig-arch-nested title="Flow with Nested Txn-Token generating service"}

In the diagram above, steps 1-5 are the same as in {{basic-flow}}.

{:start="6"}

6. An internal microservice determines it needs to generate a Nested Txn-Token. It uses its own private key to generate a Nested Txn-Token
7. The internal microservice uses the Nested Txn-Token to authorize calls to downstream services
8. Responses are provided to callers based on successful authorization by the invoked microservices
9. External client is provided a response to the external invocation

### Replacement Txn-Token Flow

An intermediate service may decide to obtain a replacement Txn-Token from the Txn-Token service. That flow is described below in {{fig-arch-replacement}}

~~~ ascii-art
                                                     
     1    ┌──────────────┐    2      ┌──────────────┐
─────────▶│              ├───────────▶              │
          │   External   │           │              │
     10   │   Endpoint   │    3      │              │
◀─────────┤              ◀───────────│              │
          └────┬───▲─────┘           │              │
               │   │                 │              │
             4 │   │ 9               │              │
          ┌────▼───┴─────┐           │              │
          │              │           │              │
          │   Internal   │           │              │
          │  µ-service   │           │              │
          │              │           │              │
          └────┬───▲─────┘           │  Txn-Token   │
               │   │                 │   Service    │
               ▼   │                 │              │
                 o                   │              │
             5   o    9              │              │
               │ o ▲                 │              │
               │   │                 │              │
               │   │                 │              │
          ┌────▼───┴─────┐    6      │              │
          │              ├───────────▶              │
          │   Internal   │           │              │
          │  µ-service   │    7      │              │
          │              ◀───────────│              │
          └────┬───▲─────┘           │              │
               │   │                 │              │
               ▼   │                 └──────────────┘
                 o                                   
             8   o    9                              
                 o                                   
               │   ▲                                 
               │   │                                 
          ┌────▼───┴─────┐                           
          │              │                           
          │   Internal   │                           
          │  µ-service   │                           
          │              │                           
          └──────────────┘                           
~~~
{: #fig-arch-replacement title="Replacement Txn-Token Flow"}

In the diagram above, steps 1-5 are the same as in {{basic-flow}}

{:start="6"}

6. An intermediate service determines that it needs to obtain a Replacement Txn-Token. It requests a Replacement Txn-Token from the Txn-Token Service. It passes the incoming Txn-Token in the request, along with any additional context it needs to send the Txn-Token Service.
7. The Txn-Token Service responds with a replacement Txn-Token
8. The service that requested the Replacement Txn-Token uses that Txn-Token for downstream call authorization
9. Responses are provided to callers based on successful authorization by the invoked microservices
10. External client is provided a response to the external invocation

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
: A virtually or physically separated network, which contains two or more workloads. The workloads within an Trust Domain may be invoked only through published interfaces. A Trust Domain must have an identifier that is used as the `aud` (audience) value in Txn-Tokens. The format of this identifier is a universal resource identifier. Each Trust Domain has exactly one Txn-Token Service.

External Endpoint:
: A published interface to an Trust Domain that results in the invocation of a workload within the Trust Domain.

Call Chain:
: A sequence of invocations that results from the invocation of an external endpoint.

Transaction Token (Txn-Token):
: A signed JWT that has a short lifetime, which provides immutable information about the user or workload, certain parameters of the call and certain contextual attributes of the call. A Txn-Token may contain a nested Txn-Token.

Leaf Txn-Token:
: A Txn-Token that does not contain a `txn_token` claim in its JWT body.

Nested Txn-Token:
: A JWT Embedded Token {{JWTEmbeddedTokens}} that embeds a Txn-Token by value.

Authorization Context:
: A JSON object containing a set of claims that represent the immutable context of a call chain.

Transaction Token Service (Txn-Token Service):
: A special service within the Trust Domain, which issues Txn-Tokens to requesting workloads. Each Trust Domain has exactly one Txn-Token Service.

# Txn-Token Format
A Txn-Token is a JSON Web Token {{RFC7519}} protected by a JSON Web Signature {{RFC7515}}. The following describes the required values in a Txn-Token:

## JWT Header {#txn-token-header}
In the JWT Header:

* The `typ` claim MUST be present and MUST have the value `txn_token`.
* Key rotation of the signing key SHOULD be supported through the use of a `kid` claim.

{{figtxtokenheader}} is a non-normative example of the JWT Header of a Txn-Token

~~~ json
{
    "typ": "txn_token",
    "alg": "RS256",
    "kid": "identifier-to-key"
}
~~~
{: #figtxtokenheader title="Example: Txn-Token Header"}

## JWT Body {#txn-token-body}

### Common Claims
The JWT body MUST have the following claims regardless of whether the Txn-Token is a Leaf Txn-Token or a Nested Txn-Token:

* An `iss` claim, whose value is a URN {{RFC8141}} that uniquely identifies the workload or the Txn-Token Service that created the Txn-Token.
* An `iat` claim, whose value is the time at which the Txn-Token was created.
* An `exp` claim, whose value is the time at which the Txn-Token expires. Note that if this claim is in a Nested Txn-Token, then this `exp` value MUST NOT exceed the `exp` value of the Txn-Token included in the JWT Body.

### Leaf Txn-Token Claims {#txn-token-claims}
The following claims MUST be present in the JWT body of a Leaf Txn-Token:

* A `txn` claim, whose value is the unique identifier of entire call chain.
* A `sub_id` claim, whose value is the unique identifier of the user or workload on whose behalf the call chain is being executed. The format of this claim MAY be a Subject Identifier as specified in {{SubjectIdentifiers}}.
* An `azc` claim, whose value is a JSON object that contains values that remain constant in the call chain.

{{figleaftxtokenbody}} shows a non-normative example of the JWT body of a Leaf Txn-Token:

~~~ json
{
    "iss": "https://trust-domain.example/txn-token-service",
    "iat": "1686536226000",
    "exp": "1686536526000",
    "txn": "97053963-771d-49cc-a4e3-20aad399c312",
    "sub_id": {
        "format": "email",
        "email": "user@trust-domain.example"
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
{: #figleaftxtokenbody title="Example: Leaf Txn-Token Body"}


### Nested Txn-Token Claim
A Nested Txn-Token is a JWT Embedded Token {{JWTEmbeddedTokens}}, which embeds a Txn-Token by value. The following claims MUST be present in a Nested Txn-Token:

* A `type` claim, whose value is `urn:ietf:params:oauth:token-type:txn_token`.
* A `token` claim, whose value is an encoded JWT representation of a Txn-Token.

{{fignestedtxtokenbody}} shows a non-normative example the JWT body of a nested Txn-Token

~~~ json
{
    "iss": "https://trust-domain.example/fraud-detection",
    "iat": "1686536236000",
    "exp": "1686536526000",
    "type": "urn:ietf:params:oauth:token-type:txn_token",
    "token": "eyJ0eXAiOiJ0cmF0Iiwi...thwd8"
}
~~~
{: #fignestedtxtokenbody title="Example: Nested Txn-Token Body"}

# Txn-Token Service
A Txn-Token Service provides a OAuth 2.0 Token Exchange {{RFC8693}} endpoint that can respond to Txn-Token issuance requests. The token exchange requests it supports require extra parameters than those defined in the OAuth 2.0 Token Exchange {{RFC8693}} specification. The unique properties of the Txn-Token requests and responses are described below. The Txn-Token Service MAY optionally support other OAuth 2.0 endpoints and features, but that is not a requirement for it to be a Txn-Token Service.

Each Trust Domain MUST have exactly one Txn-Token Service.

# Requesting Leaf Txn-Tokens
A workload requests a Txn-Token from a Transaction Token Service using OAuth 2.0 Token Exchange {{RFC8693}}. The request to obtain a Txn-Token using this method is called a Txn-Token Request, and a successful response is called a Txn-Token Response. A Txn-Token Request is a Token Exchange Request, as described in Section 2.1 of {{RFC8693}} with additional parameters. A Txn-Token Response is a OAuth 2.0 token endpoint response, as described in Section 5 of {{RFC6749}}, where the `token_type` in the response has the value `txn_token`.

## Txn-Token Request {#txn-token-request}
A Txn-Token Request is an OAuth 2.0 Token Exchange Request, as described in Section 2.1 of {{RFC8693}}, with an additional parameter in the request. The following parameters are required in the Txn-Token Request by the OAuth 2.0 Token Exchange specification {{RFC8693}}:

* The `audience` value MUST be set to the Trust Domain name
* The `requested_token_type` value MUST be `urn:ietf:params:oauth:token-type:txn_token`
* The `subject_token` value MUST be the external token received by the workload that authorized the call
* The `subject_token_type` value MUST be present and indicate the type of the authorization token present in the `subject_token` parameter

The following additional parameter MUST be present in a Txn-Token Request:

* A parameter named `rctx` , whose value is a JSON object. This object contains the request context, i.e. any information the Transaction Token Service needs to understand the context of the incoming request

{{figtxtokenrequest}} shows a non-normative example of a Txn-Token Request.

~~~ http
POST /txn-token-service/token_endpoint HTTP 1.1
Host: txn-token-service.trust-domain.example
Content-Type: application/x-www-form-urlencoded

requested_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Atxn_token
&audience=http%3A%2F%2Ftrust-domain.example
&subject_token=eyJhbGciOiJFUzI1NiIsImtpZC...kdXjwhw
&subject_token_type=urn%3Aietf%3Aparams%3Aoauth%3Atoken-type%3Aaccess_token
&rctx=%7B%22param1%22%3A%22value1%22%2C%22param2%22%3A%22value2%22%2C%22ip_address%22%3A%2269.151.72.123%22%7D
~~~
{: #figtxtokenrequest title="Example: Txn-Token Request"}


## Txn-Token Response {#txn-token-response}
A successful response to a Txn-Token Request by a Transaction Token Service is called a Txn-Token Response. If the Transaction Token Service responds with an error, the error response is as described in Section 5.2 of {{RFC6749}}. The following describes required values of a Txn-Token Response:

* The `token_type` value MUST be set to `txn_token`
* The `access_token` value MUST be the Txn-Token
* The response MUST NOT include the values `expires_in`, `refresh_token` and `scope`

{{figtxtokenresponse}} shows a non-normative example of a Txn-Token Response.

~~~ http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-cache, no-store

{
  "issued_token_type": "urn:ietf:params:oauth:token-type:txn_token",
  "access_token": "eyJCI6IjllciJ9...Qedw6rx"
}
~~~
{: #figtxtokenresponse title="Example: Txn-Token Response"}

## Creating Replacement Txn-Tokens
A workload within a call chain may request the Transaction Token Server to replace a Txn-Token. Replacement Txn-Tokens are Leaf Txn-Tokens.

Workloads MAY request replacement Txn-Tokens in order to change (add to, remove or modify) the asserted values within a Txn-Token, to remove nesting and / or reduce token bloat.

### Txn-Token Service Responsibilities
A Txn-Token Service replacing a Txn-Token must consider that modifying previously asserted values from existing Txn-Tokens can completely negate the benefits of Txn-Tokens. When issuing replacement Txn-Tokens, a Transaction Token Server therefore:

* MAY enable modifications to asserted values that reduce the scope of permitted actions
* MAY enable reduction of token bloat by removing nesting, and placing workload identifiers as asserted values instead
* MAY enable additional asserted values 
* SHOULD NOT enable modification to asserted values that expand the scope of permitted actions

### Replacement Txn-Token Request
To request a replacement Txn-Token, the requester makes a Txn-Token Request as described in {{txn-token-request}} but includes the Txn-Token to be replaced as the value of the `subject_token` parameter.

### Replacement Txn-Token Response
A successful response by the Transaction Token Server to a Replacement Txn-Token Request is a Txn-Token Response as described in {{txn-token-response}}

### Removing Nesting
A Replacement Txn-Token Request MAY include a Nested Txn-Token in its request. If the request is successful, the Transaction Token Server SHALL always respond with a Leaf Txn-Token. 

If the Replacement Txn-Token Request has a Nested Txn-Token in the request's `subject_token` parameter, then the Transaction Token Server MAY include information about services that had signed the Nested Txn-Token that is requested to be replaced.

If the Transaction Token Server wishes to include information about any nested Txn-Token signers, then it SHALL include a field named `previous_signers` in the `azc` value of the Txn-Token that it issues. The value of this field MUST be an array of strings. Each string is the value of the `iss` field of a Nested Txn-Tokens received in the Replacement Txn-Token Request. Note that:

* A Nested Txn-Token is a recursive structure, and the `iss` value is present at each level of nesting
* The Transaction Token Server MAY choose to include or exclude any `iss` value in the `previous_signers` field of the Txn-Token it generates

## Mutual Authentication of the Txn-Token Request
A Txn-Token Service MUST ensure that it authenticates any workloads requesting Txn-Tokens. In order to do so:

* It MUST name a limited, pre-configured set of workloads that MAY request Txn-Tokens
* It MUST individually authenticate the requester as being one of the named requesters
* It SHOULD rely on mechanisms, such as {{Spiffe}} or some other means of performing MTLS {{RFC8446}}, to securely authenticate the requester
* It SHOULD NOT rely on insecure mechanisms, such as long-lived shared secrets to authenticate the requesters

The requesting workload MUST have a pre-configured location for the Transaction Token Service. It SHOULD rely on mechanisms, such as {{Spiffe}}, to securely authenticate the Transaction Token Service before making a Txn-Token Request.

# Creating Nested Txn-Tokens
A workload within a call chain MAY create a Nested Txn-Token. It does so by embedding the Txn-Token it receives by value in a JWT Embedded Token {{JWTEmbeddedTokens}}. Nested Txn-Tokens are self-signed and not requested from a separate service. 

The expiration time of a enclosing Txn-Token MUST NOT exceed the expiration time of an embedded Txn-Token.

# IANA Considerations {#IANA}

This specification registers the following claims defined in Section {{txn-token-header}} to the OAuth Access Token Types Registry defined in {{RFC6749}}, and the following claims defined in Section {{txn-token-claims}} in the IANA JSON Web Token Claims Registry defined in {{RFC7519}}

## OAuth Registry Contents

* Name: `txn_token`
* Description: JWT of type Transaction Token
* Additional Token Endpoint Response Parameters: none
* HTTP Authentication Schemes: TLS {{RFC8446}}
* Change Controller: IESG
* Specification Document: Section {{txn-token-header}} of this specificaiton

## JWT Registry Contents

* Claim Name: `azc`
* Claim Description: The authorization context
* Change Controller: IESG
* Specification Document: Section {{txn-token-claims}} of this specification

# Security Considerations {#Security}

## Txn-Token Lifetime
A Txn-Token is not resistant to replay attacks. A long-lived Txn-Token therefore represents a risk if it is stored in a file, discovered by an attacker, and then replayed. For this reason, a Txn-Token lifetime must be kept short, not exceeding the lifetime of a call-chain. Even for long-running "batch" jobs, a longer lived access token should be used to initiate the request to the batch endpoint. It then obtains short-lived Txn-Tokens that may be used to authorize the call to downstream services in the call-chain.

Because Txn-Tokens are short-lived, the Txn-Token response from the Txn-Token service does not contain the `refresh_token` field. A Txn-Token is also cannot be issued by presenting a `refresh_token`.

The `expires_in` and `scope` fields of the OAuth 2.0 Token Exchange specification {{RFC8693}} are also not used in Txn-Token responses. The `expires_in` is not required since the issued token has an `exp` field, which indicates the token lifetime. The `scope` field is omitted from the request and therefore omitted in the response.

## Sender Constrained Tokens
Although Txn-Tokens are short-lived, they MAY be sender constrained as an additional layer of defence to prevent them from being re-used by a compromised or malicious workload under the control of a hostile actor. 

## Access Tokens
When creating Txn-Tokens, the Txn-Token MUST NOT contain the Access Token presented to the external endpoint. If an Access Token is included in a Txn-Token, an attacker may extract the Access Token from the Txn-Token, and replay it to any Resource Server that can accept that Access Token. Txn-Token expiry does not protect against this attack since the Access Token may remain valid even after the Txn-Token has expired.

--- back

# Acknowledgements {#Acknowledgements}
{: numbered="false"}




