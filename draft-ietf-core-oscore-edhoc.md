---
title: "Using EDHOC for OSCORE with CoAP Transport"
abbrev: "EDHOC for OSCORE with CoAP Transport"
docname: draft-ietf-core-oscore-edhoc-latest
cat: std

ipr: trust200902
area: Internet
workgroup: CoRE Working Group
keyword: Internet-Draft

stand_alone: yes
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

This document defines how the lightweight authenticated key exchange protocol EDHOC is used for establishing an OSCORE Security Context, using CoAP for message transferring. Furthermore, this document defines an optimization approach for combining EDHOC run over CoAP with the first subsequent OSCORE transaction. This reduces the number of round trips required to set up an OSCORE Security Context and to complete an OSCORE transaction using that Security Context.

--- middle

# Introduction

Ephemeral Diffie-Hellman Over COSE (EDHOC) {{I-D.ietf-lake-edhoc}} is a lightweight authenticated key exchange protocol, especially intended for use in constrained scenarios. As defined in Section 7.2 of {{I-D.ietf-lake-edhoc}}, EDHOC messages can be transported over the Constrained Application Protocol (CoAP) {{RFC7252}}.

This document builds on the EDHOC specification {{I-D.ietf-lake-edhoc}} and defines how EDHOC run over CoAP is used for establishing a Security Context to use for communicating with Object Security for Constrained RESTful Environment (OSCORE) {{RFC8613}}.

In addition, this document defines an optimization approach that combines EDHOC run over CoAP with the first subsequent OSCORE transaction. This allows for a minimum number of round trips necessary to setup the OSCORE Security Context and complete an OSCORE transaction, for example when an IoT device gets configured in a network for the first time.

This optimization is desirable, since the number of protocol round trips impacts the minimum number of flights, which in turn can have a substantial impact on the latency of conveying the first OSCORE request, when using certain radio technologies.

Without this optimization, it is not possible, not even in theory, to achieve the minimum number of flights. This optimization makes it possible also in practice, since the last message of the EDHOC protocol can be made relatively small (see Section 1 of {{I-D.ietf-lake-edhoc}}), thus allowing additional OSCORE protected CoAP data within target MTU sizes.

## Terminology

{::boilerplate bcp14}

The reader is expected to be familiar with terms and concepts defined in CoAP {{RFC7252}}, CBOR {{RFC8949}}, CBOR sequences {{RFC8742}}, OSCORE {{RFC8613}} and EDHOC {{I-D.ietf-lake-edhoc}}.

# EDHOC Overview {#overview}

The EDHOC protocol allows two peers to agree on a cryptographic secret, in a mutually-authenticated way and by using Diffie-Hellman ephemeral keys to achieve perfect forward secrecy. The two peers are denoted as Initiator and Responder, as the one sending or receiving the initial EDHOC message_1, respectively.

After successful processing of EDHOC message_3, both peers agree on a cryptographic secret that can be used to derive further security material, and especially to establish an OSCORE Security Context {{RFC8613}}. The Responder can also send an optional EDHOC message_4 to achieve key confirmation, e.g. in deployments where no protected application message is sent from the Responder to the Initiator.

Section 7.2 of {{I-D.ietf-lake-edhoc}} specifies how to transport EDHOC over CoAP. That is, the EDHOC data (referred to as "EDHOC messages") are transported in the payload of CoAP requests and responses. The default message flow consists in the CoAP Client acting as Initiator and the CoAP Server acting as Responder. Alternatively, the two roles can be reversed. In the rest of this document, EDHOC messages are considered to be transported over CoAP.

{{fig-non-combined}} shows a Client and Server running EDHOC as Initiator and Responder, respectively. That is, the Client sends a POST request with payload EDHOC message_1 to a reserved resource at the CoAP Server, by default at Uri-Path "/.well-known/edhoc". This triggers the EDHOC exchange on the Server, which replies with a 2.04 (Changed) Response with payload EDHOC message_2. Finally, the Client sends a CoAP POST request to the same resource used for EDHOC message_1, with payload EDHOC message_3. The Content-Format of these CoAP messages may be set to "application/edhoc".

After this exchange takes place, and after successful verifications as specified in the EDHOC protocol, the Client and Server can derive an OSCORE Security Context, as defined in {{oscore-ctx}} of this document. After that, they can use OSCORE to protect their communications.

