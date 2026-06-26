---
title: "PCEP Operational Clarification"
abbrev: "PCEP-OPERATIONAL"
category: info

docname: draft-ietf-pce-operational-latest
submissiontype: IETF
number:
date: {DATE}
consensus: true
v: 3
area: ""
workgroup: "Path Computation Element"

author:
 -
    fullname: Mike Koldychev
    organization: Ciena Corporation
    email: mkoldych@proton.me
 -
    fullname: Siva Sivabalan
    organization: Ciena Corporation
    email: ssivabal@ciena.com
 -
    fullname: Shuping Peng
    organization: Huawei Technologies
    email: pengshuping@huawei.com
 -
    fullname: Diego Achaval
    organization: Nokia
    email: diego.achaval@nokia.com
 -
    fullname: Hari Kotni
    organization: Juniper Networks, Inc
    email: hkotni@juniper.net
 -
    fullname: Andrew Stone
    organization: Nokia
    email: andrew.stone@nokia.com

normative:
    RFC8231:
    RFC8697:
    RFC3209:
    RFC5440:
    RFC8281:
    RFC4655:

informative:


--- abstract

This document clarifies certain operational behavior aspects of
the Path Computation Element Communication Protocol (PCEP). The
content of this document has been compiled based on several interop
exercises. This document does not make any updates or revisions to
any PCEP specifications, instead it provides additional information
to aid in the interpretation of those documents.


--- middle

# Introduction

Due to different interpretations of the PCEP standards
[RFC5440], [RFC8231], [RFC8281], [RFC8697], and [RFC3209], it was
found that implementations often had to adjust their behavior in
order to interoperate. The current document serves to clarify
certain aspects of PCEP to make it easier to produce interoperable
implementations of PCEP.

This document does not make any updates or revisions to any PCEP
specifications. It provides additional informational guidance to aid
in the consistent interpretation of those documents across
implementations.

# Terminology

The following terminology is used in this document:

PCC:  Path Computation Client [RFC4655].

PCE:  Path Computation Element [RFC4655].

PCEP:  Path Computation Element Communication Protocol [RFC5440].

MBB:  Make-Before-Break.  A procedure during which the head-end of a
traffic-engineered path wishes to move traffic to a new path
without losing any traffic, by first "making" a new path and then
"breaking" the old path (see [RFC3209]).

Association parameters:  As described in [RFC8697], the combination
of the mandatory fields Association type, Association ID and
Association Source in the ASSOCIATION object uniquely identify the
association group.  If the optional TLVs - Global Association
Source or Extended Association ID are included, then they are
included in combination with mandatory fields to uniquely identify
the association group.

Association information:  As described in [RFC8697], the ASSOCIATION
object could also include other optional TLVs based on the
association types, that provides 'information' related to the
association type.

ERO: Explicit Route Object is the path of the LSP encoded into a PCEP object.
This document distinguishes three cases for the ERO object, because an
absent ERO and an empty ERO are not equivalent:

* An absent ERO object, i.e., one not included in the message at all.
* An empty ERO object, i.e., one that is present but carries no
  subobjects, is represented with the notation "ERO={}".
* An ERO object containing a given sequence of subobjects is
  represented as "ERO={A}".

PCRPT-LSP-DB: PCEP Reported Label Switched Path Database. A logical datastore
that captures the reported state of Label Switched Paths (LSPs), organized as
Tunnels and their LSPs. Each PCEP speaker maintains
its own local copy, populated from PCRpt messages. On a PCE, the PCRPT-LSP-DB
holds the state reported by all PCCs that report to that PCE. On a PCC, it holds
the state for the LSPs for which that PCC is the head-end. This term is not
defined in the PCE architecture; however, it is used in this document to describe
how a PCEP speaker maintains LSP-related state reported via PCRpt messages.

EXTENDED-LSP-DB: Extended Label Switched Path Database. A single, optional,
implementation-specific logical datastore that holds entries for multiple LSPs,
keyed using the same identifiers as the PCRPT-LSP-DB. It is not required by a
conforming PCEP implementation. It is typically maintained on the PCE to hold
additional attributes such as desired (intended) state, telemetry data, operator
configuration, and other information not defined within IETF PCE working group
documents; a PCC may maintain an equivalent datastore where useful. This term is
not defined in the PCE architecture and is used in this document only to refer to
this conceptual datastore.

