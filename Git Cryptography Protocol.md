# Git Cryptography Protocol
## Version 0.0.7
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

This specification documents a new, proposed protocol Git uses when interacting with cryptographic signing and verification tools. The goal of this modification is to make Git able to use any signing and verification tools. The design eliminates all of the tool-specific code in Git, easing maintenance and increasing flexibility. The protocol takes is inspired by the [Assuan Protocol][2] used by GPG to link its component executables together but uses [Git's pkt-line framing][3].

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

The protocol uses Git's pkt-line for framing packets as variable length binary strings. The first four bytes of the line, the pkt-len, indicates the total length of the line, in hexadecimal. The pkt-len includes the 4 bytes used to contain the length's hexadecimal representation. The maximum length of a pkt-line's data component is 65515 bytes and pkt-len MUST NOT exceed 65520 (65515 bytes of data + 4 bytes of length data + 1 trailing newline).

Even though pkt-line supports a binary data component, the protocol URL-safe Base64 encodes all binary data sent to, and received from, the signing/verifying tools. The primary reason for this is to ensure that Git can store received signatures directly in tag and commit objects without any processing. All digital signatures are encoded in a variant of URL-safe Base64 called ["Qualified Base64" or QB64][4] which was developed as part of the work at the Decentralized Identity Foundation for encoding all cryptographic constructs in a self-describing way.

The protocol is designed as a client-server protocol with the client being the initiator of the connection and the server the receiver. In the case of Git, it uses the OS's pipe-fork mechanism to spawn a new process and then the protocol is used over the stdin/stdout pipes between the processes. The pipe-forked child process acts as the server and the parent Git process is the client.

In the rest of this document, there are example operations demonstrating the protocol. In those examples, lines that begin with `S:` are lines sent by the server (the child process) and lines that begin with `C:` are lines sent by the client (the git process).

## The Server

The server must support sending just four commands: `OK`, `ERR`, `D`, and `#`. It must support receiving the commands the client sends as documented in the section below labeled "The Client". Each server command is documented below:

```
OK [<arbitrary debugging information>]
```
The request or operation was successful.

```
ERR [<human readable error description>]
```
The request or operation failed. The error code is used to signal the exact error.

```
D <raw data encoded using URL-safe Base64 with no padding>
```
Sends a line of raw data to the client that is encoded with the URL-safe, Base64 scheme with no padding. This differs from the Assuan protocol this is based upon. There must be exactly one space after the `D`. All data lines are considered one data stream up to a terminating `OK` or `ERR` message.

```
# <string>
```
Comment line issued only for debugging purposes and totally ignored by clients.

In the Assuan protocol documentation there is a command called `INQUIRE` that is used by servers to ask clients for information required for specific operations. In this first implementation of the Git signing protocol, the `INQUIRE` command is specifically excluded because it is not needed for the immediate goal of supporting commit signing with GPG, GPGSM, and OpenSSH. In the future, signing tools that have interactive protocols may require the addition of the `INQUIRE` server command. For now though, it is left out to simplify implementation.

## The Client

The client must support sending five basic commands: `D`, `END`, `OPTION`, `BYE` and `#`. In addition to the basic commands the client must also support sending: `SIGN`, `SIGNATURE`, `VERIFY`. The client must support receiving the commands the server sends as documented in the section above labeled "The Server". Each client command is documented below:

```
D <raw data encoded using URL-safe Base64 with no padding>
```
Sends a line of raw data to the server that is encoded with the URL-safe, Base64 scheme with no padding. This command is the same as the `D` command described above in the server section.

```
END
```
Used by the client to mark the end of raw data.

```
OPTION name [[=] value]
```
Sets and option for the current operation. Leading and trailing spaces around the *name* and *value* are allowed but should be ignored. The use of the equal sign is optional but is suggested if *value* is given.

```
BYE
```
Tells the server the client is finished. The server will respond with OK, then the client can close the connection.

```
# <string>
```
Comment line issued only for debugging purposes and totally ignored by servers.

```
SIGN
```
The `SIGN` command initiates a cryptographic signing operation. Immediately following the `SIGN` command, the client must send one or more `D` commands sending the server the data to be signed. The data is encoded as a URL-safe, Base64 encoded string to simplify client implementation. This enables supports 8-bit data streams over the text based protocol. The client first assembles the data to be signed, then Base64 encodes it, then it breaks it up into pkt-lines with the `D` command and sends them to the server. The data is terminated with an `END` command, signaling to the server that the data stream has terminated and it must sign the data and return the signature.

The server responds with one or more `D` commands sending to the client the resulting data from the `SIGN` operation. The data sent by the server is terminated with either an `OK` command on success or an `ERR` command on error. 

```
SIGNATURE
```
Initiate the transfer of the signature data from the Git object. Immediately following the `SIGNATURE` command, the client sends one or more `D` commands to send the signature data found in the `sig` field to the server. The data is terminated with an `END` command. The server responds with `OK` if successful or `ERR` if there is an error. The signature data must be sent to the server *before* a `VERIFY` command is issued.

```
VERIFY
```
Initiate the transfer of the object data to the server required to execute a digital signature verification operation. Immediately after the `VERIFY` command, the client sends one or more `D` commands to send the object data to the server. The data is encoded as a URL-safe, Base64 encoded string. This supports 8-bit data streams and preserves a bit-exact copy of the Git object data. The client first assembles the data that was signed, then Base64 encodes it, then breaks it up into `D` command lines and sends them as pkt-lines to the server. The data is terminated with an `END` command, signaling to the server that the data stream has terminated and it must execute the digital signature verification. The server responds with zero or more `D` lines returning the status information as a URL-safe, Base64 encoded data stream. The server terminates the response stream by sending either an `OK`, if the verification was successful, or `ERR` if not.

## Signing a Git Object

The flow of the Git object signing process is as follows:

1. Git calculates which signing tool to execute from the config file and command line options.
2. Git pipe-forks a child process to execute the signing tool. For the purposes of the protocol, Git is the client and the signing tool is the server.
3. The signing tool starts the operation by sending an `OK` command.
4. Git issues zero or more `OPTION` commands to the signing tool to pass the options from the config file first. The signing tool responds with `OK` in response to each `OPTION` if the option is accepted. If the option is not accepted, the signing tool responds with an `ERR` command and the signing operation ends following steps 9 and 10 below. Git passes all options to the signing tool and some may not be relevant to the signing operation. The signing tool shall ignore all options that are not relevant and only return an `ERR` response to relevant options with invalid values. A good example is an option to set the signer's identifier to a value that is invalid or unknown to the signing tool.
5. Git issues zero or more `OPTION` commands to the signing tool to pass the '--sign-option' options from the command line. Because these come last, they override any prior options from the config file that have the same token. This matches Git's established behavior of command line options overriding config file options.
6. Git issues the `SIGN` command followed by one or more `D` commands to pass the Git object data to the signing tool as a URL-safe, Base64 encoded data stream. Git sends and `END` command after the last `D` command to signal the end of the data stream.
7. If the signing tool successfully signs the data, it responds with one or more `D` commands containing a URL-safe, Base64 encoded data stream that is stored in the Git object as a multi-line `sig` field. The signing tool sends the `OK` command after the last `D` command to signal a successful signing.
8. If the signing tool fails to sign the data, it responds with zero or more `D` commands containing a URL-safe, Base64 encoded data stream that, when decoded, has a detailed error message to be output from Git's stderr stream. The signing tool sends the `ERR` command after the last `D` command to signal a failed signing. The optional "reason" string with the `ERR` command is also output to Git's stderr stream.
9. Git sends the `BYE` command to conclude the signing operation whether is succeeded or failed.
10. The signing tool acknowledges the operation ending by sending an `OK` command and exiting execution.

An example successful signing operation is illustrated below. The lines beginning with `S:` are sent from the signing tool to Git and lines starting with `C:` are sent from Git to the signing tool. **NOTE**: These lines are encoded in pkt-line format that starts with four hexadecimal characters that specify the length of the line. **NOTE:** Every byte of the Git object data passed to the signing tool is significant, this includes the line feed (0x0a) character at the end of each line. To pass the object correctly, it is first encoded using the URL-safe, Base64 encoding scheme to preserve its integrity. The returned signature data is also encoded with the URL-safe, Base64 encoding but does not require decoding by Git. The digital signature is intended to be stored verbatim in the Git object.

```
S: 0007OK
C: 0015# config options
C: 0036OPTION identifier=Awesome Hacker <ah@example.com>
S: 0007OK
C: 0022OPTION minTrustLevel=marginal
S: 0007OK
C: 0018OPTION armored=true
S: 0007OK
C: 0019OPTION detached=true
S: 0007OK
C: 0019# end config options
C: 001b# command line options
C: 0033OPTION identifier=Great Coder <gc@example.net>
S: 0007OK
C: 001f# end command line options
C: 0009SIGN
C: 005dD dGFnIHYwLjAuMQp0YWdnZXI6IEdyZWF0IENvZGVyIDxnY0BleGFtcGxlLm5ldD4KCkZpcnN0IHJlbGVhc2UuCg
C: 0008END
S: 0047D sig 9ALziQIzBAABCgAdFiEEGldQSkb24AhiiioZSDIgduUQvtQFAmDdAL4ACgkQ
S: 0047D  UQvtSIlQ__fty0UB3Iev7k6WCgze___b64gG84-ApWDXrBmcmd0Dd-5Sg3IiXU7
S: 0047D  KEdwIk9YSDYacg5hf_t6E07_3exN-1Nnj50aPIz_3bNcC5tA_7XVMHzY4lRVBVq
S: 0047D  6ayiN7rUd4L7Mt6KzmkgghGbXoif0ouzNpMlwdNq1qD-g1qP-bxZ-87qwwJhQ2-
S: 0047D  V4IpwfpTX7ckuiok1-GWT9-i_opnoN_4-DKh2wyHjmpFAV6AWvh-m4xf9LpLofv
S: 0047D  nHpX0xa_0k8yHIe6ForbN1iNYtrY1ofX3iIuSiee8Ru0fvOApuO1C10EoAraGAM
S: 0047D  g-LySP4Rkjok91lALrcaeT47WSwHvUrOjqUDEW8mF3300KAzFVD6-Kls1VG8iaI
S: 0047D  iBfnrc9Jjli_EkBDATKhZ_ULkpDOep57MmK79spXPluSps3LmpckRHUrCbDJ-oB
S: 0047D  -KmxaK-g7xo8hpP-61yKyxK8ay9hsNgbQmR45vtsXNe2U7wmQ_TcZ_FW78hqtfu
S: 0047D  3k_W8HTVUJ9yCU2wHOVVYq8j_vfHvaY9GsOZX-wxCiZrFVZ3IcPNVrercwv__V5
S: 0047D  Xbi9tAIX3MnIJPgqib8P0UQ6Ib2qcTOuttoghVS-lz5PWNTfj5Ct3NUHqVmBYyQ
S: 0029D  fZh6wNqgAII70Z95zHmMrVLu4PR_Ofnyo
S: 0007OK
C: 0008BYE
S: 0007OK
```

An example of a signing operation that fails because of a bad `OPTION` set by Git. In this example Git passes the signing identity and the signing tool does not have an identity by that name so it responds with the `ERR` command and the reason for the error.

```
S: 0007OK
S: 003bOPTION identifier=Unknown Hacker <unknown@example.org>
S: 001bERR Unknown identifier
S: 0008BYE
S: 0007OK
```

### The Returned Signature Data

When a signing tool generates a successful signature, it sends to Git one or more `D` commands with signature related data that is intended to be stored verbatim inside of the Git object. The data sent is in the form of a multi-line Git `sig` field with a self-describing digital signature. 

```
sig <QB64 encoded signature data>
```
The multi-line `sig` field specifies the signature data generated in the signing operation. The signature data is encoded in the [QB64][4] format which is a URL-safe Base64 encoded type tag followed by a URL-safe Base64 (no padding) encoded digital signature. The `sig` field is wrapped at 64 Base64 characters per line for easy viewing on an 80 column terminal even when included in a mergetag. This design ensures that the signature is stored in a way that not only describes the type of signature but also preserves the raw binary generated by the signing tool. It is the signing tool's responsibility to generate a self-contained, "detached" signature that contains all data necessary for the verification of the signature except the data that was signed.

The resulting signed Git object--in this case a tag--from the successful signature example above is as follows:

```
tag v0.0.1
tagger: Great Coder <gc@example.net>
sig 9ALziQIzBAABCgAdFiEEGldQSkb24AhiiioZSDIgduUQvtQFAmDdAL4ACgkQ
 duUQvtSIlQ__fty0UB3Iev7k6WCgze___b64gG84-ApWDXrBmcmd0Dd-5Sg3IiX
 7c6AKEdwIk9YSDYacg5hf_t6E07_3exN-1Nnj50aPIz_3bNcC5tA_7XVMHzY4lR
 BVqI-v6ayiN7rUd4L7Mt6KzmkgghGbXoif0ouzNpMlwdNq1qD-g1qP-bxZ-87qw
 JhQ2-AgUV4IpwfpTX7ckuiok1-GWT9-i_opnoN_4-DKh2wyHjmpFAV6AWvh-m4x
 9LpLofvNHXnHpX0xa_0k8yHIe6ForbN1iNYtrY1ofX3iIuSiee8Ru0fvOApuO1C
 0EoAraGAMoVSg-LySP4Rkjok91lALrcaeT47WSwHvUrOjqUDEW8mF3300KAzFVD
 -Kls1VG8iaIl-9iBfnrc9Jjli_EkBDATKhZ_ULkpDOep57MmK79spXPluSps3Lm
 ckRHUrCbDJ-oBejH-KmxaK-g7xo8hpP-61yKyxK8ay9hsNgbQmR45vtsXNe2U7w
 Q_TcZ_FW78hqtfu5q53k_W8HTVUJ9yCU2wHOVVYq8j_vfHvaY9GsOZX-wxCiZrF
 Z3IcPNVrercwv__V5dGCXbi9tAIX3MnIJPgqib8P0UQ6Ib2qcTOuttoghVS-lz5
 WNTfj5Ct3NUHqVmBYyQNlSfZh6wNqgAII70Z95zHmMrVLu4PR_Ofnyo

First release.
```

A signed Git commit using the new format looks like:

```
tree eebfed94e75e7760540d1485c740902590a00332
parent 04b871796dc0420f8e7561a895b52484b701d51a
author A U Thor <author@example.com> 1465981137 +0000
committer Great Coder <gc@example.net> 1465981137 +0000
sig 9ALziQIzBAABCgAdFiEEGldQSkb24AhiiioZSDIgduUQvtQFAmDdQisACgkQ
 duUQvtTS9g_-I7GH-Lmbo6RJEWLMig4wd9bA1yKyiUgAWTjBT2mO86Mp4fJPVjK
 VbK8-Jc6p0EdQzS6Eg0wUkHj2Xn8Jh8CMc_21hjBXLWv2suBOtaQlue7K-1TTrd
 mGRw_VFGY4rtJImA25pbT52m7hT3xTkuobnTmbiVkBSYjPFTumlBZFhFc__aT2M
 RW53uIKDL--PV0nuMr_4HKAvtdRJyymrSmrwvFjc2MW3i4OY9RDTtEjcwb3ZTLO
 v1Mbf9Z454cuK3lMaNRoHx8Xio05wQut89LbqwjH0Mw-xVtTvwypX5prLlnzmUD
 QMnnw4thWtiMzrlVwmxzg8Kq1xiitGSIVIVdv6duRsohMQ_lFovB78j8oDO2VLp
 xFoHR34FfaTvmr2n1aMZlIlLlDvxIBk39kIhdy3MtmXvVlE92Smj3aZWmwicUhT
 tJpB-T3TFkruSPh8Ga5TPyZNq7NQnTW-2SVaxLWFROX3NMzLNiyrkbSrSV3jT9v
 hQNrvKeqr8MVVXUHuoPz7B5LiK_-lKCCjMLPHJYkobzV3Jfbo9Br1RDm0yitcUu
 XqB88kLtY-JppSBaQrDMLgg-nZFF9lR4gEUnnj5NO3G130inP2ci0vC5Ctv34xA
 gxypZ_ZCCX7xL5Hj_9Lb8aFvL4kIra0bBVBK9uNhsWRUbr0nKf8hYaE

signed commit

signed commit message body
```

A signed Git mergetag using the new format looks like:

```
tree c7b1cff039a93f3600a1d18b82d26688668c7dea
parent c33429be94b5f2d3ee9b0adad223f877f174b05d
parent 04b871796dc0420f8e7561a895b52484b701d51a
author A U Thor <author@example.com> 1465982009 +0000
committer Awesome Hacker <ah@example.com> 1465982009 +0000
mergetag object 04b871796dc0420f8e7561a895b52484b701d51a
 type commit
 tag signedtag
 tagger Awesome Hacker <ah@example.com> 1465981006 +0000
 sig 9ALziQIzBAABCgAdFiEEqEf0Jz30V4hoCuaLymEAVqGjingFAmDdRA4ACgkQ
  VqGjinjODA_-LBCRq__CeFG9x1O7vPkDofXoI7sAk14OMqbkn6U9f3XaaQ5o1mu
  71HgtC_Fw-gnxM-gtbBDQu1qziq7uQw2Uhgsxp5QUr9LnENxrZtYGnaINWO2COW
  mNXWiKzu6zEUeLF8UO8frMADQXxdu1iYbRtwKbRU5bnpHb2FtuxzY3Gc5muI0ZF
  xWeaJRcsRSaVq0wLDccz98XuMSeTbUuf3Galj_vZmJkXNUOP6G6Ux7yqH2LwDC6
  K5Iw_MzRT5ymZcGNqzJxH0D-liCnoJhighsh_HoTBxHupmyu_v_IQxtytiuAQ1R
  drQEarz3v1A3ju46v3Rx4QSAKBemZ5xm74ZTf1kOC1NiueuwFV6t6ynwN4ddq1Q
  Tnua-jUVndlkgmNO16H8J6LPZaall2IkVydwCmctaFGLW5NQ8AlFC3OdsV1nHfC
  weBvfdTM_Pj7S2ablZto_G8wmPaRoeq3TTBIKLFPLsAYKW4Mi_xMTCLKMwyzOrK
  27qSnUbZv6IfcpfO2Wd94RT5p0CKDH5cB8GXzMlZgecGozEnkse4EjrcwK20R9v
  Cw-VzoY8DGrBJKoN3QV1t40eIbcl_CCbS4FLcL7-4wtBdUR779j_1vcGmcEOiwY
  z6AIfrhHvVj06tQEZ-lQzw9x08YXmQF2d3z3en1vkBHDunRm_I5rBSg

 signed tag

 signed tag message body

Merge tag 'signedtag' into downstream

signed tag

signed tag message body

# gpg: Signature made Wed 30 Jun 2021 09:26:54 PM PDT
# gpg:                using RSA key A847F4273DF45788680AE68BCA610056A1A38A78
# gpg: Good signature from "Awesome Hacker <ah@example.com>" [ultimate]
```

## Verifying a Signed Git Object

The general flow of the signed Git object verification process is as follows:

1. Git parses the QB64 type tag from the multi-line `sig` to determine the signature type.
2. Git calculates the verification tool to execute from the config file and command line options using the signature type.
3. Git pipe-forks a child process to execute the verification tool. For the purposes of the protocol, Git is the client and the verification tool is the server.
4. The verification tool starts the operation by sending an `OK` command.
5. Git issues zero or more `OPTION` commands to the verification tool to pass the options from the config file.
6. Git issues zero or more `OPTION` commands to the verification tool to pass the options from the command line. Because these come after the options from the config file, they override any prior options that have the same name.
7. Git gets the `sig` field value and issues the `SIGNATURE` command followed by `D` commands to send the signature data to the verification tool. Git sends the `END` command after the last `D` command to signal the end of the signature data. The verification tool responds with either an `OK` or `ERR` command to signal success or failure. On failure the verification operation is terminated using steps 10 and 11 below.
8. Git sends gathers the object data, encodes it using the URL-safe, Base64 encoding scheme and then sends the `VERIFY` command followed by one or more `D` commands to send the encoded object data to the verification tool for signature verification. Git sends the `END` command to signal the end of the object data.
9. The verification tool responds with one or more `D` commands with the results of the verification process encoded in URL-safe, Base64 encoding. If the verification process was successful, the verification tool sends the `OK` command after the last `D` command to signal the end of the result data. If the verification failed, the verification tool sends the `ERR` command after the last `D` command to signal the end of the result data.
10. Git then sends the `BYE` command to end the operation.
11. The verification tool acknowledges the operation ending by sending an `OK` command and then exits execution.

An example successful verification operation is illustrated below. The lines beginning with `S:` are sent from the verification tool to Git and lines starting with `C:` are sent from Git to the verification tool.

```
S: 0006OK
C: 001b# signed object options
C: 001eOPTION minTrustLevel=fully
S: 0006OK
C: 001f# end signed object options
C: 0014# config options
C: 002eOPTION identifier=Jane Hacker <jane@h.com>
S: 0006OK
C: 0021OPTION minTrustLevel=marginal
S: 0006OK
C: 0017OPTION armored=true
S: 0006OK
C: 0018OPTION detached=true
S: 0006OK
C: 0018# end config options
C: 001a# command line options
C: 001e# end command line options
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
C: 0026D Tagger: Joe Coder <joe@c.com>%0a
C: 0009D %0a
C: 0017D First release.%0a
C: 0007END
S: 004fD Signature made Sun 18 Oct 2020 03:14:17 AM PDT using RSA key ID DFBBCC13
S: 0031D Good signature from "Joe Coder <joe@c.com>"
S: 0006OK
C: 0007BYE
S: 0006OK
```

An example of a failed verification operation is illustrated below.

```
S: 0006OK
C: 001b# signed object options
C: 001f# end signed object options
C: 0014# config options
C: 002eOPTION identifier=Jane Hacker <jane@h.com>
S: 0006OK
C: 0021OPTION minTrustLevel=marginal
S: 0006OK
C: 0017OPTION armored=true
S: 0006OK
C: 0018OPTION detached=true
S: 0006OK
C: 0018# end config options
C: 001a# command line options
C: 001e# end command line options
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

[0]: https://github.com/CommunitySpecification/1.0
[1]: https://github.com/TrustFrame/git-cryptography-protocol
[2]: https://www.gnupg.org/documentation/manuals/assuan/index.html
[3]: https://github.com/git/git/blob/master/Documentation/technical/protocol-common.txt
[4]: https://github.com/decentralized-identity/keri/blob/master/kids/kid0001.md