~~~~~~~~~~~~~~~~~
   CoAP Client                                  CoAP Server
        | ------------- EDHOC message_1 ------------> |
        |          Header: POST (Code=0.02)           |
        |       Uri-Path: "/.well-known/edhoc"        |
        |      Content-Format: application/edhoc      |
        |                                             |
        | <------------ EDHOC message_2 ------------- |
        |            Header: 2.04 Changed             |
        |      Content-Format: application/edhoc      |
        |                                             |
EDHOC verification                                    |
        |                                             |
        | ------------- EDHOC message_3 ------------> |
        |          Header: POST (Code=0.02)           |
        |        Uri-Path: "/.well-known/edhoc"       |
        |      Content-Format: application/edhoc      |
        |                                             |
        |                                    EDHOC verification
        |                                             +
OSCORE Sec Ctx                                OSCORE Sec Ctx
  Derivation                                     Derivation
        |                                             |
        | ------------- OSCORE Request -------------> |
        |          Header: POST (Code=0.02)           |
        |                                             |
        | <------------ OSCORE Response ------------- |
        |            Header: 2.04 Changed             |
        |                                             |
~~~~~~~~~~~~~~~~~
{: #fig-non-combined title="EDHOC and OSCORE run sequentially" artwork-align="center"}

As shown in {{fig-non-combined}}, this purely-sequential way of first running EDHOC and then using OSCORE takes three round trips to complete.

{{edhoc-in-oscore}} defines an optimization for combining EDHOC with the first subsequent OSCORE transaction. This reduces the number of round trips required to set up an OSCORE Security Context and to complete an OSCORE transaction using that Security Context.

# Transferring EDHOC in CoAP {#edhoc-in-coap}

When using EDHOC over CoAP for establishing an OSCORE Security Context, EDHOC messages are exchanged as defined in Section 7.2 of {{I-D.ietf-lake-edhoc}}, with the following addition.

EDHOC error messages sent as CoAP responses MUST be error responses, i.e. they MUST specify a CoAP error response code. In particular, it is RECOMMENDED that such error responses have response code either 4.00 (Bad Request) in case of client error (e.g. due to a malformed EDHOC message), or 5.00 (Internal Server Error) in case of server error (e.g. due to failure in deriving EDHOC key material).

# Deriving an OSCORE Security Context from EDHOC {#oscore-ctx}

After successful processing of EDHOC message_3 (see Section 5.5 of {{I-D.ietf-lake-edhoc}}), the Client and Server derive Security Context parameters for OSCORE (see Section 3.2 of {{RFC8613}}) as follows.

* The Master Secret and Master Salt are derived by using the EDHOC-Exporter interface defined in Section 4.1 of {{I-D.ietf-lake-edhoc}}.

  The EDHOC Exporter Labels to use are "OSCORE Master Secret" and "OSCORE Master Salt". By default, key_length is the key length (in bytes) of the application AEAD Algorithm and salt_length is 8 bytes. The Initiator and Responder MAY agree out-of-band on a longer key_length than the default and a different salt_length.

~~~~~~~~~~~~~~~~~~~~~~~
   Master Secret = EDHOC-Exporter( "OSCORE Master Secret", key_length )
   Master Salt   = EDHOC-Exporter( "OSCORE Master Salt", salt_length )
~~~~~~~~~~~~~~~~~~~~~~~

* The AEAD Algorithm is the application AEAD of the selected cipher suite.

* The HKDF Algorithm is the HKDF using as hash algorithm the application hash algorithm of the selected cipher suite.

* In case the Client is the Initiator and the Server is the Responder, the Client's OSCORE Sender ID is the EDHOC connection identifier C_R, while the Server's OSCORE Sender ID is EDHOC connection identifier C_I. The reverse applies in case the Client is the Responder and the Server is the Initiator.

   The two peers MUST ensure that the EDHOC connection identifiers are unique, i.e. C_R MUST NOT be equal to C_I. In particular, the Responder MUST NOT include in EDHOC message_2 a connection identifier C_R equal to the connection identifier C_I received in EDHOC message_1. The initiator MUST discontinue the protocol and reply with an EDHOC error message when receiving an EDHOC message_2 that includes a connection identifier C_R equal to C_I.

The Client and Server use the parameters above to establish an OSCORE Security Context, as per Section 3.2.1 of {{RFC8613}}.

From then on, the Client and Server MUST be able to retrieve the OSCORE protocol state using its chosen connection identifier, and optionally other information such as the 5-tuple.

# EDHOC Combined with OSCORE {#edhoc-in-oscore}

This section defines an optimization for combining the EDHOC exchange with the first subsequent OSCORE transaction, thus minimizing the number of round trips between the two peers. This approach can be used only if the default EDHOC message flow is used, i.e. when the Client acts as Initiator and the Server acts as Responder.

When running the purely-sequential flow in {{overview}}, the Client has all the information to derive the OSCORE Security Context already after receiving EDHOC message_2 and before sending EDHOC message_3.

Hence, the Client can potentially send both EDHOC message_3 and the subsequent OSCORE Request at the same time. On a semantic level, this requires to send two REST requests at once, as in {{fig-combined}}.

~~~~~~~~~~~~~~~~~
   CoAP Client                                  CoAP Server
        | ------------- EDHOC message_1 ------------> |
        |          Header: POST (Code=0.02)           |
        |       Uri-Path: "/.well-known/edhoc"        |
        |      Content-Format: application/edhoc      |
        |                                             |
        | <------------ EDHOC message_2 ------------- |
        |            Header: 2.04 Changed             |
        |      Content-Format: application/edhoc      |
        |                                             |
EDHOC verification                                    |
        +                                             |
  OSCORE Sec Ctx                                      |
    Derivation                                        |
        |                                             |
        | ---- EDHOC message_3 + OSCORE Request ----> |
        |          Header: POST (Code=0.02)           |
        |                                             |
        |                                    EDHOC verification
        |                                             +
        |                                     OSCORE Sec Ctx
        |                                        Derivation
        |                                             |
        | <------------ OSCORE Response ------------- |
        |            Header: 2.04 Changed             |
        |                                             |
~~~~~~~~~~~~~~~~~
{: #fig-combined title="EDHOC and OSCORE combined" artwork-align="center"}

The specific approach defined in this section consists of sending EDHOC message_3 inside an OSCORE protected CoAP message.

The resulting EDHOC + OSCORE request is in practice the OSCORE Request from {{fig-non-combined}}, sent to a protected resource and with the correct CoAP method and options, with the addition that it also transports EDHOC message_3.

As EDHOC message_3 may be too large to be included in a CoAP Option, e.g. if containing a large public key certificate chain, it has to be transported in the CoAP payload of the EDHOC + OSCORE request.

The rest of this section specifies how to transport the data in the EDHOC + OSCORE request and their processing order. In particular, the use of this approach is explicitly signalled by including an EDHOC Option (see {{edhoc-option}}) in the EDHOC + OSCORE request. The processing of the EDHOC + OSCORE request is specified in {{client-processing}} for the Client side and in {{server-processing}} for the Server side.

## EDHOC Option {#edhoc-option}

This section defines the EDHOC Option, which is used in a CoAP request to signal that the request combines an EDHOC message_3 and OSCORE protected data.

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
|  OSCORE option  |   EDHOC option  | other options (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1 1 1 1 1 1 1 1|    Payload
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~~~~~~~~~~~~~
{: #fig-edhoc-opt title="CoAP message for EDHOC and OSCORE combined - signalled with the EDHOC Option" artwork-align="center"}

## Client Processing {#client-processing}

The Client prepares an EDHOC + OSCORE request as follows.

1. Compose EDHOC message_3 as per Section 5.4.2 of {{I-D.ietf-lake-edhoc}}.

   Since the Client is the EDHOC Initiator and the used Correlation Method is 1 (see Section 3.2.4 of {{I-D.ietf-lake-edhoc}}), the EDHOC message_3 always includes the Connection Identifier C_R and CIPHERTEXT_3. Note that C_R is the OSCORE Sender ID of the Client, encoded as a bstr_identifier (see Section 5.1 of {{I-D.ietf-lake-edhoc}}).

2. Encrypt the original CoAP request as per Section 8.1 of {{RFC8613}}, using the new OSCORE Security Context established after receiving EDHOC message_2.

   Note that the OSCORE ciphertext is not computed over EDHOC message_3, which is not protected by OSCORE. That is, the result of this step is the OSCORE Request as in {{fig-non-combined}}.

3. Build a CBOR sequence {{RFC8742}} composed of two CBOR byte strings in the following order.

   * The first CBOR byte string is the CIPHERTEXT_3 of the EDHOC message_3 resulting from step 3.

   * The second CBOR byte string has as value the OSCORE ciphertext of the OSCORE protected CoAP request resulting from step 2.

4. Compose the EDHOC + OSCORE request, as the OSCORE protected CoAP request resulting from step 2, where the payload is replaced with the CBOR sequence built at step 3.

5. Signal the usage of this approach within the EDHOC + OSCORE request, by including the new EDHOC Option defined in {{edhoc-option}}.

## Server Processing {#server-processing}

When receiving a request containing the EDHOC option, i.e. an EDHOC + OSCORE request, the Server MUST perform the following steps.

1. Check that the payload of the EDHOC + OSCORE request is a CBOR sequence composed of two CBOR byte strings. If this is not the case, the Server MUST stop processing the request and MUST respond with a 4.00 (Bad Request) error message.

2. Extract CIPHERTEXT_3 from the payload of the EDHOC + OSCORE request, as the first CBOR byte string in the CBOR sequence.

3. Rebuild EDHOC message_3, as a CBOR sequence composed of two CBOR byte strings in the following order.

   * The first CBOR byte string is the 'kid' of the Client indicated in the OSCORE option of the EDHOC + OSCORE request, encoded as a bstr_identifier (see Section 5.1 of {{I-D.ietf-lake-edhoc}}).

   * The second CBOR byte string is the CIPHERTEXT_3 retrieved at step 2.

4. Perform the EDHOC processing on the EDHOC message_3 rebuilt at step 3, including verifications, and the OSCORE Security Context derivation, as per Section 5.4.3 and Section 7.2.1 of {{I-D.ietf-lake-edhoc}}, respectively.

   If the applicability statement used in the EDHOC session specifies that EDHOC message_4 shall be sent, the Server MUST stop the EDHOC processing and consider it failed, as due to a client error.

5. Extract the OSCORE ciphertext from the payload of the EDHOC + OSCORE request, as the value of the second CBOR byte string in the CBOR sequence.

6. Rebuild the OSCORE protected CoAP request as the EDHOC + OSCORE request, where the payload is replaced with the OSCORE ciphertext resulting from step 5.

7. Decrypt and verify the OSCORE protected CoAP request resulting from step 6, as per Section 8.2 of {{RFC8613}}, by using the new OSCORE Security Context established at step 4.

8. Process the CoAP request resulting from step 7.

If steps 4 (EDHOC processing) and 7 (OSCORE processing) are both successfully completed, the Server MUST reply with an OSCORE protected response, in order for the Client to achieve key confirmation (see Section 5.4.2 of {{I-D.ietf-lake-edhoc}}). The usage of EDHOC message_4 as defined in Section 7.1 of {{I-D.ietf-lake-edhoc}} is not applicable to the approach defined in this specification.

If step 4 (EDHOC processing) fails, the server discontinues the protocol as per Section 5.4.3 of {{I-D.ietf-lake-edhoc}} and responds with an EDHOC error message, formatted as defined in Section 6.1 of {{I-D.ietf-lake-edhoc}}. In particular, the CoAP response conveying the EDHOC error message MUST have Content-Format set to application/edhoc defined in Section 9.5 of {{I-D.ietf-lake-edhoc}}.

If step 4 (EDHOC processing) is successfully completed but step 7 (OSCORE processing) fails, the same OSCORE error handling applies as defined in Section 8.2 of {{RFC8613}}.

## Example of EDHOC + OSCORE Request # {#example}

An example based on the OSCORE test vector from Appendix C.4 of {{RFC8613}} and the EDHOC test vector from Appendix B.2 of {{I-D.ietf-lake-edhoc}} is given in {{fig-edhoc-opt-2}}. In particular, the example assumes that:

* The used OSCORE Partial IV is 0, consistently with the first request protected with the new OSCORE Security Context.

* The OSCORE Sender ID of the Client is 0x00. This corresponds to the EDHOC Connection Identifier C_R, which is encoded as the bstr_identifier 0x37 in EDHOC message_3.

* The EDHOC option is registered with CoAP option number 21.

~~~~~~~~~~~~~~~~~
   o  OSCORE option value: 0x090020 (3 bytes)

   o  EDHOC option value: - (0 bytes)

   o  C_R: 0x37 (1 byte)

   o  CIPHERTEXT_3: 0x52d5535f3147e85f1cfacd9e78abf9e0a81bbf
      (19 bytes)

   o  EDHOC message_3: 0x37 52d5535f3147e85f1cfacd9e78abf9e0a81bbf
      (20 bytes)

   o  OSCORE ciphertext: 0x612f1092f1776f1c1668b3825e (13 bytes)

   From there:

   o  Protected CoAP request (OSCORE message):

      0x44025d1f               ; CoAP 4-byte header
        00003974               ; Token
        39 6c6f63616c686f7374  ; Uri-Host Option: "localhost"
        63 090020              ; OSCORE Option
        C0                     ; EDHOC Option
        ff 52d5535f3147e85f1cfacd9e78abf9e0a81bbf
           4d612f1092f1776f1c1668b3825e
      (57 bytes)
~~~~~~~~~~~~~~~~~
{: #fig-edhoc-opt-2 title="Example of CoAP message with EDHOC and OSCORE combined" artwork-align="center"}

# Security Considerations

The same security considerations from OSCORE {{RFC8613}} and EDHOC {{I-D.ietf-lake-edhoc}} hold for this document.

TODO (more considerations)

# IANA Considerations

This document has the following actions for IANA.

## CoAP Option Numbers Registry ## {#iana-coap-options}

IANA is asked to enter the following option numbers to the "CoAP Option Numbers" registry defined in {{RFC7252}} within the "CoRE Parameters" registry.

\[

The CoAP option numbers 13 and 21 are both consistent with the properties of the EDHOC Option defined in {{edhoc-option}}, and they both allow the EDHOC Option to always result in an overall size of 1 byte. This is because:

* The EDHOC option is always empty, i.e. with zero-length value; and

* Since the OSCORE option with option number 9 is always present in the CoAP request, the EDHOC option would be encoded with a maximum delta of 4 or 12, depending on its option number being 13 or 21.

At the time of writing, the CoAP option numbers 13 and 21 are both unassigned in the "CoAP Option Numbers" registry, as first available and consistent option numbers for the EDHOC option.

This document suggests 21 (TBD21) as option number to be assigned to the new EDHOC option, since both 13 and 21 are consistent for the use case in question, but different use cases or protocols may make better use of the option number 13.

\]

~~~~~~~~~~~
+--------+-------+-------------------+
| Number | Name  |     Reference     |
+--------+-------+-------------------+
| TBD21  | EDHOC | [[this document]] |
+--------+-------+-------------------+
~~~~~~~~~~~
{: artwork-align="center"}

## EDHOC Exporter Label Registry ## {#iana-edhoc-exporter-label}

IANA is asked to enter the following entries to the "EDHOC Exporter Label" registry defined in {{I-D.ietf-lake-edhoc}}.

~~~~~~~~~~~~~~~~~~~~~~~
Label: OSCORE Master Secret
Description: Derived OSCORE Master Secret
Reference: [[this document]]
~~~~~~~~~~~~~~~~~~~~~~~

~~~~~~~~~~~~~~~~~~~~~~~
Label: OSCORE Master Salt
Description: Derived OSCORE Master Salt
Reference: [[this document]]
~~~~~~~~~~~~~~~~~~~~~~~

--- back

# Document Updates # {#sec-document-updates}

RFC EDITOR: PLEASE REMOVE THIS SECTION.

## Version -00 to -01 ## {#sec-00-01}

* Imported OSCORE-specific content from draft-ietf-lake-edhoc.

* Expanded scope and revised section organization.

* Improved error handling for the combined approach.

* Recommended number for EDHOC option changed to 21.

# Acknowledgments
{:numbered="false"}

The authors sincerely thank Christian Amsuess, Klaus Hartke, Jim Schaad and Malisa Vucinic for their feedback and comments in the discussion leading up to this draft.

The work on this document has been partly supported by VINNOVA and the Celtic-Next project CRITISEC; and by the H2020 project SIFIS-Home (Grant agreement 952652).