PCE-ASSO-DB: PCEP Association Database. A logical datastore that captures
PCEP Association membership, populated by PCRpt messages and/or by configuration
on the PCE. This term is not defined in the PCE architecture; it is used in this
document to describe how a PCE maintains Association state. See the PCEP
Association Database section.

PLSP-ID (Path LSP Identifier): Introduced in [RFC8231]. A unique identifier used in PCEP to
distinguish a specific LSP between a PCC and a PCE which is constant for the lifetime of
a PCEP session.

# PCEP LSP Database

This document uses the concept of the PCRPT-LSP-DB, a database of actual LSP
state in the network, to illustrate the internal state of PCEP speakers in
response to PCRpt messages.

Note that the term "LSP", which stands for "Label Switched Path", if
taken too literally, would restrict the discussion to the MPLS dataplane
only. In this document, the term "LSP" is applied to non-MPLS paths as well,
to avoid renaming the term.

## Structure

The distinction between Tunnels and LSPs in the PCRPT-LSP-DB is essential for accurately
modeling state information in PCEP, including procedures such as make-before-break,
and for maintaining consistent state management across PCEP speakers.

The PCRPT-LSP-DB contains two types of information: LSPs and Tunnels. An LSP is identified
by the LSP-IDENTIFIERS TLV, while a Tunnel is identified by the PLSP-ID in the LSP
object and/or the SYMBOLIC-NAME (see [RFC8231]).

A Tunnel in PCEP does not necessarily map one-to-one to a single tunnel
construct on the router. A tunnel into which a client inserts traffic may
be supported by one or more underlying transport paths.
For example, the working and protect paths of
a single tunnel interface on the router are represented in PCEP as two
different Tunnels with different PLSP-IDs.

An LSP can be viewed as an instance of a Tunnel. In steady state,
a Tunnel typically has a single LSP; however, during make-before-break
procedures (see [RFC3209]), multiple LSPs may exist simultaneously to represent
both the new and old instances during the transition.

## Synchronization

Since the PCRPT-LSP-DB contains PCEP-specific information such as PLSP-IDs,
which remain constant for the duration of a PCEP session, both the PCE and
PCC maintain their own local copies of the PCRPT-LSP-DB.
The PCE PCRPT-LSP-DB is only modified by PCRpt messages, no other PCEP message
may modify the PCE PCRPT-LSP-DB.  The PCC PCRPT-LSP-DB is built from actual
forwarding state that PCC has installed.  PCC uses PCRpt messages to
synchronize its local PCRPT-LSP-DB to the PCE.

The PCE is intended to act based on the most recent state available in the PCRPT-LSP-DB,
which represents the reported state of Label Switched Paths (LSPs) in the network. This
does not preclude the PCE from leveraging information obtained outside the PCRPT-LSP-DB.
For example, implementation-specific mechanisms may be used to collect traffic statistics
that can be considered during path computation. Such additional information, including
desired (intended) state, telemetry, and operator configuration, is not part of the
PCRPT-LSP-DB itself and may instead be held in the separate, optional EXTENDED-LSP-DB
described in the Terminology section.

The PCRPT-LSP-DB reflects the reported state of LSPs and does not imply alignment with the
desired state. It also does not necessarily reflect the live state of the network, because
there are two convergence windows during which the databases can lag the network:

* The network state has changed, but the PCC has not yet updated its own PCRPT-LSP-DB.
* The PCC has updated its PCRPT-LSP-DB, but the PCRpt carrying that change has not yet
  been received and processed by the PCE.

For example, in the case of a PCE-initiated LSP, a change in the LSP configuration made by
an operator represents a modification to the desired state. However, the actual state does
not change until the PCE sends a PCInit or PCUpd message to the PCC. Upon receipt of such a
message, the PCC may act on the request, update its local PCRPT-LSP-DB, and generate a PCRpt
message to inform the PCE of the change. The PCE then updates its own PCRPT-LSP-DB accordingly.
Once this exchange is complete, the PCRPT-LSP-DBs on both the PCC and the PCE are synchronized.

## Make before Break (MBB)

Due to variations in how different implementations interpret or handle MBB
procedures, sometimes resulting in incorrect message processing or
misinterpretation, the following section provides illustrative examples to
clarify expected MBB behavior.

### Successful MBB

Below is an example of performing MBB to switch a Tunnel from one
path to another. The path encoded into the ERO object
is represented as ERO={A} and ERO={B}.

