# Community Specification Template 1.0

Community Specifications are recommended to be drafted in accordance with international best practices.  Doing so provides clarity, helps adoption, and also eases the transition of this specification to other standards body if so desired.  Accordingly, the recommended template below is based on ISO standard drafting conventions.

To help you, the ISO/TMP has published a [guide on writing standards][0].

A model manuscript is available of a draft International Standard known as [“The Rice Model”][1]

In addition, we recommend using the key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" as described in RFC 2119 - https://tools.ietf.org/html/rfc2119
 
## Git Cryptography Protocol
### Version 0.0.1
### Status Pre-draft

© 2021 TrustFrame, Inc.

This specification is subject to the [Community Specification License 1.0][2].

## Contents

1. [Foreword](#Foreword)
2. [Introduction](#Introduction)
  1. [Scope](#Scope)
  2. [Normative References](#Normative References)
  3. [Terms and Definitions](#Terms and Definitions)
3. [The Protocol](#The Protocol)
4. [The Server](#The Server)
5. [The Client](#The Client)
6. [Signing a Git Object](#Signing a Git Object)
  1. [The Returned Signature Data](#The Returned Signature Data)
7. [Verifying a Signed Git Object](#Verifying a Signed Git Object)
8. [Conclusion](#Conclusion)
9. [Bibliography](#Bibliography)

## Foreword

Attention is drawn to the possibility that some of the elements of this document may be the subject of patent rights. No party shall not be held responsible for identifying any or all such patent rights.

Any trade name used in this document is information given for the convenience of users and does not constitute an endorsement.

This document was prepared by TrustFrame, Inc.

Known patent licensing exclusions are available in the specification’s repository’s Notices.md file.

Any feedback or questions on this document should be directed to the [specification repository][3].

THESE MATERIALS ARE PROVIDED “AS IS.” The Contributors and Licensees expressly disclaim any warranties (express, implied, or otherwise), including implied warranties of merchantability, non-infringement, fitness for a particular purpose, or title, related to the materials.  The entire risk as to implementing or otherwise using the materials is assumed by the implementer and user. IN NO EVENT WILL THE CONTRIBUTORS OR LICENSEES BE LIABLE TO ANY OTHER PARTY FOR LOST PROFITS OR ANY FORM OF INDIRECT, SPECIAL, INCIDENTAL, OR CONSEQUENTIAL DAMAGES OF ANY CHARACTER FROM ANY CAUSES OF ACTION OF ANY KIND WITH RESPECT TO THIS DELIVERABLE OR ITS GOVERNING AGREEMENT, WHETHER BASED ON BREACH OF CONTRACT, TORT (INCLUDING NEGLIGENCE), OR OTHERWISE, AND WHETHER OR NOT THE OTHER MEMBER HAS BEEN ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

## Introduction

This specification documents the new protocol Git uses when interracting with cryptographic signing and verification tools. The protocol takes is inspired by the [Assuan Protocol][4] used by GPG to link its component executables together.
 
### Scope

This specification...
* Describes the protocol Git uses for communicating with external processes for the purposes of executing cryptographic operations.
* Defines the format for storing cryptographic data inside of Git objects.

### Normative References

There are no normative references in this document.

### Terms and Definitions

For the purposes of this document, the following terms and definitions apply.

digital signature
: A mathematical scheme for verifying the authenticity of digital data.

signing tool
: An application that accepts data and produces a non-repudiable digital signature over the data.

verification tool
: An application that accepts a digital signature and the signed data and verifies that the digital signature is valid.

server
: The process that is executing either the signing or verification tool to produce or verify a digital signature.

client
: The process that executes the server process and passes data to it and receives back results. This is always the Git process.

ISO and IEC maintain terminological databases for use in standardization at the following addresses:

* ISO Online browsing platform: available at https://www.iso.org/obp
* IEC Electropedia: available at http://www.electropedia.org/

## The Protocol

The protocol is line based with a maximum line length of 1000 octets. All data is text encoded in UTF-8 encoding. Binary data is encoded as hexadecimal text. It is designed as a client-server protocol with the client being the initiator of the connection and the server the receiver. In the case of Git, it uses the pipe-fork mechanism to spawn a new task and then the protocol is used over the stdin/stdout pipes between the processes. The pipe-forked child process acts as the server and the parent Git process is the client.

In the rest of this document, there are example sessions demonstrating the protocol. In those examples, lines that begin with "S:" are lines sent by the server (the child process) and lines that begin with "C:" are lines sent by the client (the git process).

## The Server

The server must support just four commands: `OK`, `ERR`, `D`, and `#`. Each command is documented below:

```
OK [<arbitrary debugging information>]
```
The request or operation was successful.

```
ERR [<human readable error description>]
```
The request or operation failed. The error code is used to signal the exact error.

```
D <raw data>
```
Sends raw data to the client. There must be exactly one space after the 'D'. The values for '%', carriage return (0x0d), and line feed (0x0a) must be escaped using percent escaping of their ascii values in hexadecimal. So '%' is encoded as '%25' and carriage return and line feed are %0D and %0A respectively. Only uppercase letters should be used in the hexadecimal representation. Other characters may be percent escaped for easier debugging. All data lines are considered one data stream up to a terminating OK or ERR message.

```
# <string>
```
Comment line issued only for debugging purposes and totally ignored by clients.

In the Assuan protocol documentation there is a command called `INQUIRE` that is used by servers to ask clients for information required for specific operations. In this first implementation of the Git signing protocol, the `INQUIRE` command is specifically excluded because it is not needed for the immediate goal of supporting commit signing with GPG, GPGSM, and OpenSSH. In the future, signing tools that have interactive protocols may require the addition of the `INQUIRE` server command. For now though, it is left out to simplify implementation.

## The Client

The client must support five basic commands: `D`, `END`, `OPTION`, `BYE` and `#`. In addition they must also support the signing command: `SIGN`. It also must support the verification commands: `KEY`, `SIGNATURE`, and `VERIFY`. Each of the commands are documented below:

```
D <raw data>
```
Sends raw data to the server. This command is the same as the D command described above in the server section.

```
END
```
Used by the client to mark the end of raw data.

```
OPTION name [[=] value]
```
Sets and option for the current session. Leading and trailing spaces around the *name* and *value* are allowed but should be ignored. The use of the equal sign is optional but is suggested if *value* is given.

```
BYE
```
Tells the server the client is finished. The server will respond with OK, then the client can close the connection.

```
# <string>
```
Comment line issued only for debugging purposes and totally ignored by clients.

```
SIGN
```
The sign command initiates a cryptographic signing operation. Immediately following the SIGN command, the client must send one or more `D` commands sending the server the data to be signed. The data is encoded as hexadecimal string to simplify client implementation. The data is terminated with an `END` command, signaling to the server to sign the data and return the signature.

The server will respond with one or more `D` commands sending to the client the resulting data from the `SIGN` operation. The data sent by the server is terminated with either an `OK` command on success or an `ERR` command on error. The data sent from the server contains tagged values that are to be stored in the Git object. These tagged values may include `sigtype`, `sigoptions`, `sigkey`, and `sig` values. They are significant for the verification process and described below in the section on verification.

```
KEY <sigkey data>
```
Send `sigkey` data, if any, to the server before a `VERIFY` command. This is used for signature schemes that publish the public key along with the signature.

```
SIGNATURE
```
Initiate the transfer of the `sig` data to the server. Immediately following the `SIGNATURE` command, the client must send one or more `D` command sending the signature data to the server. The data is terminated with an `END` command. The server will respond with `OK` if successful or `ERR` if there is an error. The signature data must be sent to the server before a `VERIFY` command is issued.

```
VERIFY
```
Initiate the transfer of the signed object data to the server and execute the signature verification operation. Immediately after the `VERIFY` command, the client must send one or more `D` commands to send the signed object data to the server. The data is terminated with an `END` command, signaling the server to execute the signature verification. The server will respond with zero or more `D` lines with status information about the digital signature. The server will then send either an `OK`, if the verification was successful, or `ERR` if not.

## Signing a Git Object

The general flow of the Git object signing process is as follows:

1. Git calculates the signing executable to execute from the config file and command line options.
2. Git pipe-forks a child process to execute the signing executable. For the purposes of the protocol, Git is the client and the signing tool is the server.
3. The signing tool starts the session by sending an `OK` command.
4. Git issues zero or more `OPTION` commands to the signing tool to pass the options from the config file and command line that relate to the signing operation. The signing tool responds with `OK` in response to each `OPTION` if the option is valid. If the option is not valid, the signing tool responds with an `ERR` command and the signing session will en following steps 8 and 9 below.
5. Git issues the `SIGN` command followed by one or more `D` commands to pass the Git object data to the signing tool to be signed. Git sends and `END` command after the last `D` command to signal the end of the data.
6. If the signing tool successfully signs the data, it responds with one or more `D` commands containing data that must be stored in the Git object verbatim. The signing tool sends the `OK` command after the last `D` command to signal a successful signing.
7. If the signing tool fails to sign the data, it responds with zero or more `D` commands containing detailed error data to be output from Git's stderr stream. The signing tool sends the `ERR` command after the last `D` command to signal a failed signing and the optional "reason" string with the `ERR` command is also output to Git's stderr stream.
8. Git then sends the `BYE` command to conclude the signing session.
9. The signing tool acknowledges the session ending by sending an `OK` command and exiting execution.

An example successful signing session is illustrated below. The lines beginning with "S: " are sent from the signing tool to Git and lines starting with "C:" are sent from Git to the signing tool. **NOTE:** Every byte of the Git object data passed to the signing tool is significant, this includes the line feed (0x0A) character at the end of each line. To pass line feed as data to the signing tool it must be escaped as "%0A". The returned signature data also has significant line feeds and will also have escaped line feed characters.

```
S: OK
C: OPTION identity=Jane Hacker <jane@h.com>
S: OK
C: OPTION min_trust_level=marginal
S: OK
C: OPTION armored=true
S: OK
C: OPTION detached=true
S: OK
C: SIGN
C: D tag v0.0.1
C: D Tagger: Jane Hacker <jane@h.com>
C: D
C: D First release.
C: END
S: D sigtype:openpgp
S: D sigoption:min_trust_level=marginal
S: D sig:-----BEGIN PGP SIGNATURE-----
S: D sig:
S: D sig:iHUEABYKAB0WIQTXto4BPKlfA2YYS5Pn3hDaTgk8fAUCX5C+ugAKCRDn3hDaTgk8
S: D sig:fOk8AQCRGkdNGMXhJ95e5QIHk44rvfNsyibxY6ZvTXdLQJvt/gEAlFCeEM3SfaDL
S: D sig:8RQR368L0+caDlaZW51VZVP2UBXP6w0=
S: D sig:=1Fby
S: D sig:-----END PGP SIGNATURE-----
S: OK
C: BYE
S: OK
```

An example of a signing session that fails because of a bad `OPTION` set by Git. In this example Git passes the signing identity and the signing tool does not have an identity by that name so it responds with the `ERR` command and the reason for the error.

```
S: OK
C: OPTION identity=Jane Hacker <jane@h.com>
S: ERR Unknown signing identity
C: BYE
S: OK
```

### The Returned Signature Data

When a signing tool generates a successful signature, it sends to Git one or more `D` commands with signature related data that is intended to be stored verbatim inside of the Git object. The data is formatted so that Git can easily execute a signature verification process in the future. Each line of the signature data starts with a tag and a colon (`:`) followed by data. Each line is a maximum of 1000 octets including the tag, colon, and the line ending characters (e.g. newline and/or carriage return).

There are four different tags that may be used in the signature data that are defined below:

```
sigtype:<signature scheme name>
```
The `sigtype:` tag is used to identify the signing scheme used to generate and verify this signature. The signature scheme name must match the name used in the config file and also on the command line. For GPG signatures the scheme name is `openpgp`. For GPGSM signatures the scheme name is `x509`. For OpenSSH signatures the scheme name is `openssh`. With this design, Git no longer has to know any details specific to any signature scheme and nothing needs to be changed in Git to use new signature schemes in the future. There may only be one `sigtype:` tagged line in a given signature and it must be the first line in the signature data.

```
sigoption:<option name> = <option value>
```
The `sigoption:` tag is used by the signing tool to specify options that Git will pass to the verification tool using the `OPTION` command during a signature verification session. There may be zero or more `sigoption:` lines in the signature data.

```
sigkey:<verification key data>
```
The `sigkey:` tag is used by the signing tool to specify the verification key to be used by the verification tool to verify the signature. There may be zero or more `sigkey:` lines in the signature data.

```
sig:<signature data>
```
The `sig:` tag is used by the signing tool to specify the signature data it generated in the signing operation. There may be one or more `sig:` lines in the signature data and they must come last in the signature.

The resulting signed Git object, in this case a tag, from the successful
signature example above is as follows:

```
tag v0.0.1
Tagger: Jane Hacker <jane@h.com>

First release.
sigtype:openpgp
sigoption:min_trust_level=marginal
sig:-----BEGIN PGP SIGNATURE-----
sig:
sig:iHUEABYKAB0WIQTXto4BPKlfA2YYS5Pn3hDaTgk8fAUCX5C+ugAKCRDn3hDaTgk8
sig:fOk8AQCRGkdNGMXhJ95e5QIHk44rvfNsyibxY6ZvTXdLQJvt/gEAlFCeEM3SfaDL
sig:8RQR368L0+caDlaZW51VZVP2UBXP6w0=
sig:=1Fby
sig:-----END PGP SIGNATURE-----
```

## Verifying a Signed Git Object

The general flow of the signed Git object verification process is as follows:

1. Git parses the `sigtype:` line from the signed object to determine the signature scheme used to generate the signature.
2. Git calculates the verifying executable to execute from the config file and command line options using the signature scheme name.
3. Git pipe-forks a child process to execute the verification executable. For the purposes of the protocol, Git is the client and the verification tool is the server.
4. The verification tool starts the session by sending an `OK` command.
5. Git parses any `sigoption:` lines from the signed object and issues one `OPTION` command for each `sigoption:` line. The verification tool sends an `OK` command in response to each `OPTION` command if the option is valid. If the option is not valid, the verification tool sends an `ERR` command to signal failure and the verification session is terminated using steps 10 and 11 below.
6. Git parses any `sigkey:` lines from the signed object and issues the `KEY` command followed by `D` commands to pass the data from the `sigkey:` lines to the verification tool. Git sends the `END` command after the last `D` command to signal the end of the key data. The verification tool then responds with either an `OK` or `ERR` command to signal success or failure. On failure the verification session is terminated using steps 10 and 11 below.
7. Git parses the `sig:` lines and issues the `SIGNATURE` command followed by `D` commands to send the signature data to the verification tool. Git sends the `END` command after the last `D` command to signal the end of the signature data. The verification tool responds with either an `OK` or `ERR` command to signal success or failure. On failure the verification session is terminated using steps X and Y below.
8. Git sends the `VERIFY` command followed by one or more `D` commands to send the object data to the verification tool for signature verification. Git sends the `END` command to signal the end of the object data.
9. The verification tool responds with one or more `D` commands with the results of the verification process. If the verification process was successful, the verification tool sends the `OK` command after the last `D` command to signal the end of the result data. If the verification failed, the verification tool sends the `ERR` command after the last `D` command to signal the end of the result data.
10. Git then sends the `BYE` command to end the session.
11. The verification tool acknowledges the session ending by sending an `OK` command and then exits execution.

An example successful verification session is illustrated below. The lines beginning with "S: " are sent from the verification tool to Git and lines starting with "C: " are sent from Git to the signing tool.

```
S: OK
C: OPTION min_trust_level=marginal
S: OK
C: SIGNATURE
C: D -----BEGIN PGP SIGNATURE-----
C: D 
C: D iHUEABYKAB0WIQTXto4BPKlfA2YYS5Pn3hDaTgk8fAUCX5C+ugAKCRDn3hDaTgk8
C: D fOk8AQCRGkdNGMXhJ95e5QIHk44rvfNsyibxY6ZvTXdLQJvt/gEAlFCeEM3SfaDL
C: D 8RQR368L0+caDlaZW51VZVP2UBXP6w0=
C: D =1Fby
C: D -----END PGP SIGNATURE-----
C: END
S: OK
C: VERIFY
C: D tag v0.0.1
C: D Tagger: Jane Hacker <jane@h.com>
C: D
C: D First release.
C: END
S: D Sinature made Sun 18 Oct 2020 03:14:17 AM PDT using RSA key ID DFBBCC13
S: D Good signature from "Jane Hacker <jane@h.com>"
S: OK
C: BYE
S: OK
```

An example of a failed verification session is illustrated below.

```
S: OK
C: OPTION min_trust_level=marginal
S: OK
C: SIGNATURE
C: D -----BEGIN PGP SIGNATURE-----
C: D 
C: D iHUEABYKAB0WIQTXto4BPKlfA2YYS5Pn3hDaTgk8fAUCX5C+ugAKCRDn3hDaTgk8
C: D 8RQR368L0+caDlaZW51VZVP2UBXP6w0=
C: D =1Fby
C: D -----END PGP SIGNATURE-----
C: END
S: OK
C: VERIFY
C: D tag v0.0.1
C: D Tagger: Jane Hacker <jane@h.com>
C: D
C: D First release.
C: END
S: D Signature made Sun 18 Oct 2020 03:14:17 AM PDT using RSA key ID DFBBCC13
S: D BAD signature from "Jane Hacker <jane@h.com>"
S: ERR
C: BYE
S: OK
```

## Conclusion

The goal of this modification is to make Git able to use any signing and verification tools that understand this protocol. This eliminates all of the code that is specific to a signing tool and eases maintenance while increasing flexibility.

## Bibliography

[0]: https://www.iso.org/files/live/sites/isoorg/files/developing_standards/docs/en/how-to-write-standards.pdf
[1]: https://www.iso.org/files/live/sites/isoorg/files/developing_standards/docs/en/model_document-rice_model.pdf
[2]: https://github.com/CommunitySpecification/1.0
[3]: https://github.com/TrustFrame/git-cryptography-protocol
[4]: https://www.gnupg.org/documentation/manuals/assuan/index.html
