---
title: "Profiling EDHOC for CoAP and OSCORE"
abbrev: "Profiling EDHOC for CoAP and OSCORE"
docname: draft-ietf-core-oscore-edhoc-latest
cat: std

ipr: trust200902
area: Internet
workgroup: CoRE Working Group
keyword: Internet-Draft

stand_alone: yes
coding: utf-8
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: F. Palombini
    name: Francesca Palombini
    organization: Ericsson
    email: francesca.palombini@ericsson.com
 -
    ins: M. Tiloca
    name: Marco Tiloca
    org: RISE AB
    street: Isafjordsgatan 22
    city: Kista
    code: SE-16440 Stockholm
    country: Sweden
    email: marco.tiloca@ri.se
 -
    ins: R. Hoeglund
    name: Rikard Hoeglund
    org: RISE AB
    street: Isafjordsgatan 22
    city: Kista
    code: SE-16440 Stockholm
    country: Sweden
    email: rikard.hoglund@ri.se
 -
    ins: S. Hristozov
    name: Stefan Hristozov
    organization: Fraunhofer AISEC
    email: stefan.hristozov@aisec.fraunhofer.de
 -
    ins: G. Selander
    name: Goeran Selander
    organization: Ericsson
    email: goran.selander@ericsson.com

normative:
  RFC2119:
  RFC7252:
  RFC8174:
  RFC8613:
  RFC8742:
  RFC8949:
  I-D.ietf-lake-edhoc:

informative:



--- abstract

This document defines an optimization approach for combining the lightweight authenticated key exchange protocol EDHOC run over CoAP with the first subsequent OSCORE transaction. This combination reduces the number of round trips required to set up an OSCORE Security Context and to complete an OSCORE transaction using that Security Context.

--- middle

# Introduction

Ephemeral Diffie-Hellman Over COSE (EDHOC) {{I-D.ietf-lake-edhoc}} is a lightweight authenticated key exchange protocol, especially intended for use in constrained scenarios. In particular, EDHOC messages can be transported over the Constrained Application Protocol (CoAP) {{RFC7252}} and used for establishing a Security Context for Object Security for Constrained RESTful Environments (OSCORE) {{RFC8613}}.

This document defines an optimization approach that combines EDHOC run over CoAP with the first subsequent OSCORE transaction. This allows for a minimum number of round trips necessary to setup the OSCORE Security Context and complete an OSCORE transaction, for example when an IoT device gets configured in a network for the first time.

This optimization is desirable, since the number of protocol round trips impacts on the minimum number of flights, which in turn can have a substantial impact on the latency of conveying the first OSCORE request, when using certain radio technologies.

Without this optimization, it is not possible, not even in theory, to achieve the minimum number of flights. This optimization makes it possible also in practice, since the last message of the EDHOC protocol can be made relatively small (see {{Section 1.3 of I-D.ietf-lake-edhoc}}), thus allowing additional OSCORE protected CoAP data within target MTU sizes.

## Terminology

{::boilerplate bcp14}

The reader is expected to be familiar with terms and concepts defined in CoAP {{RFC7252}}, CBOR {{RFC8949}}, CBOR sequences {{RFC8742}}, OSCORE {{RFC8613}} and EDHOC {{I-D.ietf-lake-edhoc}}.

# EDHOC Overview {#overview}

The EDHOC protocol allows two peers to agree on a cryptographic secret, in a mutually-authenticated way and by using Diffie-Hellman ephemeral keys to achieve perfect forward secrecy. The two peers are denoted as Initiator and Responder, as the one sending or receiving the initial EDHOC message_1, respectively.

After successful processing of EDHOC message_3, both peers agree on a cryptographic secret that can be used to derive further security material, and especially to establish an OSCORE Security Context {{RFC8613}}. The Responder can also send an optional EDHOC message_4 to achieve key confirmation, e.g., in deployments where no protected application message is sent from the Responder to the Initiator.