PCC has an existing LSP in UP state, with LSP-ID=2.  PCC sends
PCRpt(R-FLAG=0, PLSP-ID=100, LSP-ID=2, ERO={A}, OPER-FLAG=UP).

~~~
+---------------------------------------------------------------+
| TUNNEL          | LSP                                         |
+-----------------+---------------------------------------------+
| PLSP-ID=100     | LSP-ID=2, ERO={A}, OPER=UP                  |
+---------------------------------------------------------------+

                Figure 1: Content of PCRPT-LSP-DB
~~~

The resulting content of the PCRPT-LSP-DB is shown in Figure 1.

PCC initiates the MBB procedure by creating a new LSP with LSP-ID=3.
It does not matter what triggered the creation of the new LSP, it
could have been due to a new path received via PCUpd (if the given
Tunnel is delegated), or it could have been local computation on the
PCC (if the Tunnel is locally computed on the PCC), or it could have
been a change in configuration on the PCC (if the Tunnel's path is
explicitly configured on the PCC).  It is important to emphasize that
the procedure for updating the PCRPT-LSP-DB is common, regardless of the
trigger that caused the change.

PCC sends PCRpt(R-FLAG=0, PLSP-ID=100, LSP-ID=3, ERO={B}, OPER-
FLAG=UP).

~~~
+---------------------------------------------------------------+
| TUNNEL          | LSP                                         |
+-----------------+---------------------------------------------+
| PLSP-ID=100     | LSP-ID=2, ERO={A}, OPER=UP                  |
|                 | LSP-ID=3, ERO={B}, OPER=UP                  |
+---------------------------------------------------------------+

                Figure 2: Content of PCRPT-LSP-DB
~~~

Both LSPs now exist simultaneously, as shown in Figure 2.

After traffic has successfully switched to the new LSP, the PCC
cleans up the old LSP.  PCC sends PCRpt(R-FLAG=1, PLSP-ID=100, LSP-
ID=2).

~~~
+---------------------------------------------------------------+
| TUNNEL          | LSP                                         |
+-----------------+---------------------------------------------+
| PLSP-ID=100     | LSP-ID=3, ERO={B}, OPER=UP                  |
+---------------------------------------------------------------+

                Figure 3: Content of PCRPT-LSP-DB
~~~

Only the new LSP remains, as shown in Figure 3.

### Aborted MBB

The MBB process can abort when the newly created LSP is destroyed
before it is installed as traffic carrying.  This scenario is
described below.

PCC has an existing LSP in UP state, with LSP-ID=2.  PCC sends
PCRpt(R-FLAG=0, OPER-FLAG=UP, PLSP-ID=100, LSP-ID=2).

~~~
+---------------------------------------------------------------+
| TUNNEL          | LSP                                         |
+-----------------+---------------------------------------------+
| PLSP-ID=100     | LSP-ID=2, OPER=UP                           |
+---------------------------------------------------------------+

                Figure 4: Content of PCRPT-LSP-DB
~~~

The initial content of the PCRPT-LSP-DB is shown in Figure 4.

MBB procedure is initiated, a new LSP is created with LSP-ID=3.  LSP
is currently being established, so its oper state is DOWN.  PCC sends
PCRpt(R-FLAG=0, OPER-FLAG=DOWN, PLSP-ID=100, LSP-ID=3).

~~~
+---------------------------------------------------------------+
| TUNNEL          | LSP                                         |
+-----------------+---------------------------------------------+
| PLSP-ID=100     | LSP-ID=2, OPER=UP                           |
|                 | LSP-ID=3, OPER=DOWN                         |
+---------------------------------------------------------------+

                Figure 5: Content of PCRPT-LSP-DB
~~~

The new LSP is being established while the old LSP remains UP, as shown
in Figure 5.

MBB procedure is aborted.  PCC sends PCRpt(R-FLAG=1, PLSP-ID=100,
LSP-ID=3).

~~~
+---------------------------------------------------------------+
| TUNNEL          | LSP                                         |
+-----------------+---------------------------------------------+
| PLSP-ID=100     | LSP-ID=2, OPER=UP                           |
+---------------------------------------------------------------+

                Figure 6: Content of PCRPT-LSP-DB
~~~

The new LSP has been removed and only the original LSP remains, as shown
in Figure 6.

# PCEP Association Database

A PCEP Association is the instantiation of a group containing at least one LSP.

