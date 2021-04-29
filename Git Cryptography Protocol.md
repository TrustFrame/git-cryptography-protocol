# Git Cryptography Protocol
## Version 0.0.1
## Status Pre-draft

© 2021 TrustFrame, Inc.

This specification is subject to the [Community Specification License 1.0][0].

## Contents

1. [Foreword](#foreword)
2. [Introduction](#introduction)
   1. [Scope](#scope)
   2. [Normative References](#normative-references)
   3. [Terms and Definitions](#terms-and-definitions)
3. [The Protocol](#the-protocol)
4. [The Server](#the-server)
5. [The Client](#the-client)
6. [Signing a Git Object](#signing-a-git-object)
   1. [The Returned Signature Data](#the-returned-signature-data)
7. [Verifying a Signed Git Object](#verifying-a-signed-git-object)
8. [Conclusion](#conclusion)

## Foreword

Attention is drawn to the possibility that some of the elements of this document may be the subject of patent rights. No party shall not be held responsible for identifying any or all such patent rights.

Any trade name used in this document is information given for the convenience of users and does not constitute an endorsement.

This document was prepared by TrustFrame, Inc.

Known patent licensing exclusions are available in the specification’s repository’s Notices.md file.

Any feedback or questions on this document should be directed to the [specification repository][1].

THESE MATERIALS ARE PROVIDED “AS IS.” The Contributors and Licensees expressly disclaim any warranties (express, implied, or otherwise), including implied warranties of merchantability, non-infringement, fitness for a particular purpose, or title, related to the materials.  The entire risk as to implementing or otherwise using the materials is assumed by the implementer and user. IN NO EVENT WILL THE CONTRIBUTORS OR LICENSEES BE LIABLE TO ANY OTHER PARTY FOR LOST PROFITS OR ANY FORM OF INDIRECT, SPECIAL, INCIDENTAL, OR CONSEQUENTIAL DAMAGES OF ANY CHARACTER FROM ANY CAUSES OF ACTION OF ANY KIND WITH RESPECT TO THIS DELIVERABLE OR ITS GOVERNING AGREEMENT, WHETHER BASED ON BREACH OF CONTRACT, TORT (INCLUDING NEGLIGENCE), OR OTHERWISE, AND WHETHER OR NOT THE OTHER MEMBER HAS BEEN ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

## Introduction

This specification documents the new protocol Git uses when interracting with cryptographic signing and verification tools. The protocol takes is inspired by the [Assuan Protocol][2] used by GPG to link its component executables together. This protocol differs from the Assuan Protocol by reducing the command set and also uses [Git's pkt-line framing][3].

### Scope

This specification...
* Describes the protocol Git uses for communicating with external processes for the purposes of executing cryptographic operations.
* Defines the format for storing cryptographic data inside of Git objects.

### Normative References

There are no normative references in this document.

### Terms and Definitions

For the purposes of this document, the following terms and definitions apply.

**digital signature**
: A mathematical scheme for verifying the authenticity of digital data.

**signing tool**
: An application that accepts data and produces a non-repudiable digital signature over the data.

**verification tool**
: An application that accepts a digital signature and the signed data and verifies that the digital signature is valid.

**server**
: The process that is executing either the signing or verification tool to produce or verify a digital signature.

**client**
: The process that executes the server process and passes data to it and receives back results. This is always the Git process.

ISO and IEC maintain terminological databases for use in standardization at the following addresses:

* ISO Online browsing platform: available at https://www.iso.org/obp
* IEC Electropedia: available at http://www.electropedia.org/

## The Protocol

The protocol uses Git's pkt-line for framing packets as variable length binary strings. The first four bytes of the line, the pkt-len, indicates the total length of the line, in hexadecimal. The pkt-len includes the 4 bytes used to contain the length's hexadecimal representation. The maximum length of a pkt-line's data component is 65516 bytes and pkt-len MUST NOT exceed 65520 (65516 bytes of data + 4 bytes of length data).

Even though pkt-line supports a binary data component, the protocol Base64 encodes all binary data sent to, and received from, the signing/verifying tools. The primary reason for this is to ensure that Git can store received signatures directly in tag and commit objects without any processing. This is compatible with the assumptions about the existing GPG signing code that instructs GPG to create ASCII encoded signatures that are stored directly in tag and commit objects.

The protocol is designed as a client-server protocol with the client being the initiator of the connection and the server the receiver. In the case of Git, it uses the pipe-fork mechanism to spawn a new task and then the protocol is used over the stdin/stdout pipes between the processes. The pipe-forked child process acts as the server and the parent Git process is the client.

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
Sends raw data to the client. There must be exactly one space after the 'D'. The values for '%', carriage return (0x0d), and line feed (0x0a) must be escaped using percent escaping of their ascii values in hexadecimal. So '%' is encoded as '%25' and carriage return and line feed are '%0d' and '%0a' respectively. Other characters may be percent escaped for easier debugging. All data lines are considered one data stream up to a terminating OK or ERR message.

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

An example successful signing session is illustrated below. The lines beginning with "S: " are sent from the signing tool to Git and lines starting with "C:" are sent from Git to the signing tool. **NOTE**: These lines are encoded in pkt-line format that starts with four hexadecimal characters that specify the length of the line. **NOTE:** Every byte of the Git object data passed to the signing tool is significant, this includes the line feed (0x0a) character at the end of each line. To pass line feed as data to the signing tool it must be escaped as '%0a'. The returned signature data also has significant line feeds and will also have escaped line feed characters.

```
S: 0006OK
C: 002eOPTION identifier=Jane Hacker <jane@h.com>
S: 0006OK
C: 001fOPTION min_trust_level=marginal
S: 0006OK
C: 0017OPTION armored=true
S: 0006OK
C: 0018OPTION detached=true
S: 0006OK
C: 0008SIGN
C: 0013D tag v0.0.1%0a
C: 0029D Tagger: Jane Hacker <jane@h.com>%0a
C: 0005D %0a
C: 0017D First release.%0a
C: 0007END
S: 0015D sigtype openpgp
S: 0028D sigoption min_trust_level=marginal
S: 002aD sig -----BEGIN PGP SIGNATURE-----%0a
S: 000dD sig %0a
S: 004dD sig iHUEABYKAB0WIQTXto4BPKlfA2YYS5Pn3hDaTgk8fAUCX5C+ugAKCRDn3hDaTgk8%0a
S: 004dD sig fOk8AQCRGkdNGMXhJ95e5QIHk44rvfNsyibxY6ZvTXdLQJvt/gEAlFCeEM3SfaDL%0a
S: 002dD sig 8RQR368L0+caDlaZW51VZVP2UBXP6w0=%0a
S: 0012D sig =1Fby%0a
S: 0028D sig -----END PGP SIGNATURE-----%0a
S: 0006OK
C: 0007BYE
S: 0006OK
```

An example of a signing session that fails because of a bad `OPTION` set by Git. In this example Git passes the signing identity and the signing tool does not have an identity by that name so it responds with the `ERR` command and the reason for the error.

```
S: 0006OK
C: 002eOPTION identifier=Jane Hacker <jane@h.com>
S: 001aERR Unknown identifier
C: 0007BYE
S: 0006OK
```

### The Returned Signature Data

When a signing tool generates a successful signature, it sends to Git one or more `D` commands with signature related data that is intended to be stored verbatim inside of the Git object. The data is formatted so that Git can easily execute a signature verification process in the future. Each line of the signature data starts with a tag and a space (` `) followed by data. Each line of the signature is a maximum of 1000 octets including the tag, space, and the line ending characters (e.g. newline and/or carriage return).

There are four different tags that may be used in the signature data that are defined below:

```
sigtype <signature scheme name>
```
The `sigtype` tag is used to identify the signing scheme used to generate and verify this signature. The signature scheme name must match the name used in the config file and also on the command line. For GPG signatures the scheme name is `openpgp`. For GPGSM signatures the scheme name is `x509`. For OpenSSH signatures the scheme name is `openssh`. With this design, Git no longer has to know any details specific to any signature scheme and nothing needs to be changed in Git to use new signature schemes in the future. There may only be one `sigtype` tagged line in a given signature and it must be the first line in the signature data.

```
sigoption <option name> = <option value>
```
The `sigoption` tag is used by the signing tool to specify options that Git will pass to the verification tool using the `OPTION` command during a signature verification session. There may be zero or more `sigoption` lines in the signature data.

```
sigkey <verification key data>
```
The `sigkey` tag is used by the signing tool to specify the verification key to be used by the verification tool to verify the signature. There may be zero or more `sigkey` lines in the signature data.

```
sig <signature data>
```
The `sig` tag is used by the signing tool to specify the signature data it generated in the signing operation. There may be one or more `sig` lines in the signature data and they must come last in the signature.

The resulting signed Git object--in this case a tag--from the successful
signature example above is as follows:

```
tag v0.0.1
Tagger: Jane Hacker <jane@h.com>

First release.
sigtype openpgp
sigoption min_trust_level=marginal
sig -----BEGIN PGP SIGNATURE-----%0a
sig %0a
sig iHUEABYKAB0WIQTXto4BPKlfA2YYS5Pn3hDaTgk8fAUCX5C+ugAKCRDn3hDaTgk8%0a
sig fOk8AQCRGkdNGMXhJ95e5QIHk44rvfNsyibxY6ZvTXdLQJvt/gEAlFCeEM3SfaDL%0a
sig 8RQR368L0+caDlaZW51VZVP2UBXP6w0=%0a
sig =1Fby%0a
sig -----END PGP SIGNATURE-----%0a
```

A signed Git commit using the new tagging system would look like:

```
tree eebfed94e75e7760540d1485c740902590a00332
parent 04b871796dc0420f8e7561a895b52484b701d51a
author A U Thor <author@example.com> 1465981137 +0000
committer C O Mitter <committer@example.com> 1465981137 +0000
sigtype openpgp
sigoption min_trust_level=marginal
sig -----BEGIN PGP SIGNATURE-----%0a
sig Version: GnuPG v1%0a
sig %0a
sig iQEcBAABAgAGBQJXYRjRAAoJEGEJLoW3InGJ3IwIAIY4SA6GxY3BjL60YyvsJPh/%0a
sig HRCJwH+w7wt3Yc/9/bW2F+gF72kdHOOs2jfv+OZhq0q4OAN6fvVSczISY/82LpS7%0a
sig DVdMQj2/YcHDT4xrDNBnXnviDO9G7am/9OE77kEbXrp7QPxvhjkicHNwy2rEflAA%0a
sig zn075rtEERDHr8nRYiDh8eVrefSO7D+bdQ7gv+7GsYMsd2auJWi1dHOSfTr9HIF4%0a
sig HJhWXT9d2f8W+diRYXGh4X0wYiGg6na/soXc+vdtDYBzIxanRqjg8jCAeo1eOTk1%0a
sig EdTwhcTZlI0x5pvJ3H0+4hA2jtldVtmPM4OTB0cTrEWBad7XV6YgiyuII73Ve3I=%0a
sig =jKHM%0a
sig -----END PGP SIGNATURE-----%0a

signed commit

signed commit message body
```

A signed Git mergetag using the new tagging system would look like:

```
tree c7b1cff039a93f3600a1d18b82d26688668c7dea
parent c33429be94b5f2d3ee9b0adad223f877f174b05d
parent 04b871796dc0420f8e7561a895b52484b701d51a
author A U Thor <author@example.com> 1465982009 +0000
committer C O Mitter <committer@example.com> 1465982009 +0000
mergetag object 04b871796dc0420f8e7561a895b52484b701d51a
 type commit
 tag signedtag
 tagger C O Mitter <committer@example.com> 1465981006 +0000

 signed tag

 signed tag message body
 sigtype openpgp
 sigoption min_trust_level=marginal
 sig -----BEGIN PGP SIGNATURE-----%0a
 sig Version: GnuPG v1%0a
 sig %0a
 sig iQEcBAABAgAGBQJXYRhOAAoJEGEJLoW3InGJklkIAIcnhL7RwEb/+QeX9enkXhxn%0a
 sig rxfdqrvWd1K80sl2TOt8Bg/NYwrUBw/RWJ+sg/hhHp4WtvE1HDGHlkEz3y11Lkuh%0a
 sig 8tSxS3qKTxXUGozyPGuE90sJfExhZlW4knIQ1wt/yWqM+33E9pN4hzPqLwyrdods%0a
 sig q8FWEqPPUbSJXoMbRPw04S5jrLtZSsUWbRYjmJCHzlhSfFWW4eFd37uquIaLUBS0%0a
 sig rkC3Jrx7420jkIpgFcTI2s60uhSQLzgcCwdA2ukSYIRnjg/zDkj8+3h/GaROJ72x%0a
 sig lZyI6HWixKJkWw8lE9aAOD9TmTW9sFJwcVAzmAuFX2kUreDUKMZduGcoRYGpD7E=%0a
 sig =jpXa%0a
 sig -----END PGP SIGNATURE-----%0a

Merge tag 'signedtag' into downstream

signed tag

signed tag message body

# gpg: Signature made Wed Jun 15 08:56:46 2016 UTC using RSA key ID B7227189
# gpg: Good signature from "Eris Discordia <discord@example.net>"
# gpg: WARNING: This key is not certified with a trusted signature!
# gpg:          There is no indication that the signature belongs to the owner.
# Primary key fingerprint: D4BE 2231 1AD3 131E 5EDA  29A4 6109 2E85 B722 7189
```

## Verifying a Signed Git Object

The general flow of the signed Git object verification process is as follows:

1. Git parses the `sigtype` line from the signed object to determine the signature scheme used to generate the signature.
2. Git calculates the verifying executable to execute from the config file and command line options using the signature scheme name.
3. Git pipe-forks a child process to execute the verification executable. For the purposes of the protocol, Git is the client and the verification tool is the server.
4. The verification tool starts the session by sending an `OK` command.
5. Git parses any `sigoption` lines from the signed object and issues one `OPTION` command for each `sigoption` line. The verification tool sends an `OK` command in response to each `OPTION` command if the option is valid. If the option is not valid, the verification tool sends an `ERR` command to signal failure and the verification session is terminated using steps 10 and 11 below.
6. Git parses any `sigkey` lines from the signed object and issues the `KEY` command followed by `D` commands to pass the data from the `sigkey` lines to the verification tool. Git sends the `END` command after the last `D` command to signal the end of the key data. The verification tool then responds with either an `OK` or `ERR` command to signal success or failure. On failure the verification session is terminated using steps 10 and 11 below.
7. Git parses the `sig` lines and issues the `SIGNATURE` command followed by `D` commands to send the signature data to the verification tool. Git sends the `END` command after the last `D` command to signal the end of the signature data. The verification tool responds with either an `OK` or `ERR` command to signal success or failure. On failure the verification session is terminated using steps X and Y below.
8. Git sends the `VERIFY` command followed by one or more `D` commands to send the object data to the verification tool for signature verification. Git sends the `END` command to signal the end of the object data.
9. The verification tool responds with one or more `D` commands with the results of the verification process. If the verification process was successful, the verification tool sends the `OK` command after the last `D` command to signal the end of the result data. If the verification failed, the verification tool sends the `ERR` command after the last `D` command to signal the end of the result data.
10. Git then sends the `BYE` command to end the session.
11. The verification tool acknowledges the session ending by sending an `OK` command and then exits execution.

An example successful verification session is illustrated below. The lines beginning with "S: " are sent from the verification tool to Git and lines starting with "C: " are sent from Git to the signing tool.

```
S: 0006OK
C: 0023OPTION min_trust_level=marginal
S: 0006OK
C: 000dSIGNATURE
C: 0026D -----BEGIN PGP SIGNATURE-----%0a
C: 0009D %0a
C: 0049D iHUEABYKAB0WIQTXto4BPKlfA2YYS5Pn3hDaTgk8fAUCX5C+ugAKCRDn3hDaTgk8%0a
C: 0049D fOk8AQCRGkdNGMXhJ95e5QIHk44rvfNsyibxY6ZvTXdLQJvt/gEAlFCeEM3SfaDL%0a
C: 0029D 8RQR368L0+caDlaZW51VZVP2UBXP6w0=%0a
C: 000eD =1Fby%0a
C: 0024D -----END PGP SIGNATURE-----%0a
C: 0007END
S: 0006OK
C: 000aVERIFY
C: 0013D tag v0.0.1%0a
C: 0029D Tagger: Jane Hacker <jane@h.com>%0a
C: 0009D %0a
C: 0017D First release.%0a
C: 0007END
S: 004dD Sinature made Sun 18 Oct 2020 03:14:17 AM PDT using RSA key ID DFBBCC13
S: 0034D Good signature from "Jane Hacker <jane@h.com>"
S: 0006OK
C: 0007BYE
S: 0006OK
```

An example of a failed verification session is illustrated below.

```
S: 0006OK
C: 0023OPTION min_trust_level=marginal
S: 0006OK
C: 000dSIGNATURE
C: 0026D -----BEGIN PGP SIGNATURE-----%0a
C: 0009D %0a
C: 0049D iHUEABYKAB0WIQTXto4BPKlfA2YYS5Pn3hDaTgk8fAUCX5C+ugAKCRDn3hDaTgk8%0a
C: 0029D 8RQR368L0+caDlaZW51VZVP2UBXP6w0=%0a
C: 000eD =1Fby%0a
C: 0024D -----END PGP SIGNATURE-----%0a
C: 0007END
S: 0006OK
C: 000aVERIFY
C: 0013D tag v0.0.1%0a
C: 0029D Tagger: Jane Hacker <jane@h.com>%0a
C: 0009D %0a
C: 0017D First release.%0a
C: 0007END
S: 004eD Signature made Sun 18 Oct 2020 03:14:17 AM PDT using RSA key ID DFBBCC13
S: 0033D BAD signature from "Jane Hacker <jane@h.com>"
S: 0007ERR
C: 0007BYE
S: 0006OK
```

## Conclusion

The goal of this modification is to make Git able to use any signing and verification tools that understand this protocol. This eliminates all of the code that is specific to a signing tool and eases maintenance while increasing flexibility.

[0]: https://github.com/CommunitySpecification/1.0
[1]: https://github.com/TrustFrame/git-cryptography-protocol
[2]: https://www.gnupg.org/documentation/manuals/assuan/index.html
[3]: https://github.com/git/git/blob/master/Documentation/technical/protocol-common.txt