{{Section A.3 of I-D.ietf-lake-edhoc}} specifies how to transfer EDHOC over CoAP. That is, the EDHOC data (referred to as "EDHOC messages") are transported in the payload of CoAP requests and responses. The default message flow consists in the CoAP Client acting as Initiator and the CoAP Server acting as Responder. Alternatively, the two roles can be reversed. In the rest of this document, EDHOC messages are considered to be transferred over CoAP.

{{fig-non-combined}} shows a CoAP Client and Server running EDHOC as Initiator and Responder, respectively. That is, the Client sends a POST request to a reserved EDHOC resource at the Server, by default at the Uri-Path "/.well-known/edhoc". The request payload consists of the CBOR simple value "true" (0xf5) concatenated with EDHOC message_1, which also includes the EDHOC connection identifier C_I of the Client.

This triggers the EDHOC exchange at the Server, which replies with a 2.04 (Changed) response. The response payload consists of EDHOC message_2, which also includes the EDHOC connection identifier C_R of the Server. The Content-Format of the response may be set to "application/edhoc".

Finally, the Client sends a POST request to the same EDHOC resource used earlier to send EDHOC message_1. The request payload consists of the EDHOC connection identifier C_R, concatenated with EDHOC message_3.

After this exchange takes place, and after successful verifications as specified in the EDHOC protocol, the Client and Server can derive an OSCORE Security Context, as defined in {{Section A.2 of I-D.ietf-lake-edhoc}}. After that, they can use OSCORE to protect their communications as per {{RFC8613}}.

~~~~~~~~~~~~~~~~~
   CoAP Client                                       CoAP Server
(EDHOC Initiator)                                 (EDHOC Responder)
        |                                                  |
        |                                                  |
        | ----------------- EDHOC Request ---------------> |
        |   Header: 0.02 (POST)                            |
        |   Uri-Path: "/.well-known/edhoc"                 |
        |   Payload: true, EDHOC message_1                 |
        |                                                  |
        | <---------------- EDHOC Response---------------- |
        |              Header: 2.04 (Changed)              |
        |              Content-Format: application/edhoc   |
        |              Payload: EDHOC message_2            |
        |                                                  |
EDHOC verification                                         |
        |                                                  |
        | ----------------- EDHOC Request ---------------> |
        |   Header: 0.02 (POST)                            |
        |   Uri-Path: "/.well-known/edhoc"                 |
        |   Payload: C_R, EDHOC message_3                  |
        |                                                  |
        |                                         EDHOC verification
        |                                                  +
OSCORE Sec Ctx                                      OSCORE Sec Ctx
  Derivation                                          Derivation
        |                                                  |
        | ---------------- OSCORE Request ---------------> |
        |   Header: 0.02 (POST)                            |
        |                                                  |
        | <--------------- OSCORE Response --------------- |
        |                         Header: 2.04 (Changed)   |
        |                                                  |