The PCE-ASSO-DB is populated by PCRpt messages and/or via
configuration on the PCE itself. An Association is identified by the
Association Parameters. As the Association Parameters contain many fields,
all fields are grouped into a single value for convenience.
The notation ASSO_PARAM=A and ASSO_PARAM=B is used to refer to
different PCEP Associations: A and B, respectively.

The message parameter notation used in the examples below (for example,
R-FLAG, OPER-FLAG, ASSO_R_FLAG, and ASSO_PARAM) refers to the
corresponding flags and fields of the LSP and ASSOCIATION objects
defined in [RFC8231] and [RFC8697].

## LSPs in same Association

The following example illustrates how LSPs join the same Association.

The PCC creates the first LSP.  The PCC sends PCRpt(R-FLAG=0, PLSP-ID=100,
LSP-ID=1, ASSO_PARAM=A, ASSO_R_FLAG=0).

~~~
+---------------------------------------------------------------+
| ASSO            | LSP                                         |
+-----------------+---------------------------------------------+
| ASSO_PARAM=A    | PLSP-ID=100, LSP-ID=1                       |
+---------------------------------------------------------------+

                Figure 7: Content of PCE-ASSO-DB
~~~

The resulting content of the PCE-ASSO-DB is shown in Figure 7.

The PCC creates the second LSP.  The PCC sends PCRpt(R-FLAG=0, PLSP-ID=200,
LSP-ID=1, ASSO_PARAM=A, ASSO_R_FLAG=0).

~~~
+---------------------------------------------------------------+
| ASSO            | LSP                                         |
+-----------------+---------------------------------------------+
| ASSO_PARAM=A    | PLSP-ID=100, LSP-ID=1                       |
|                 | PLSP-ID=200, LSP-ID=1                       |
+---------------------------------------------------------------+

                Figure 8: Content of PCE-ASSO-DB
~~~

Both LSPs are now members of the same Association, as shown in Figure 8.

The PCC updates the first LSP. As [RFC8697] indicates, subsequent PCRpt
should include only the associations that are being modified or removed.
Therefore it is optional whether to send the
ASSOCIATION object in this PCRpt, since the LSP is already in the
Association.  The PCC sends PCRpt(R-FLAG=0, PLSP-ID=100, LSP-ID=1).  The
content of the PCE-ASSO-DB is unchanged.  Note that the PCC sends the
ASSOCIATION OBJECT in the first PCRpt during SYNC state, even if it
has already issued a PCRpt with the association object sometime in
the past with this PCE.  The synchronization steps outlined in
[RFC8697] are to be followed.

~~~
+---------------------------------------------------------------+
| ASSO            | LSP                                         |
+-----------------+---------------------------------------------+
| ASSO_PARAM=A    | PLSP-ID=100, LSP-ID=1                       |
|                 | PLSP-ID=200, LSP-ID=1                       |
+---------------------------------------------------------------+

                Figure 9: Content of PCE-ASSO-DB
~~~

The content of the PCE-ASSO-DB is unchanged, as shown in Figure 9.

The PCC decides to delete the second LSP.  The PCC sends PCRpt(R-FLAG=1,
PLSP-ID=200, LSP-ID=1).

~~~
+---------------------------------------------------------------+
| ASSO            | LSP                                         |
+-----------------+---------------------------------------------+
| ASSO_PARAM=A    | PLSP-ID=100, LSP-ID=1                       |
+---------------------------------------------------------------+

                Figure 10: Content of PCE-ASSO-DB
~~~

Only the first LSP remains in the Association, as shown in Figure 10.

The PCC decides to remove the first LSP from the Association, but not
delete the LSP itself.  The PCC sends PCRpt(R-FLAG=0, PLSP-ID=100, LSP-
ID=1, ASSO_PARAM=A, ASSO_R_FLAG=1).  The PCE-ASSO-DB is now empty.

~~~
+---------------------------------------------------------------+
| ASSO            | LSP                                         |
+-----------------+---------------------------------------------+
| -               |                                             |
+---------------------------------------------------------------+

                Figure 11: Content of PCE-ASSO-DB
~~~

The Association is now empty, as shown in Figure 11.

## Switch Association during MBB

The following example illustrates how a Tunnel goes through MBB
and switches from Association A to Association B.

Each new LSP (identified by the LSP-ID) does not inherit the
Association membership of any previous LSPs within the same Tunnel.
This is so that a Tunnel can have two LSPs that are in different
Associations, this may be done when switching from one Association to
another.

