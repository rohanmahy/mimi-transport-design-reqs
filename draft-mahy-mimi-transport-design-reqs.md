---
title: Design Requirements for the More Instant Messaging Interoperability (MIMI) Transport Protocol
abbrev: MIMI Design Requirements
docname: draft-mahy-mimi-transport-design-reqs-latest
ipr: trust200902
submissiontype: IETF  # also: "independent", "IAB", or "IRTF"area: art
workgroup: MIMI
area: art
category: info
keyword:
 - design requirements
 - design reqs
 - MIMI transport

stand_alone: yes
pi: [toc, sortrefs, symrefs]

venue:
  group: MIMI
  type: Working Group
  mail: mimi@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/mimi/
  github: rohan-wire/mimi-groupchat/
#  latest: https://github.com/rohan-wire/mimi-groupchat/latest

author:
 -  ins: R. Mahy
    name: Rohan Mahy
    organization: Wire
    email: rohan.mahy@wire.com

normative:

informative:
  DoubleRatchet:
    target: https://signal.org/docs/specifications/doubleratchet
    title: The Double Ratchet Algorithm
    author:
     - name: Trevor Perrin
       organization: Signal
     - name: Moxie Marlinspike
       organization: Signal
    date: 2016-11-20

--- abstract

This document describes design requirements on the More Instant Messaging
Interoperability (MIMI) Working Group provider-to-provider message transport
protocol. These requirements are based on the requirements of the group
encryption using the Messaging Layer Security (MLS) protocol and
requirements for high volume message transfer which would be needed with
large messaging providers.

--- middle

# Introduction

As described in the Group Chat Framework for More Instant Messaging
Interoperability (MIMI) {{?I-D.mahy-mimi-group-chat}}, the basic operations
of users creating, joining, and leaving a chat, map to specific primitives in the
Messaging Layer Security (MLS) protocol {{!I-D.ietf-mls-protocol}}.
Some of these primitives have implications on the design of the MIMI
inter-provider message transport protocol.

This document describes constraints and requirements to be used during the
design of that protocol.

# Conventions and Definitions

The terms MLS client, MLS group, Proposal, Commit, External Commit,
external join, group_id, epoch, Welcome, KeyPackage, GroupInfo,
and GroupContext have the same meanings as in the MLS
protocol {{!I-D.ietf-mls-protocol}}.

An MLS **KeyPackage** (KP) is used to establish initial keying material in a group,
analogous to DoubleRatchet prekeys, except one KP is used for a client per group
but each recipient does not require a separate one.

An MLS **GroupInfo** (GI) object is the information needed for a client to
externally join an MLS group using an External Commit. The GroupInfo
changes with each MLS epoch.

The terms in this document and {{?I-D.ralston-mimi-terminology}} have not yet
been aligned.

**Room**:
: A room, also known as a chat room or group chat, is a virtual space users
figuratively enter in order to participate in text-based conferencing.
When used with MLS it typically has a 1:1 relationship with an MLS group.

**User**:
: A single human user or automated agent (ex: chat bot) with a distinct identifiable
representation in a room.

**Client**:
: An instant messaging agent instance associated with a specific user account on a
specific device. For example, the mobile phone instance used by the user
@alice@example.com.

**Owning Provider:**
: For a given room, the owning provider is the provider which is authoritative
for room policy and which determines which Commit to accept if more than one
valid Commit arrives for the same epoch.

The rest of this document is separated into sections based on distinct
handling requirements within the protocol. 

# Primitives which request an exclusive resource

Some primitives request use of an exclusive resource. These could be
handled differently from other requests in a high availability environment.

## Claiming a single-use KeyPackage

Claiming single-use KeyPackages requires that the target's domain is online, that
the requestor has consent to claim them, and there are sufficient
KeyPackages uploaded by the client that are valid and have parameters that are
compatible with the request.

One provider should be able to claim all the KeyPackages needed by the original
requestor, for all users in a target provider in a single request. For example,
say Alice at provider A wants KeyPackages for:

- Bobby at provider B
- Betty at provider B
- Bruce at provider B
- Bella at provider B
- Cathy at provider C
- Carl at provider C

Provider A should be able to send a single request to provider B, for
KeyPackages for all of Bobby, Betty, Bruce, and Bella's clients.

Below is the list of fields which should be provided when claiming KeyPackages:

- the requesting user's user ID (or pseudonym ID)
- the list of requested user IDs (only user IDs associated with the target provider)
- the intended room ID (optional)
- a list of MLS versions (or version mls10 if not provided)
- an ordered ciphersuite list (or the default ciphersuite if not provided)
- any required capabilities (optional)

