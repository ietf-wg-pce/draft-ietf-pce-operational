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
    RFC2119:
    RFC8174:
    RFC8231:
    RFC8697:
    RFC3209:

informative:


--- abstract

This document clarifies certain operational behavior aspects of
the PCEP protocol.  The content of this document has been compiled
based on several interop exercises.


--- middle

# Introduction

Due to different interpretations of PCEP standards, it was found that
implementations often had to adjust their behavior in order to
interoperate.  The current document serves to clarify certain aspects
of PCEP to make it easier to produce interoperable implementations of
PCEP.

# Requirements Language

{::boilerplate bcp14-tagged}

# Terminology

The following terminologies are used in this document:

PCC:  Path Computation Client.  Any client application requesting a
path computation to be performed by a Path Computation Element.

PCE:  Path Computation Element.  An entity (component, application,
or network node) that is capable of computing a network path or
route based on a network graph and applying computational
constraints.

PCEP:  Path Computation Element Protocol.

MBB:  Make-Before-Break.  A procedure during which the head-end of a
traffic-engineered path wishes to move traffic to a new path
without losing any traffic, by first "making" a new path and then
"breaking" the old path.

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
In this document, an empty ERO object, i.e., without any subobjects,
is represented with notation "ERO={}". An ERO object containing a given
sequence of subobjects is represented as "ERO={A}".

LSP-DB: Label Switched Path Database. A datastore that captures the state
information of Label Switched Paths (LSPs) within a PCEP speaker.
Not defined in the PCE Architecture, however this document uses the LSP-DB
to illustrate how a PCEP speaker manages LSP-related information.

PLSP-ID (Path LSP Identifier): Introduced in [RFC8231]. A unique identifier used in PCEP to
distinguish a specific LSP between a PCC and a PCE which is constant for the lifetime of
a PCEP session.

# PCEP LSP Database

This document uses the concept of the LSP-DB, a database of actual LSP
state in the network, to illustrate the internal state of PCEP speakers in
response to various PCEP messages.

Note that the term "LSP", which stands for "Label Switched Path", if
taken too literally, would restrict the discussion to the MPLS dataplane
only. In this document, the term "LSP" is applied to non-MPLS paths as well,
to avoid renaming the term. Alternatively, the term "LSP" could be replaced with "Instance".

## Structure

The distinction between Tunnels and LSPs in the LSP-DB is essential for accurately
modeling state information in PCEP, including procedures such as make-before-break,
and for maintaining consistent state management across PCEP speakers.

The LSP-DB contains two types of information: LSPs and Tunnels. An LSP is identified
by the LSP-IDENTIFIERS TLV, while a Tunnel is identified by the PLSP-ID in the LSP
object and/or the SYMBOLIC-NAME (see [RFC8231]).

A Tunnel may or may not correspond to an actual tunnel on the router. For
example, working and protect paths can be implemented as a single
tunnel interface, but in PCEP, these are represented as two different
Tunnels with different PLSP-IDs.

An LSP can be viewed as an instance of a Tunnel. In steady state,
a Tunnel typically has a single LSP; however, during make-before-break procedures (see [RFC3209]),
multiple LSPs may exist simultaneously to represent both the new and old instances during the transition.

## Synchronization

Since the LSP-DB contains PCEP-specific information such as PLSP-IDs,
which remain constant for the duration of a PCEP session, both the PCE and
PCC maintain their own local copies of the LSP-DB.
The PCE LSP-DB is only modified by PCRpt messages, no other PCEP message
may modify the PCE LSP-DB.  The PCC LSP-DB is built from actual
forwarding state that PCC has installed.  PCC uses PCRpt messages to
synchronize its local LSP-DB to the PCE.

The PCE is intended to act on the latest state of the PCE LSP DB.  Note,
this does not mean that the PCE cannot use information from
outside of LSP-DB.  For example, the PCE can use other mechanisms to
collect traffic statistics and use them in the computation.  However,
these traffic statistics are not part of the LSP-DB, but only
reference it.