The PCC creates the first LSP.  The PCC sends PCRpt(R-FLAG=0, PLSP-ID=100,
LSP-ID=1, ASSO_PARAM=A, ASSO_R_FLAG=0).

~~~
+---------------------------------------------------------------+
| ASSO            | LSP                                         |
+-----------------+---------------------------------------------+
| ASSO_PARAM=A    | PLSP-ID=100, LSP-ID=1                       |
+---------------------------------------------------------------+

                Figure 12: Content of PCE-ASSO-DB
~~~

The resulting content of the PCE-ASSO-DB is shown in Figure 12.

The PCC creates the MBB LSP in a different Association.  The PCC sends
PCRpt(R-FLAG=0, PLSP-ID=100, LSP-ID=2, ASSO_PARAM=B, ASSO_R_FLAG=0).

~~~
+---------------------------------------------------------------+
| ASSO            | LSP                                         |
+-----------------+---------------------------------------------+
| ASSO_PARAM=A    | PLSP-ID=100, LSP-ID=1                       |
+---------------------------------------------------------------+
| ASSO_PARAM=B    | PLSP-ID=100, LSP-ID=2                       |
+---------------------------------------------------------------+

                Figure 13: Content of PCE-ASSO-DB
~~~

The two LSPs of the Tunnel are now in different Associations, as shown
in Figure 13.

The PCC deletes the old LSP.  The PCC sends PCRpt(R-FLAG=1, PLSP-ID=100, LSP-
ID=1).

~~~
+---------------------------------------------------------------+
| ASSO            | LSP                                         |
+-----------------+---------------------------------------------+
| ASSO_PARAM=B    | PLSP-ID=100, LSP-ID=2                       |
+---------------------------------------------------------------+

                Figure 14: Content of PCE-ASSO-DB
~~~

Only the new LSP in Association B remains, as shown in Figure 14.

# Computation Constraints

As well as reporting any state change in the network on a PCRpt message,
a PCC may also change the parameters of a delegated LSP. For example, it may remove or
modify the computation constraints that it wishes the PCE to apply as it
computes any updated paths in the future. For any PCEP object that
specifies a path computation constraint and that does not have a defined
explicit removal flag, the absence of that entire object on a repeat or
follow-up PCRpt message indicates removal of the constraint previously
specified by that object. For example,
suppose the first PCRpt contains an LSPA object with some affinity constraints.
Then if a subsequent PCRpt does not contain an LSPA object, then this means that
the previously specified affinity constraints do not apply anymore.
Same applies to all PCEP objects, like METRIC, BANDWIDTH, etc., which
do not have an explicit flag for removal.  This simply ensures that
it is possible to remove a constraint without using an explicit
removal flag.


# Overloaded PCE

[RFC5440] defines the concept of an Overloaded PCE and explains how
a PCE may signal to a PCC that it is congested, instructing that
"no requests should be sent to that PCE until the overload state is cleared."

In the case of an overloaded PCE, a PCC implementation could choose to wait for the PCE
to no longer be overloaded or instead send a PCReq to a backup, non-overloaded PCE instead.

[RFC8231] builds upon [RFC5440] by introducing the concept of a
Stateful PCE, which allows delegation of LSP control to a single PCE.
However, it does not address the case of an overloaded PCE. According
to [RFC8231], a PCE maintains delegation until it is revoked by the PCC
or returned back to PCC by the PCE. The PCC may revoke delegation and re-assign
it to another PCE.

As a result, a PCE in an overload state still retains LSP delegation.
For PCC-initiated LSPs, the PCC may revoke delegation from the overloaded
PCE and maintain delegation for itself or delegate it to another PCE. For
PCE-initiated LSPs, since the PCC cannot revoke delegation as per [RFC8281],
the overloaded PCE may return the delegation to the PCC.

## Reports toward an Overloaded PCE

The guidance in this section clarifies behavior for reports and does not
modify [RFC5440]. Any change that would require a PCC to stop sending reports
toward an overloaded PCE would be an update to [RFC5440] and is out of scope for
this document.

There is a trade-off in how a PCC handles PCRpt messages while a PCE
signals that it is overloaded. Fully reprocessing a flood of PCRpt messages consumes
CPU and memory on the PCE, and so continuing to send reports could prolong or worsen
the overload condition.However, the longer the PCE is without the latest reported state,
the more likely it is to compute paths using stale information.