~~~~~~~~~~~~~~~~~
{: #fig-non-combined title="EDHOC and OSCORE run sequentially" artwork-align="center"}

As shown in {{fig-non-combined}}, this purely-sequential flow where EDHOC is run first and then OSCORE is used takes three round trips to complete.

{{edhoc-in-oscore}} defines an optimization for combining EDHOC with the first subsequent OSCORE transaction. This reduces the number of round trips required to set up an OSCORE Security Context and to complete an OSCORE transaction using that Security Context.

# EDHOC Combined with OSCORE {#edhoc-in-oscore}

This section defines an optimization for combining the EDHOC exchange with the first subsequent OSCORE transaction, thus minimizing the number of round trips between the two peers.

This approach can be used only if the default EDHOC message flow is used, i.e., when the Client acts as Initiator and the Server acts as Responder, while it cannot be used in the case with reversed roles.

When running the purely-sequential flow of {{overview}}, the Client has all the information to derive the OSCORE Security Context already after receiving EDHOC message_2 and before sending EDHOC message_3.

Hence, the Client can potentially send both EDHOC message_3 and the subsequent OSCORE Request at the same time. On a semantic level, this requires sending two REST requests at once, as in {{fig-combined}}.

~~~~~~~~~~~~~~~~~
   CoAP Client                                       CoAP Server
(EDHOC Initiator)                                 (EDHOC Responder)
        |                                                  |
        | ----------------- EDHOC Request ---------------> |
        |   Header: 0.02 (POST)                            |
        |   Uri-Path: "/.well-known/edhoc"                 |
        |   Payload: true, EDHOC message_1                 |
        |                                                  |
        | <---------------- EDHOC Response---------------- |
        |              Header: Changed (2.04)              |
        |              Content-Format: application/edhoc   |
        |              Payload: EDHOC message_2            |
        |                                                  |
EDHOC verification                                         |
        +                                                  |
  OSCORE Sec Ctx                                           |
    Derivation                                             |
        |                                                  |
        | ------------ EDHOC + OSCORE Request -----------> |
        |   Header: 0.02 (POST)                            |
        |                                                  |
        |                                         EDHOC verification
        |                                                  +
        |                                          OSCORE Sec Ctx
        |                                             Derivation
        |                                                  |
        | <-------------- OSCORE Response ---------------- |
        |                         Header: 2.04 (Changed)   |
        |                                                  |
~~~~~~~~~~~~~~~~~
{: #fig-combined title="EDHOC and OSCORE combined" artwork-align="center"}

To this end, the specific approach defined in this section consists of sending a single EDHOC + OSCORE request, which conveys the pair (C_R, EDHOC message_3) within an OSCORE protected CoAP message.

That is, the EDHOC + OSCORE request is in practice the OSCORE Request from {{fig-non-combined}}, as still sent to a protected resource and with the correct CoAP method and options intended for accessing that resource. At the same time, the EDHOC + OSCORE request also transports the pair (C_R, EDHOC message_3) required for completing the EDHOC exchange.

As EDHOC message_3 may be too large to be included in a CoAP Option, e.g., if containing a large public key certificate chain, it has to be transported in the CoAP payload of the EDHOC + OSCORE request.

The rest of this section specifies how to transport the data in the EDHOC + OSCORE request and their processing order. In particular, the use of this approach is explicitly signalled by including an EDHOC Option (see {{edhoc-option}}) in the EDHOC + OSCORE request. The processing of the EDHOC + OSCORE request is specified in {{client-processing}} for the Client side and in {{server-processing}} for the Server side.

## EDHOC Option {#edhoc-option}

This section defines the EDHOC Option. The option is used in a CoAP request, to signal that the request payload conveys both an EDHOC message_3 and OSCORE protected data, combined together.

The EDHOC Option has the properties summarized in {{fig-edhoc-option}}, which extends Table 4 of {{RFC7252}}. The option is Critical, Safe-to-Forward, and part of the Cache-Key. The option MUST occur at most once and is always empty. If any value is sent, the value is simply ignored. The option is intended only for CoAP requests and is of Class U for OSCORE {{RFC8613}}.

~~~~~~~~~~~
+-------+---+---+---+---+-------+--------+--------+---------+
| No.   | C | U | N | R | Name  | Format | Length | Default |
+-------+---+---+---+---+-------+--------+--------+---------+
| TBD21 | x |   |   |   | EDHOC | Empty  |   0    | (none)  |
+-------+---+---+---+---+-------+--------+--------+---------+
       C=Critical, U=Unsafe, N=NoCacheKey, R=Repeatable
~~~~~~~~~~~
{: #fig-edhoc-option title="The EDHOC Option." artwork-align="center"}

The presence of this option means that the message payload contains also EDHOC data, that must be extracted and processed as defined in {{server-processing}}, before the rest of the message can be processed.

{{fig-edhoc-opt}} shows the format of a CoAP message containing both the EDHOC data and the OSCORE ciphertext, using the newly defined EDHOC option for signalling.

~~~~~~~~~~~~~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Ver| T |  TKL  |      Code     |          Message ID           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Token (if any, TKL bytes) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  OSCORE option  |   EDHOC option  | Other options (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1 1 1 1 1 1 1 1|    Payload
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~
{: #fig-edhoc-opt title="CoAP message for EDHOC and OSCORE combined - signalled with the EDHOC Option" artwork-align="center"}

## Client Processing {#client-processing}

The Client prepares an EDHOC + OSCORE request as follows.

1. Compose EDHOC message_3 as per {{Section 5.4.2 of I-D.ietf-lake-edhoc}}.

2. Encrypt the original CoAP request as per {{Section 8.1 of RFC8613}}, using the new OSCORE Security Context established after receiving EDHOC message_2.

   Note that the OSCORE ciphertext is not computed over EDHOC message_3, which is not protected by OSCORE. That is, the result of this step is the OSCORE Request as in {{fig-non-combined}}.

3. Build a CBOR sequence {{RFC8742}} composed of two CBOR byte strings in the following order.

   * The first CBOR byte string is the EDHOC message_3 resulting from step 1.

   * The second CBOR byte string has as value the OSCORE ciphertext of the OSCORE protected CoAP request resulting from step 2.

4. Compose the EDHOC + OSCORE request, as the OSCORE protected CoAP request resulting from step 2, where the payload is replaced with the CBOR sequence built at step 3.

   Note that the new payload includes EDHOC message_3, but it does not include the EDHOC connection identifier C_R. As the Client is the EDHOC Initiator, C_R encodes the OSCORE Sender ID of the Client, which is already specified as 'kid' in the OSCORE Option of the request from step 2, hence of the EDHOC + OSCORE request.

5. Signal the usage of this approach, by including the new EDHOC Option defined in {{edhoc-option}} into the EDHOC + OSCORE request.

## Server Processing {#server-processing}

When receiving a request containing the EDHOC option, i.e., an EDHOC + OSCORE request, the Server MUST perform the following steps.

1. Check that the payload of the EDHOC + OSCORE request is a CBOR sequence composed of two CBOR byte strings. If this is not the case, the Server MUST stop processing the request and MUST reply with a 4.00 (Bad Request) error response.

2. Extract EDHOC message_3 from the payload of the EDHOC + OSCORE request, as the first CBOR byte string in the CBOR sequence.

3. Take the value of 'kid' from the OSCORE option of the EDHOC + OSCORE request (i.e., the OSCORE Sender ID of the Client), and use it to rebuild the EDHOC connection identifier C_R, as per {{oscore-to-edhoc-id}}.

4. Retrieve the correct EDHOC session by using the connection identifier C_R rebuilt at step 3.

   If the applicability statement used in the EDHOC session specifies that EDHOC message_4 shall be sent, the Server MUST stop the EDHOC processing and consider it failed, as due to a client error.

   Otherwise, perform the EDHOC processing on the EDHOC message_3 extracted at step 2 as per {{Section 5.4.3 of I-D.ietf-lake-edhoc}}, based on the protocol state of the retrieved EDHOC session.

   The applicability statement used in the EDHOC session is the same one associated to the EDHOC resource where the server received the request conveying EDHOC message_1 that started the session. This is relevant in case the server provides multiple EDHOC resources, which may generally refer to different applicability statements.

5. Establish a new OSCORE Security Context associated to the client as per {{Section A.2 of I-D.ietf-lake-edhoc}}, using the EDHOC output from step 4.

6. Extract the OSCORE ciphertext from the payload of the EDHOC + OSCORE request, as the value of the second CBOR byte string in the CBOR sequence.

7. Rebuild the OSCORE protected CoAP request as the EDHOC + OSCORE request, where the payload is replaced with the OSCORE ciphertext extracted at step 6.

8. Decrypt and verify the OSCORE protected CoAP request rebuilt at step 7, as per {{Section 8.2 of RFC8613}}, by using the OSCORE Security Context established at step 5.

9. Deliver the CoAP request resulting from step 8 to the application.

If steps 4 (EDHOC processing) and 8 (OSCORE processing) are both successfully completed, the Server MUST reply with an OSCORE protected response, in order for the Client to achieve key confirmation (see {{Section 5.4.2 of I-D.ietf-lake-edhoc}}). The usage of EDHOC message_4 as defined in {{Section 5.5 of I-D.ietf-lake-edhoc}} is not applicable to the approach defined in this document.

If step 4 (EDHOC processing) fails, the server discontinues the protocol as per {{Section 5.4.3 of I-D.ietf-lake-edhoc}} and responds with an EDHOC error message with error code 1, formatted as defined in {{Section 6.2 of I-D.ietf-lake-edhoc}}. In particular, the CoAP response conveying the EDHOC error message MUST have Content-Format set to application/edhoc defined in {{Section 9.12 of I-D.ietf-lake-edhoc}}.

If step 4 (EDHOC processing) is successfully completed but step 8 (OSCORE processing) fails, the same OSCORE error handling as defined in {{Section 8.2 of RFC8613}} applies.

## Example of EDHOC + OSCORE Request # {#example}

{{fig-edhoc-opt-2}} shows an example of EDHOC + OSCORE Request. In particular, the example assumes that:

* The used OSCORE Partial IV is 0, consistently with the first request protected with the new OSCORE Security Context.

* The OSCORE Sender ID of the Client is 0x01.

   As per {{oscore-to-edhoc-id}}, this corresponds to the numeric EDHOC connection identifier C_R with value 1. When using the purely-sequential flow shown in {{fig-non-combined}}, this would be prepended to EDHOC message_3 as the CBOR integer 1 (0x01 in CBOR encoding), in the paylod of the second EDHOC request.

* The EDHOC option is registered with CoAP option number 21.

~~~~~~~~~~~~~~~~~
o  OSCORE option value: 0x090001 (3 bytes)

o  EDHOC option value: - (0 bytes)

o  EDHOC message_3: 0x52d5535f3147e85f1cfacd9e78abf9e0a81bbf (19 bytes)

o  OSCORE ciphertext: 0x612f1092f1776f1c1668b3825e (13 bytes)

From there:

o  Protected CoAP request (OSCORE message):

   0x44025d1f               ; CoAP 4-byte header
     00003974               ; Token
     39 6c6f63616c686f7374  ; Uri-Host Option: "localhost"
     63 090001              ; OSCORE Option
     c0                     ; EDHOC Option
     ff 52d5535f3147e85f1cfacd9e78abf9e0a81bbf
        4d612f1092f1776f1c1668b3825e
   (57 bytes)
~~~~~~~~~~~~~~~~~
{: #fig-edhoc-opt-2 title="Example of CoAP message with EDHOC and OSCORE combined" artwork-align="center"}

# Conversion from OSCORE to EDHOC Identifiers # {#conversion}

{{Section A.1 of I-D.ietf-lake-edhoc}} defines how an EDHOC connection identifier is converted to an OSCORE Sender/Recipient ID.

In the following, {{oscore-to-edhoc-id}} defines a method for converting from OSCORE Sender/Recipient ID to EDHOC connection identifier.

When running EDHOC through a certain EDHOC resource, the Client and Server MUST both use the conversion method defined in {{oscore-to-edhoc-id}} and MUST perform the additional message processing specified in {{oscore-edhoc-message-processing}}, if at least one of the following conditions hold.

* The applicability statement associated to the EDHOC resource indicates that the server supports the EDHOC + OSCORE request defined in {{edhoc-in-oscore}}.

* The applicability statement associated to the EDHOC resource indicates that the conversion method defined in {{oscore-to-edhoc-id}} is the one to use.

Instead, if none of the above conditions hold, the Client and the Server can independently use any consistent conversion method, such as the one defined in {{oscore-to-edhoc-id}} or different ones defined in separate specifications. In particular, the Client and Server are not required to use the same conversion method. In fact, as per {{Section A.1 of I-D.ietf-lake-edhoc}}, it is sufficient that the two connection identifiers C_I and C_R exchanged during an EDHOC execution are different and not "equivalent", hence not convertible to the same OSCORE Sender/Recipient ID.

Even in case none the above conditions hold, it is RECOMMENDED for the Client and Server to use the conversion method defined in {{oscore-to-edhoc-id}}, since it ensures that an OSCORE Sender/Recipient ID is always converted to the EDHOC identifier with the smallest size among the two equivalent ones.

## Conversion Method {#oscore-to-edhoc-id}

The process defined in this section ensures that every OSCORE Sender/Recipient ID is converted to only one of the two corresponding, equivalent EDHOC connection identifiers, see {{Section A.1 of I-D.ietf-lake-edhoc}}.

An OSCORE Sender/Recipient ID, OSCORE_ID, is converted to an EDHOC connection identifier, EDHOC_ID, as follows.

* If OSCORE_ID is 0 bytes in size, it is converted to the empty byte string EDHOC_ID (0x40 in CBOR encoding).

* If OSCORE_ID has a size in bytes different than 0, 1, 2, 3, 5 or 9, it is converted to a byte-valued EDHOC_ID, i.e., a CBOR byte string with value OSCORE_ID.

   For example, the OSCORE_ID 0x001122334455 is converted to the byte-valued EDHOC_ID 0x001122334455 (0x46001122334455 in CBOR encoding).

* If OSCORE_ID has a size in bytes equal to 1, 2, 3, 5 or 9 the following applies.

   - If OSCORE_ID is a valid CBOR encoding for an integer value (i.e., it can be correctly decoded as a CBOR integer), then it is converted to a numeric EDHOC_ID having OSCORE_ID as its CBOR encoded form.

      For example, the OSCORE_ID 0x01 is converted to the numeric EDHOC_ID 1 (0x01 in CBOR encoding), while the OSCORE_ID 0x2B is converted to the numeric EDHOC_ID -12 (0x2B in CBOR encoding).

   - If OSCORE_ID is _not_ a valid CBOR encoding for an integer value (i.e., it _cannot_ be correctly decoded as a CBOR integer), then it is converted to a byte-valued EDHOC_ID having OSCORE_ID as its value.

       For example, the OSCORE_ID 0xFF is converted to the byte-valued EDHOC_ID 0xFF (0x41FF in CBOR encoding).

   Implementations can easily determine which case holds for a given OSCORE_ID with no need to try to actually CBOR-decode it, e.g., by using the approach in {{sec-cbor-numeric-check}}.

When performing the conversions above, the two peers MUST always refer to Deterministically Encoded CBOR as specified in {{Section 4.2.1 of RFC8949}}, consistently with what is required by the EDHOC protocol {{I-D.ietf-lake-edhoc}}.

## Additional Processing of EDHOC Messages {#oscore-edhoc-message-processing}

This section specifies additional EDHOC message processing compared to what is specified in {{Section 5 of I-D.ietf-lake-edhoc}}.

### Initiator Processing of Message 1

The Initiator selects C_I as follows.

1. The Initiator initializes a set ID_SET as the empty set.

2. The Initiator selects an available OSCORE Recipient ID, ID*, which is not included in ID_SET.

3. The Initiator converts ID* to the EDHOC connection identifier C_I, as per {{oscore-to-edhoc-id}}.

4. If the resulting C_I is already used, the Initiator adds ID* to ID_SET and moves back to step 2. Otherwise, it uses C_I as its EDHOC connection identifier.

### Responder Processing of Message 1

The Responder MUST discontinue the protocol and reply with an EDHOC error message with error code 1, formatted as defined in {{Section 6.2 of I-D.ietf-lake-edhoc}}, if C_I is a CBOR byte string and it has as value a valid CBOR encoding of an integer value (e.g., C_I is CBOR encoded as 0x4100).

In fact, this would mean that the Initiator has not followed the conversion rule in {{oscore-to-edhoc-id}} when converting its (to be) OSCORE Recipient ID to C_I.

### Responder Processing of Message 2

The Responder selects C_R as follows.

1. The Responder initializes a set ID_SET as the empty set.

2. The Responder selects an available OSCORE Recipient ID, ID*, which is not included in ID_SET.

3. The Responder converts ID* to the EDHOC connection identifier C_R, as per {{oscore-to-edhoc-id}}.

4. If the resulting C_R is already used or it is equal byte-by-byte to the C_I specified in EDHOC message_1, the Initiator adds ID* to ID_SET and moves back to step 2. Otherwise, it uses C_R as its EDHOC connection identifier.

### Initiator Processing of Message 2

If any of the following conditions holds, the Initiator MUST discontinue the protocol and reply with an EDHOC error message with error code 1, formatted as defined in {{Section 6.2 of I-D.ietf-lake-edhoc}}.

* C_R is equal byte-by-byte to the C_I that was specified in EDHOC message_1.

* C_R is a CBOR byte string and it has as value a valid CBOR encoding of an integer value (e.g., C_R is CBOR encoded as 0x4100).

   In fact, this would mean that the Responder has not followed the conversion rule in {{oscore-to-edhoc-id}} when converting its (to be) OSCORE Recipient ID to C_R.

# Security Considerations

The same security considerations from OSCORE {{RFC8613}} and EDHOC {{I-D.ietf-lake-edhoc}} hold for this document.

TODO (more considerations)

# IANA Considerations

RFC Editor: Please replace "\[\[this document\]\]" with the RFC number of this document and delete this paragraph.

This document has the following actions for IANA.

## CoAP Option Numbers Registry ## {#iana-coap-options}

IANA is asked to enter the following option number to the "CoAP Option Numbers" registry within the "CoRE Parameters" registry group.

\[

The CoAP option numbers 13 and 21 are both consistent with the properties of the EDHOC Option defined in {{edhoc-option}}, and they both allow the EDHOC Option to always result in an overall size of 1 byte. This is because:

* The EDHOC option is always empty, i.e., with zero-length value; and

* Since the OSCORE option with option number 9 is always present in the CoAP request, the EDHOC option would be encoded with a maximum delta of 4 or 12, depending on its option number being 13 or 21.

At the time of writing, the CoAP option numbers 13 and 21 are both unassigned in the "CoAP Option Numbers" registry, as first available and consistent option numbers for the EDHOC option.

This document suggests 21 (TBD21) as option number to be assigned to the new EDHOC option, since both 13 and 21 are consistent for the use case in question, but different use cases or protocols may make better use of the option number 13.

\]

~~~~~~~~~~~
+--------+-------+-------------------+
| Number | Name  | Reference         |
+--------+-------+-------------------+
| TBD21  | EDHOC | [[this document]] |
+--------+-------+-------------------+
~~~~~~~~~~~
{: artwork-align="center"}


--- back

# Checking CBOR Encoding of Numeric Values # {#sec-cbor-numeric-check}

A binary string of N bytes in size is a valid CBOR encoding of an integer value if and only if both the following conditions hold, with reference to the table below.

* The size N is one of the values specified in the "Size" column.

* The first byte of the binary string is equal to one of the byte values specified in the "First byte" column, exactly for the row having N as value of the "Size" column.

~~~~~~~~~~~~~~~~~
+------+-----------------------+
| Size | First byte            |
+------+-----------------------+
| 1    | 0x00-0x17 , 0x20-0x37 |
+------+-----------------------+
| 2    | 0x18 , 0x38           |
+------+-----------------------+
| 3    | 0x19 , 0x39           |
+------+-----------------------+
| 5    | 0x1A , 0x3A           |
+------+-----------------------+
| 9    | 0x1B , 0x3B           |
+------+-----------------------+
~~~~~~~~~~~~~~~~~
{: artwork-align="center"}

# Document Updates # {#sec-document-updates}

RFC Editor: Please remove this section.

## Version -01 to -02 ## {#sec-01-02}

* New title.

* Restructured table of content.

* Alignment with latest format of EDHOC messages.

* Guideline on ID conversions based on applicability statement.

* Clarifications on applicability statements.

* RFC8126 terminology in IANA considerations.

* Revised Appendix "Checking CBOR Encoding of Numeric Values".

## Version -00 to -01 ## {#sec-00-01}

* Improved background overview of EDHOC.

* Added explicit rules for converting OSCORE Sender/Recipient IDs to EDHOC connection identifiers following the removal of bstr_identifier from EDHOC.

* Revised section organization.

* Recommended number for EDHOC option changed to 21.

* Editorial improvements.

# Acknowledgments
{:numbered="false"}

The authors sincerely thank Christian Amsuess, Klaus Hartke, Jim Schaad and Malisa Vucinic for their feedback and comments.

The work on this document has been partly supported by VINNOVA and the Celtic-Next project CRITISEC; and by the H2020 project SIFIS-Home (Grant agreement 952652).