If authorized, this should return a list of KeyPackages (which comply with the
restrictions) for all clients of the requested users.

If the requestor is authorized to request KeyPackages from the target user,
there should be either an error or a KeyPackage for each client of the target user.

For example, response from provider B:

- Bobby
  - client Bob1: KP
  - client Bob2: KP
  - client Bob3: KP
- Betty
  - client Betty1: KP
  - client Betty2: no compatible KPs
- Bruce: no consent to provide KPs
- Bella: user deleted

Note: A similar API could be used between clients and their own provider. If so,
if the client requests KeyPackages for their own user, presumably the client
wants KeyPackages for all of their clients except the requesting client.
This guidance is out-of-scope for MIMI, but is a sufficient gotcha to merit a note.

In the DoubleRatchet {{DoubleRatchet}} protocol, a client requests prekeys for every
other client with which it communicates (one prekey for correspondant).
If A, B, C, D, and E have never communicated and want to form a room, each client
needs a prekey for the other 4 (20 prekeys total). However the prekeys are used
for a session regardless of the number of rooms involved. If A, B, C, and X form
a new room, only 6 new prekeys are needed.

In MLS, only one KeyPackage per member is needed per group, but a different
KeyPackage is needed per group. If A, B, C, D and E join a group, only 5 KeyPackages
are required. However when A, B, C, and X form a different group, 4 more KeyPackages
are needed.

The implication is that it is harder to rate-limit the consumption of KeyPackages
in MLS, but straightforward to associate KeyPackages with consent relationships
and specific uses. KeyPackages are needed even when a client is temporarily
offline, so KeyPackage exhaustion is an important DoS attack vector we want to
prevent.

## Sending a Commit

Sending a Commit requires exclusive access to the group's epoch on the
owning provider, which needs to be online and available. By exclusive access, we mean
that if multiple otherwise valid commits are received for the same epoch,
only one of them can be accepted.

The most efficient way to send a Commit is as a Commit bundle.
A Commit bundle consists of a Commit, GroupInfo, an optional Welcome, and
enough information necessary so the responsible domain can provide the
`ratchet_tree` associated with the group.

| Number | Component |
|:-------|:----------|
| 1      | Commit |
| 0 or 1 | Welcome |
| 1      | GroupInfo |
| 1      | ratchet_tree container |

The Welcome is
the Welcome that the Committer would send if the Commit is accepted, and the
GroupInfo object is the new GroupInfo that would be valid in the new epoch if the
Commit is accepted. The GroupInfo needs to include an `external_pub` extension
so it can be used for external joins in the new epoch if the Commit is accepted.

If a Commit bundle is rejected because the epoch has already advanced,
the Commit, and the tentative Welcome and
GroupInfo need to be discarded by the requesting client. If the operation
represented by the rejected Commit is still relevant, the requester can
regenerate a new Commit bundle in the new epoch.

The GroupInfo logically contains the following fields, nested as shown:

- GroupInfo
  - GroupContext
    - MLS version
    - ciphersuite
    - group ID
    - epoch
    - tree hash
    - transcript hash
    - (GroupContext) extensions
      - `required_capabilities`
      - `external_sender`s
      - `room_policy` (proposed extension)
  - (GroupInfo) extensions
    - `external_pub`
    - `ratchet_tree` (optional)
  - confirmation_tag
  - signer
  - signature

The size of a GroupInfo without a `ratchet_tree` extension is relatively modest
and need not vary with the number of clients in the group. Even if it includes
a somewhat complicated `room_policy` extension
(proposed in {{!I-D.mahy-mls-room-policy-ext}}), the size of that extension typically
would not increase for each member of the group. By contrast, the `ratchet_tree`
extension grows linearly with the number of clients in the group. Depending on
the size of credentials used, it can easily grow to several megabytes for a large group.
Therefore, the MIMI transport protocol should require that the `ratchet_tree` is
conveyed outside of the GroupInfo.

Instead of sending the `ratchet_tree` directly, we can include a `ratchet_tree` container
object in the Commit bundle. This could have options for various ways to convey
a `ratchet_tree`: a complete ratchet tree, a compressed version, a delta from or patch to the
previous epoch's tree, or a reference to a separate service. This provides future
proofing. We will need to also come to consensus on a mandatory-to-implement option.
Including the entire tree is wasteful and likely to have real performance impacts for
large groups; requiring that the provider compute the ratchet tree itself is likely a
non-starter for providers with a high-volume of traffic.