What it means for a PCE to be overloaded is implementation specific. For example, an
implementation may use separate resources for ingesting reports and for
performing computation, in which case it may be overloaded for
computation while still able to process reports. A PCE may also wish to
remain synchronized with the network without yet participating in
active computation, for example while a new instance is being
bootstrapped.

Given these trade offs, the following describes the behavior a PCC may
adopt toward an overloaded PCE. Sending reports as usual is the
baseline interoperable behavior and throttling reports is an optional
behavior:

* Sending reports as usual: the PCC continues to send PCRpt messages so
  that the PCRPT-LSP-DB on the PCE remains synchronized. This is the
  baseline behavior and requires no change to existing implementations.

* Throttling reports: the PCC may make a local decision
  about whether to immediately send a PCRpt message to the PCE or to
  hold that message until the PCE indicates it is no longer overloaded.
  This decision may depend on how significant the change to the LSP is,
  how long the message has been held, and how many other messages are
  held for the same reason. In all cases the PCC should log the
  situation, applying thresholding to the rate of generation of log
  events, so that an operator can determine what is happening.

Stopping reports entirely while the PCE is overloaded is not
recommended, because it risks the PCRPT-LSP-DB on the PCE going out of
sync with the network for the full duration of the overload condition.

The decision to throttle cannot depend on local policy alone, because
some reports remain significant regardless of policy. At a minimum, the
following reports should not be throttled:

* A PCRpt sent in response to a PCUpd, since a PCE may still take
  actions on a delegated LSP while marked overloaded.
* PCRpt messages sent as part of state synchronization during session
  establishment.

A PCC may throttle other less significant reports such as frequent or
redundant updates, for example local protection availability flags.

As an alternative to per-report throttling, a PCE under overload
may selectively close one or more PCEP sessions, choosing which
sessions to prioritize. When the overload condition clears, the
sessions are re-established and standard state synchronization
[RFC8231] brings the PCE back in sync with the latest state. This
avoids the complexity of deciding, on a per-report basis, which reports
to hold or drop.

# Security Considerations

This document does not define any new protocol mechanisms and does not
change the PCEP messages or objects defined in [RFC5440], [RFC8231],
[RFC8281], or [RFC8697]. The security considerations of those documents
therefore continue to apply.

The clarifications in this document aim to reduce the risk of
interoperability problems that arise when implementations interpret
PCEP behavior differently. Such divergence can have security relevant
consequences such as inconsistent handling of reported state can cause a PCE
to compute or install paths based on an inaccurate view of the network,
and the absence of an agreed behavior toward an overloaded PCE can be
exploited to amplify a denial-of-service condition by sustaining a flood
of reports. Conversely, holding or throttling PCRpt messages toward an
overloaded PCE, as described in the Overloaded PCE section, can delay
the visibility of state changes, including those resulting from
misconfiguration or compromise. The logging recommended in that section
helps an operator detect such situations. Selective closing of PCEP
sessions during overload is an operational control that should be
governed by local policy.

# Manageability Considerations

The logical datastores described in this document (the PCRPT-LSP-DB,
the optional EXTENDED-LSP-DB, and the PCE-ASSO-DB) are useful points of
operational visibility. Where an implementation exposes them, they map
naturally onto state data models, for example YANG. An operator
benefits from being able to read these datastores in order to
understand the state held by a PCEP speaker.

Because the PCRPT-LSP-DB on a PCC and on a PCE can lag one another
during the convergence windows described in the Synchronization
section, an operator may need to verify that the two are aligned. This
can be done, for example, by comparing the set of reported LSPs and
their last-reported state, or by observing the state synchronization
procedures of [RFC8231].

As described in the Overloaded PCE section, a PCC that throttles or
holds PCRpt messages toward an overloaded PCE should log that it is
doing so, applying thresholding to the rate of log generation, so that
an operator can determine what is happening.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

The authors would like to thank Adrian Farrel for a thorough review on this document
and Tom Petch and Boris Hassanov for useful review and feedback.

# Contributors
{:numbered="false"}

~~~
Dhruv Dhody
Huawei Technologies
Email: dhruv.ietf@gmail.com

Samuel Sidor
Cisco Systems
Email: ssidor@cisco.com

Mahendra Singh Negi
RtBrick Inc
Email: mahend.ietf@gmail.com
~~~
