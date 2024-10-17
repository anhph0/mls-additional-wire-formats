
---
<!--
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
-->
title: "MLS Wire Formats for PublicMessage and PrivateMessage without Authenticated_data"
<!--- abbrev: "TODO - Abbreviation" -->
category: info

docname: draft-pham-additional-wire-formats-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: AREA
workgroup: WG Working Group
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: WG
  type: Working Group
  mail: WG@example.com
  arch: https://example.com/WG
  github: USER/REPO
  latest: https://example.com/LATEST

author:
 -
    fullname: Your Name Here
    organization: Your Organization Here
    email: your.email@example.com

normative:

informative:


--- abstract

This document describes an extension to support two new wire formats for MLS messages: PublicMessageWithoutAAD and PrivateMessageWithoutAAD.

--- middle

# Introduction
Sometimes it is desirable to have additional authenticated data to be included in the computation of `MLSMessage` constructions, but not to have it sent on the wire as part of these messages. A use-case is applications that want to have some information available to the server together with a `MLSMessage` and at the same time want to prove the authenticity of the information to other clients. 

An example of this is the case of delivery receipts where the server needs to know that a message from Alice has been delivered to Bob, but at the same time it wants Alice to be able to verify that the delivery receipt indeed comes from Bob.

This document proposes an extension to support new wire formats for MLS `PrivateMessage` and `PublicMessage` to support such cases. Applications will supply in additional data as part of the `MLSMessage` computation, but the additional data is not included in the `MLSMessage`.

# Extension Definition
```
 struct {
     ExtensionType extension_type;
     opaque extension_data<V>;
   } ExtensionContent;
```

Where `extension_type` ExtensionType is a unique uint16 identifier registered in MLS Extension Types IANA registry (see Section 17.3 of [RFC9420]) and `extension_data` is set to an empty vector.

# Message Framing
```
uint16 WireFormat;

struct {
    opaque group_id<V>;
    uint64 epoch;
    Sender sender;

    ContentType content_type;
    select (FramedContent.content_type) {
        case application:
          opaque application_data<V>;
        case proposal:
          Proposal proposal;
        case commit:
          Commit commit;
    };
} FramedContentWithoutAAD;

```

```
struct {
    ProtocolVersion version = mls10;
    WireFormat wire_format;
    select (MLSMessage.wire_format) {
        case mls_public_message:
            PublicMessage public_message;
        case mls_private_message:
            PrivateMessage private_message;
        case mls_welcome:
            Welcome welcome;
        case mls_group_info:
            GroupInfo group_info;
        case mls_key_package:
            KeyPackage key_package;
        
        // Q: I don't really follow the explanation about this in SafeExtension
        // what should I put here?
        case mls_extension_message:
            ExtensionContent extension_content;        
    };
} MLSMessage;
```

# Content Authentication
`FramedContentWithoutAAD` is authenticated using the same procedure for `FramedContent` described in Section 6.1 of [RFC9420]. A difference is that in the `FramedContentTBS` definition, we have `FramedContentWithoutAAD` in lieu of `FramedContent`. 

```
struct {
    ProtocolVersion version = mls10;
    WireFormat wire_format;
    FramedContentWithoutAAD content;
    select (FramedContentTBS.content.sender.sender_type) {
        case member:
        case new_member_commit:
            GroupContext context;
        case external:
        case new_member_proposal:
            struct{};
    };
} FramedContentWithoutAadTBS;

```

Moreover, the `signature` in the `FramedContentAuthData` is computed by using SafeExtension 

```
SignWithLabel(SignatureKey, "LabeledExtensionContent", LabeledExtensionContent)
```
where `LabeledExtensionContent` is defined as:
```
label = Label
extension_type = ExtensionType
extension_data = FramedContentWithoutAadTBS | AdditionalData
```

with `AdditionalData` being supplied by the application. 

# PublicMessageWithoutAAD

```
struct {
    FramedContentWithoutAAD content;
    FramedContentAuthDataWithoutAAD auth;
    select (PublicMessageWithoutAAD.content.sender.sender_type) {
        case member:
            MAC membership_tag;
        case external:
        case new_member_commit:
        case new_member_proposal:
            struct{};
    };
} PublicMessageWithoutAAD;
```

The membership_tag in the `PublicMessageWithoutAAD` authenticates the sender's membership in the group. It is computed as follows:

```
membership_tag = MAC(membership_key, AuthenticatedContentTBM | AdditionalData)
```

with `AuthenticatedContentTBM` and `membership_key` as defined as in the [RFC9420]

<!-- Q: do we need to have an extension label for `membership_key`? -->

# PrivateMessageWithoutAAD
```
struct {
    opaque group_id<V>;
    uint64 epoch;
    ContentType content_type;
    opaque encrypted_sender_data<V>;
    opaque ciphertext<V>;
} PrivateMessageWithoutAAD;
```

Similar to `PrivateMessage`, `encrypted_sender_data` and `ciphertext` are encrypted using the AEAD function specified by the cipher suite in use, using the `SenderData` and `PrivateMessageContent` structures as input. 

The `SenderData` is encrypted using the `sender_data_secret` of the group. 

The actual message content is encrypted using the key derived as follows:

- Derive a `secret` using

```
DeriveExtensionSecret(Secret, Label) =
  ExpandWithLabel(epoch_secret, "ExtensionExport " + ExtensionType + " " + Label)
```
as defined by in Section 2.1.5 of the Extension Framework.

- Use the the `secret` in lieu of `encryption_tree` to seed the Secret Tree (Section 9 of RFC 9420). 
- Follow the procedure of the Secret Tree to generate encryption keys and nonces for the encryption of the message content.

<!--
# Conventions and Definitions

#{::boilerplate bcp14-tagged}
-->

# Security Considerations

No implications on the security of the base MLS protocol due to the use of SafeExtension.


# IANA Considerations

This document requests the addition of various new values under the heading of "Messaging Layer Security".

--- back

# Acknowledgments
#{:numbered="false"}

#TODO acknowledge.