Note: Sending a Proposal does NOT require exclusive access to the epoch.
Assume a group contains clients A, B, and C. A proposal from A to remove B
and a proposal from B to remove A are independently valid proposals, but
together they would be incompatible and the combination would be invalid.

## Requesting the GroupInfo

Requesting the GroupInfo from the owning provider requires the owning provider
to be online, and requires that the requesting user has authorization to
receive this (privacy sensitive) information. This request is expected to be
followed immediately by a Commit bundle, except in cases like a sudden
network loss or client crash.

The owning provider should make the GroupInfo available to a single requestor
for a short amount of time (ex: from a handful of seconds to 30 seconds). The
provider might put other restrictions in place to preventing abuse of this
primitive, for example denying repeated GroupInfo requests in the absense of
corresponding Commits.

The provider requesting the GroupInfo needs to be able to present a joining
authorization passcode if one was included by the joining user. For more discussion
of joining links and codes, see {{join-codes}}.

# Provisional events / messages

Ordinary messages and a handful of other events need to be sent *provisionally*
from non-owning providers to the owning provider for any room, as the owning
provider still has the ability to refuse these messages or events.

MLS application messages which are sent in the current epoch, could be sent
by a client Alice to provider A, while simultaneously a Commit is arriving
at owning provider B.
Provider A should still deliver the messages to provider B, which should accept
messages from authorized members, even if the message is from a slightly older
epoch. However, if the Commit removed Alice from the group, Alice's message
should be rejected. In practice, most provisional messages sent in good faith
will be accepted. Therefore the protocol should send these messages
efficiently (in bulk) between providers and communicate rejections
asynchronously.

MLS Proposals should also be sent provisionally. If Alice sends a Proposal, Alice
knows this Proposal was accepted or rejected only after receiving the Commit for
the next epoch. The next Commit will either include the Proposal (indicating acceptance)
or not (indicating rejection or incompatibility).

# Fanout of events / messages

Messages and events which have been accepted by the owning
provider then need to be fanned-out to the relevant providers,
which in turn fan-out the messages and events to their relevant clients.
Below are a list of events that would be fanned out:

- application messages
- commits
- proposals
- welcomes
- room policy changes
  - characteristics
  - who is pre-authorized for each role
  - who has which role
- room destroyed
- who has voice

In commercial messaging deployments, fanout messages among two providers
could result in millions of messages per minute. Therefore it is crucial
that fanout messages can be communicated and acknowledged in bulk.

At their discretion (according to their policies), providers might also
choose to inform other providers when users are deleted. For example, two
closely federated enterprise IM systems might do this for all user deletions
where the deleted user is present on a room owned by the other provider.
In another model, large consumer providers might inform each other when a
user was deleted due to spam or abuse.

Regarding the ordering requirements of fanned-out messages:
 
- Every message/event needs to be delivered.
- It is desirable that events/messages within a single group are no more
  than a few hundred messages out-of-order, so the client does not advance
  its decryption generation counter too far.
- Application messages do not need to be delivered strictly in order.
- Commits within the same group must be in-order relative to each other
- Proposal referenced in a Commit must be delivered before or with the Commit

Note: Clients might ask their own provider for MLS Commits, Proposals,
Welcomes, and consent request/grant/reject information before other information,
in order to display most recent messages without a long delay.

Delivering the most recent messages first seems desirable from a user interface
perspective, however there are constraints; in order to decrypt messages,
the entire sequence of Commits
leading up to the current epoch must be processed in order. Clients typically only
save the keying material for a small number of epochs (two to five). Keeping more
epochs increases memory/storage consumption and reduces security, but allows
clients to decrypt messages, while lots of joining and leaving is being
processed. 

Within each epoch, clients use a "generation" counter to calculate the
key to decrypt a specific message *within* an epoch. Clients can run the
generation counter forward and save keys for pending messages to decrypt a
message whichs arrive ahead of turn, but clients can typically store from
several hundred to a small number of thousands of these keys per group. Setting
this to a very large number exposes clients to a very simple DoS attack where
the client processes millions of forward ratchet operations to try to decrypt
a malicious message.

# Directed Async requests

Some primitives are requests that are neither sent to every member of a room,
not do they necessarily result in a timely response. They are not a natural
fit for HTTP REST calls for example.

## Consent messages

For the consent primitive, the sender of a consent request should receive an
acknowledgement that the request was received by the provider of the target
user. For privacy reasons, the requestor should not know if the target user
received or viewed the request. The original requestor will obviously find out
about a consent accept, but a consent reject or block is typically not
communicated to the rejected/blocked user (again for privacy reasons).

The consent primitive needs to include the following:

- the specific operation (a request for consent, a grant of consent, or a
  rejection/revocation of consent);
- the user ID of the user requesting consent;
- the user ID of the target user; and
- optionally, the room ID for which the consent was requested.

If the consent primitive does not specify a room, it implies consent for
any room. This is a common model for systems that use connection requests.
Once a user accepts a connection request, either party is consenting to add
the other to any number of rooms. In other systems, consent for Alice to
add Bob to a soccer fans room does not imply that Alice has permission to
add Bob to a timeshare presentations room. Both models are common, so MIMI
should be able to support both models.

Note that a user could reject consent for all rooms from a user even if there
was a consent request for a *specific* room, or even *no consent request*.

## Knock messages

A client sending a knock request expects to receive some indication
that its knock was received.  However the ultimate goal of the knock
(to be added to a room) may take 2 seconds or 2 yearsm or may never
result in a response.

## Reporting abuse/spam

Likewise, reporting spam or abuse needs to result in some acknowledgement of
the report itself. However a delete or ban action for a spamming user may never
happen, may happen immediately, or may happens after weeks or months.

## Moderation messages

For completeness, requests for permission to send messages (voice) and the
corresponding grant voice and revoke voice primitives have a similar asynchronous
form of operation. However, in a moderated room it is expected for each member to
know if they have voice or not. Note that moderation is currently out-of-scope for
MIMI.

# Other requests

## Search and discovery

This section assumes provider A is searching on provider B. It takes no position on how
a user or providers figures out which provider or providers are queried.

Adding a user to a group, or obtaining consent to contact a user in MIMI requires
discovery of the user ID of the target user. Whether a user can be discovered/
searched depends on the user's provider's policies and the user's configured
preferences. Some users (for example a realtor or salesperson) may choose to be
broadly searchable, while another user may allow only a search on a single
specific field. The MIMI discovery protocol needs to provide a way to search for
a user at a specific provider and should be able to indicate the specific field
or fields that are being provided or searched:

- user handle or nickname
- email address
- phone number
- partial name search
- search in the entire user profile
- any specific field in the OpenID Connect "Standard Claims"
- any specific field in a vCard

In the case of an exact handle search it is likely that only a single
user ID is returned (if at all). However when less specific information is
requested, the search could yield multiple results, even a large number,
which may need to be paginated and/or rate limited. The specific information
provided in the response is also subject to the combination of user and
provider policies.

Note that a provider could use a blanket consent rejection to prevent a user
from being found via search.

## Attached files or assets

Uploading a file to a room in a federated environment can
use one of two models. Either each user uploads the file to their own provider, or
every user uploads any files to the provider owning the room. Any member can
download the files from the provider storing the file. 

The MIMI message transport protocol could easily support both options.

Note that in many enterprise
environments it may not be possible for clients to use a link directly to or from
the provider storing the file. It may need to be proxied through the client's
provider.

## Joining/Invite links and codes {#join-codes}

Many messaging systems have a way to generate a link (sometimes represented as a
QR Code) that is used to identify a room, find a room, or authorize a user to
join a room. There are many variations in the behavior of links when they are
generated. For example, some links merely point to the target room but do not
grant any additional authorization for a joiner. Typically they include both the
address of the room and some authorization. Authorization could apply only to the
first client to join using that code, or to any number of users. It could also be
limited in time, or require an additional out-of-band password/passphrase or
role-based authorization.

How a user generates or discovers such a link is out-of-scope of
MIMI, however the way such a link is used to join a room across providers should
be in scope, as it is a common mode of joining. 
To be useful in an interoperable way, a link needs to embed the room ID (probably
as a URI) and a code.
The MIMI message transfer protocol would then include the code in a request for
GroupInfo for the room. Then the GroupInfo could be used in an external join.

A personal "introduction code" might also provide the user ID of a user, which
could be used to request consent to communicate.

## Current status information

IM systems carry some types of information where only the most recent event
may be needed. This could include (for example):

- typing or composing status for a user
- name of room / subject of room
- user status / display name / nickname
- user presence

Note that presence is explicitly out-of-scope of MIMI. However the MIMI
working group may discover that a message transfer primitive which only
delivers the current status is useful and necessary.


# Security Considerations

Assumptions about authentication, privacy, and consent of individual users
are discussed in the body of this document. The security considerations of
MLS {{!I-D.ietf-mls-protocol}} and the MLS group chat framework
{{!I-D.mahy-mimi-group-chat}} also apply.

TODO: Discussion of the end-to-middle problem.

# IANA Considerations

This document has no IANA actions.


--- back