The LSP-DB on both the PCC and the PCE primarily stores the actual state
in the network. An implementation may choose to store additional information such as desired state,
however the LSP-DB still contains the live view of actual state and not infer the actual state
is an immediate reflection of the desired state.
For example, consider the case of PCE Initiated LSP, configured on the PCE.  When
the operator modifies the configuration of this LSP, that is a change
in desired state.  The actual state has not yet changed, so LSP-DB is
not modified yet.  The LSP-DB is only modified after the PCE sends
PCInit/PCUpd message to the PCC and the PCC decides to act on that
message.  When the PCC acts on a message from a PCE, it would update
its own PCC LSP DB and send a PCRpt to the PCE(s) to synchronize the
change.  When the PCE receives the PCRpt msg, it updates its own PCE
LSP DB.  After this, the PCC LSP-DB and PCE LSP-DB are in sync.

## Make before Break (MBB)

Due to variations in how different implementations interpret or handle MBB
procedures—sometimes resulting in incorrect message processing or misinterpretation—the
following section provides illustrative examples to clarify expected MBB behavior.

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

                Figure 1: Content of LSP DB
~~~

PCC initiates the MBB procedure by creating a new LSP with LSP-ID=3.
It does not matter what triggered the creation of the new LSP, it
could have been due to a new path received via PCUpd (if the given
Tunnel is delegated), or it could have been local computation on the
PCC (if the Tunnel is locally computed on the PCC), or it could have
been a change in configuration on the PCC (if the Tunnel's path is
explicitly configured on the PCC).  It is important to emphasize that
the procedure for updating the LSP-DB is common, regardless of the
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

                Figure 2: Content of LSP DB
~~~

After traffic has successfully switched to the new LSP, the PCC
cleans up the old LSP.  PCC sends PCRpt(R-FLAG=1, PLSP-ID=100, LSP-
ID=2).

~~~
+---------------------------------------------------------------+
| TUNNEL          | LSP                                         |
+-----------------+---------------------------------------------+
| PLSP-ID=100     | LSP-ID=3, ERO={B}, OPER=UP                  |
+---------------------------------------------------------------+

                Figure 3: Content of LSP DB
~~~

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

                Figure 4: Content of LSP DB
~~~

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

                Figure 5: Content of LSP DB
~~~

MBB procedure is aborted.  PCC sends PCRpt(R-FLAG=1, PLSP-ID=100,
LSP-ID=3).

~~~
+---------------------------------------------------------------+
| TUNNEL          | LSP                                         |
+-----------------+---------------------------------------------+
| PLSP-ID=100     | LSP-ID=2, OPER=UP                           |
+---------------------------------------------------------------+

                Figure 6: Content of LSP DB
~~~

# PCEP Association Database

PCEP Association is the instantiate of a group containing at least one LSP.

The PCE ASSO DB is populated by PCRpt messages and/or via
configuration on the PCE itself. An Association is identified by the
Association Parameters. As the Association Parameters contain many fields,
all fields are grouped into a single value for convenience.
The notation ASSO_PARAM=A and ASSO_PARAM=B is used to refer to
different PCEP Associations: A and B, respectively.

## LSPs in same Association

The following example illustrates how LSPs join the same Association.

PCC creates the first LSP.  PCC sends PCRpt(R-FLAG=0, PLSP-ID=100,
LSP-ID=1, ASSO_PARAM=A, ASSO_R_FLAG=0).

~~~
+---------------------------------------------------------------+
| ASSO            | LSP                                         |
+-----------------+---------------------------------------------+
| ASSO_PARAM=A    | PLSP-ID=100, LSP-ID=1                       |
+---------------------------------------------------------------+

                Figure 7: Content of PCE ASSO DB
~~~

PCC creates the second LSP.  PCC sends PCRpt(R-FLAG=0, PLSP-ID=200,
LSP-ID=1, ASSO_PARAM=A, ASSO_R_FLAG=0).

~~~
+---------------------------------------------------------------+
| ASSO            | LSP                                         |
+-----------------+---------------------------------------------+
| ASSO_PARAM=A    | PLSP-ID=100, LSP-ID=1                       |
|                 | PLSP-ID=200, LSP-ID=1                       |
+---------------------------------------------------------------+

                Figure 8: Content of PCE ASSO DB
~~~

PCC updates the first LSP. As [RFC8697] indicates, subsequent PcRpt
should include only the associations that are being modified or removed.
Therefore it is optional as to send the
ASSOCIATION object in this PCRpt, since the LSP is already in the
Association.  PCC sends PCRpt(R-FLAG=0, PLSP-ID=100, LSP-ID=1).  The
content of the PCE ASSO DB is unchanged.  Note that the PCC sends the
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

                Figure 9: Content of PCE ASSO DB
