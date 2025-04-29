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

This document clarifies certain aspects of
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
Source or Extended Association ID are included, then they MUST be
included in combination with mandatory fields to uniquely identify
the association group.

Association information:  As described in [RFC8697], the ASSOCIATION
object could also include other optional TLVs based on the
association types, that provides 'information' related to the
association type.

ERO:  Explicit Route Object is the path of the LSP encoded into a
PCEP object.  To represent an empty ERO object, i.e., without any
subobjects, we use the notation "ERO={}".  To represent an ERO
object containing some given sequence of subobjects, we use the
notation "ERO={A}".

# PCEP LSP Database

We use the concept of the LSP-DB, as a database of actual LSP state
in the network, to illustrate the internal state of PCEP speakers in
response to various PCEP messages.

Note that the term "LSP", which stands for "Label Switched Path", if
taken too literally would restrict our discussion to MPLS dataplane
only.  We take the term "LSP" to apply to non-MPLS paths as well, to
avoid changing the name.  Alternatively, we could rename LSP to
"Instance".

## Structure

LSP-DB contains two types of objects: LSPs and Tunnels.  An LSP is
identified by the LSP-IDENTIFIERS TLV.  A Tunnel is identified by the
PLSP-ID in the LSP object and/or the SYMBOLIC-NAME.  See [RFC8231].

A Tunnel may or may not be an actual tunnel on the router.  For
example, working and protect paths can be implemented as a single
tunnel interface, but in PCEP we would refer to them as two different
Tunnels, because they would have different PLSP-IDs.

An LSP can be thought of as a instance of a Tunnel.  In steady-state,
a Tunnel has only one LSP, but during make-before-break (see
[RFC3209]) it can have multiple LSPs, to represent both new and old
instances that exist simultaneously for a time.

## Synchronization

Both PCE and PCC maintain their separate copies of the LSP-DB.  The
PCE LSP-DB is only modified by PCRpt messages, no other PCEP message
may modify the PCE LSP-DB.  The PCC LSP-DB is built from actual
forwarding state that PCC has installed.  PCC uses PCRpt messages to
synchronize its local LSP-DB to the PCE.

The PCE MUST always act on the latest state of the PCE LSP DB.  Note
that this does not mean that the PCE cannot use information from
outside of LSP-DB.  For example, the PCE can use other mechanisms to
collect traffic statistics and use them in the computation.  However,
these traffic statistics are not part of the LSP-DB, but only
reference it.

The LSP-DB on both the PCC and the PCE only stores the actual state
in the network, it does not store the desired state.  For example,
consider the case of PCE Initiated LSP, configured on the PCE.  When
the operator modifies the configuration of this LSP, that is a change
in desired state.  The actual state has not yet changed, so LSP-DB is
not modified yet.  The LSP-DB is only modified after the PCE sends
PCInit/PCUpd message to the PCC and the PCC decides to act on that
message.  When the PCC acts on a message from a PCE, it would update
its own PCC LSP DB and send a PCRpt to the PCE(s) to synchronize the
change.  When the PCE receives the PCRpt msg, it updates its own PCE
LSP DB.  After this, the PCC LSP-DB and PCE LSP-DB are in sync.


## Successful MBB

Below we give an example of doing MBB to switch the Tunnel from one
path to another.  We represent the path encoded into the ERO object
as ERO={A} and ERO={B}.

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

## Aborted MBB

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

PCEP Association is a group of zero or more LSPs.

The PCE ASSO DB is populated by PCRpt messages and/or via
configuration on the PCE itself.  An Association is identified by the
Association Parameters.  The Association parameters contain many
fields, so for convenience we will group all the fields into a single
value.  We will use ASSO_PARAM=A, ASSO_PARAM=B, to refer to different
PCEP Associations: A and B, respectively.


## LSPs in same Association

Below, we give an example to illustrate how LSPs join the same
Association.

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

PCC updates the first LSP, the PCC is NOT REQUIRED to send the
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
| ASSO_PARAM=A    |                                             |
+---------------------------------------------------------------+

                Figure 11: Content of PCE ASSO DB
~~~

## Switch Association during MBB

Below, we give an example to illustrate how a Tunnel goes through MBB
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

For any PCEP object that does not have an explicit removal flag, the
absence of that object indicates removal of the constraint specified
by that object.  For example, suppose the first state-report contains
an LSPA object with some affinity constraints.  Then if a subsequent
state-report does not contain an LSPA object, then this means that
the previously specified affinity constraints do not apply anymore.
Same applies to all PCEP objects, like METRIC, BANDWIDTH, etc., which
do not have an explicit flag for removal.  This simply ensures that
it is possible to remove a constraint without using an explicit
removal flag.


# Overloaded PCE

[RFC5440] defines the concept of an Overloaded PCE and explains how
a PCE may signal to a PCC that it is congested, instructing that
"no requests should be sent to that PCE until the overload state is cleared."

In the case of an overloaded PCE, this document clarifies that it is RECOMMENDED
a PCC send a PcReq to a backup PCE instead if the primary PCE is overloaded.

[RFC8231] builds upon [RFC5440] by introducing the concept of a
Stateful PCE, which allows delegation of LSP control to a single PCE.
However, it does not address the case of an overloaded PCE. According
to [RFC8231], a PCE maintains delegation until it is revoked by the PCC
or returned back to PCC by the PCE. The PCC may revoke delegation and re-assign
it to another PCE.

As a result, a PCE in an overload state still retains LSP delegation.
For PCC-initiated LSPs, the PCC MAY revoke delegation from the overloaded
PCE and maintain delegation for itself or delegate it to another PCE. For
PCE-initiated LSPs, since the PCC cannot revoke delegation as per [RFC8281],
the overloaded PCE MAY return the delegation to the PCC.

This document clarifies that a PCC MUST continue to send PcRpt messages to a PCE in overload state otherwise
the LSP-DB on PCE may go out of sync.

# Security Considerations

None at this time.

# Managability Considerations

None at this time.

# IANA Considerations

None at this time.

--- back

# Acknowledgments
{:numbered="false"}

The authors would like to thank Adrian Farrel for useful review
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