~~~

PCC decides to delete the second LSP.  PCC sends PCRpt(R-FLAG=1,
PLSP-ID=200, LSP-ID=1).

~~~
+---------------------------------------------------------------+
| ASSO            | LSP                                         |
+-----------------+---------------------------------------------+
| ASSO_PARAM=A    | PLSP-ID=100, LSP-ID=1                       |
+---------------------------------------------------------------+

                Figure 10: Content of PCE ASSO DB
~~~

PCC decides to remove the first LSP from the Association, but not
delete the LSP itself.  PCC sends PCRpt(R-FLAG=0, PLSP-ID=100, LSP-
ID=1, ASSO_PARAM=A, ASSO_R_FLAG=1).  The PCE ASSO DB is now empty.

~~~
+---------------------------------------------------------------+
| ASSO            | LSP                                         |
+-----------------+---------------------------------------------+
| -               |                                             |
+---------------------------------------------------------------+

                Figure 11: Content of PCE ASSO DB
~~~

## Switch Association during MBB

The following example illustrates how a Tunnel goes through MBB
and switches from Association A to Association B.

Each new LSP (identified by the LSP-ID) does not inherit the
Association membership of any previous LSPs within the same Tunnel.
This is so that a Tunnel can have two LSPs that are in different
Associations, this may be done when switching from one Association to
another.

PCC creates the first LSP.  PCC sends PCRpt(R-FLAG=0, PLSP-ID=100,
LSP-ID=1, ASSO_PARAM=A, ASSO_R_FLAG=0).

~~~
+---------------------------------------------------------------+
| ASSO            | LSP                                         |
+-----------------+---------------------------------------------+
| ASSO_PARAM=A    | PLSP-ID=100, LSP-ID=1                       |
+---------------------------------------------------------------+

                Figure 12: Content of PCE ASSO DB
~~~

PCC creates the MBB LSP in a different Association.  PCC sends
PCRpt(R-FLAG=0, PLSP-ID=100, LSP-ID=2, ASSO_PARAM=B, ASSO_R_FLAG=0).

~~~
+---------------------------------------------------------------+
| ASSO            | LSP                                         |
+-----------------+---------------------------------------------+
| ASSO_PARAM=A    | PLSP-ID=100, LSP-ID=1                       |
+---------------------------------------------------------------+
| ASSO_PARAM=B    | PLSP-ID=100, LSP-ID=2                       |
+---------------------------------------------------------------+

                Figure 13: Content of PCE ASSO DB
~~~

PCC deletes the old LSP.  PCC sends PCRpt(R-FLAG=1, PLSP-ID=100, LSP-
ID=1).

~~~
+---------------------------------------------------------------+
| ASSO            | LSP                                         |
+-----------------+---------------------------------------------+
| ASSO_PARAM=B    | PLSP-ID=100, LSP-ID=2                       |
+---------------------------------------------------------------+

                Figure 14: Content of PCE ASSO DB
~~~

# Computation Constraints

As well as reporting any state change in the network on a PCRpt message,
a PCC may also change the parameters of a delegated LSP. For example, it may remove or
modify the computation constraints that it wishes the PCE to apply as it
computes any updated paths in the future. For any PCEP object that
specifies a path computation constraint and that does not have a defined
explicit removal flag, the absence of that entire object on a repeat or
follow-up PcRpt message indicates removal of the constraint previously
specified by that object. For example,
suppose the first PcRpt contains an LSPA object with some affinity constraints.
Then if a subsequent PcRpt does not contain an LSPA object, then this means that
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
to no longer be overloaded or instead send a PcReq to a backup, non-overloaded PCE instead.

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

The PCE will continue to send PcRpt messages to PCE even though it may indicate
it is overloaded, otherwise the the LSP-DB on PCE may go out of sync.


# Security Considerations

None at this time.

# Managability Considerations

None at this time.

# IANA Considerations

None at this time.

--- back

# Acknowledgments
{:numbered="false"}

The authors would like to thank Adrian Farrel and Tom Petch for useful review and feedback.
comments.

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
