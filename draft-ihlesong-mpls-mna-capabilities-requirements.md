---
title: "Requirements for the Discovery of MNA Capabilities"
abbrev: "REQ-MNA-CAP"
category: info

docname: draft-ihlesong-mpls-mna-capabilities-requirements-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Routing"
workgroup: "Multiprotocol Label Switching"
keyword:
 - mpls
 - mna
 - capabilities
 - requirements
venue:
  group: "Multiprotocol Label Switching"
  type: "Working Group"
  mail: "mpls@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/mpls/"
  github: "uni-tue-kn/draft-ihlesong-mpls-mna-capabilities-requirements"
  latest: "https://uni-tue-kn.github.io/draft-ihlesong-mpls-mna-capabilities-requirements/draft-ihlesong-mpls-mna-capabilities-requirements.html"

author:
 -
    fullname: Fabian Ihle
    organization: University of Tuebingen
    city: Tuebingen
    country: Germany
    email: fabian.ihle@uni-tuebingen.de
 -
    fullname: Xueyan Song
    organization: ZTE Corporation
    country: China
    email: song.xueyan2@zte.com.cn
 -
    fullname: Greg Mirsky
    organization: Independent
    email: gregimirsky@gmail.com
 -
    fullname: Michael Menth
    organization: University of Tuebingen
    city: Tuebingen
    country: Germany
    email: michael.menth@uni-tuebingen.de

normative:

informative:
  IhMe25:
    -: ihme25
    target: https://ieeexplore.ieee.org/document/10947349
    title: MPLS Network Actions; Technological Overview and P4-Based Implementation on a High-Speed Switching ASIC
    author:
      -
        ins: F. Ihle
        name: Fabian Ihle
        org: University of Tuebingen
      -
        ins: M. Menth
        name: Michael Menth
        org: University of Tuebingen
    seriesinfo:
      DOI: 10.1109/OJCOMS.2025.3557082
    date: 2025-04-02
    format:
      PDF: https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=10947349

...

--- abstract

The MPLS Network Actions (MNA) framework encodes network actions and their ancillary data within the MPLS label stack (In-Stack Data) or after it (Post-Stack Data). To construct MPLS label stacks that carry network actions, the ingress Label Edge Router (LER) has to know the MNA capabilities of the nodes along the path, such as the Readable Label Depth (RLD), the maximum sizes of differently scoped Network Action Sub-stacks (MLD_NAS), the supported In-Stack and Post-Stack network action opcodes, and the Post-Stack parameters. This document defines these MNA capabilities, motivates why the ingress LER needs to learn them, and describes the problems that a mechanism for discovering or advertising them has to solve, including the derivation of path-wide constraints, the presence of Equal-Cost Multipath (ECMP), and changes of capabilities over time. It states requirements on such a mechanism but does not specify one.

--- middle

# Introduction
The MPLS Network Actions (MNA) framework {{?rfc9789}} provides a general mechanism for encoding network actions and their data in the MPLS label stack.
Network actions are encoded in Network Action Sub-stacks (NAS) that are placed within the MPLS label stack as In-Stack Data (ISD) or follow after it as Post-Stack Data (PSD).
The In-Stack MNA header encoding is defined in {{!rfc9994}}, and the Post-Stack MNA solution is defined in {{!I-D.ietf-mpls-mna-ps-hdr}}.

To correctly construct MPLS label stacks that carry network actions, the ingress LER needs to know the MNA capabilities of the nodes along the path that are expected to process these network actions.
An LSR is not required to process arbitrarily large or arbitrarily deep NAS, and it is not required to support every network action opcode.
If the ingress LER pushes a NAS that a node on the path cannot process, e.g., because the NAS is beyond the node's readable depth, exceeds the node's maximum supported NAS size, or uses an unsupported opcode, the requested network action is silently not performed or, depending on the implementation, the packet is dropped or punted to the slow path.
The ingress LER therefore has to know these capabilities before it uses MNA on a path.
The relevant capabilities include:

1. In-Stack MNA capabilities:
   - The Readable Label Depth (RLD): the number of Label Stack Entries (LSEs) a node can parse without performance impact.
   - The NAS Maximum Label Depth (MLD_NAS): the maximum supported NAS size for each scope (select, HBH, I2E).
   - The Maximum NAS Count (MNC): the maximum number of NAS of each scope (select, HBH, I2E) that the node can process.
   - The supported In-Stack network action opcodes.
2. Post-Stack MNA capabilities:
   - Whether the node supports Post-Stack MNA processing as defined in {{!I-D.ietf-mpls-mna-ps-hdr}},
   - The maximum Post-Stack MPLS Header (PSMH) size (MLD_PSMH),
   - The maximum number of PSMHs the node can process (MPC), since a packet may carry one PSMH per scope,
   - The RLD including the PSMH (RLD_PSMH),
   - The maximum start offset at which a node can still locate a PSMH (MPSO),
   - The supported Post-Stack network action opcodes.

Post-Stack MNA does not replace the In-Stack capabilities: the presence of a PSMH is indicated by the P bit in an In-Stack NAS {{!I-D.ietf-mpls-mna-ps-hdr}}, so a packet carrying PSD also carries ISD, and the In-Stack capabilities apply to it as well ({{psd-isd}}).

Section 5.3 of {{!I-D.ietf-mpls-mna-ps-hdr}} explicitly requires that each participating node signals its Post-Stack capabilities to the encapsulating node.
More generally, none of the In-Stack or Post-Stack capabilities above is known to the ingress LER a priori.

This document has two purposes.
First, it defines the MNA capabilities that the ingress LER needs to know and motivates each of them ({{capabilities}}).
Second, it describes the problems that any mechanism for making these capabilities available to the ingress LER has to solve, namely the derivation of path-wide constraints from per-node capabilities ({{aggregation}}) and a number of considerations such as the relationship to path selection, the presence of Equal-Cost Multipath (ECMP), and changes of capabilities over time ({{considerations}}).
From these, it derives a set of requirements on such a mechanism ({{requirements}}).

## Scope and Non-Goals
This document defines capabilities and requirements only.
It does not specify how the capabilities are discovered or advertised, and it neither favors nor excludes a particular solution or class of solutions.
The requirements in {{requirements}} are therefore stated so that they can be evaluated against any candidate mechanism, whether the mechanism distributes capability information in the control plane, exposes it through a management or telemetry interface, collects or verifies it on the forwarding path, obtains it by configuration, or takes some other approach altogether.
A solution may also combine several such mechanisms, for example by using one mechanism for coarse feasibility information and another for the detailed capability set.
The trade-offs between these approaches, for example between the overhead of distributing capability information to nodes that do not need it and the path fidelity that an on-path mechanism can offer, are discussed at the level of requirements in {{considerations}} but are not resolved here.

## Terminology

{::boilerplate bcp14-tagged}

Although this document does not specify a protocol mechanism, it uses BCP 14 language to state the requirements that constrain a conforming mechanism and the constraints that the ingress LER has to observe when it constructs MPLS label stacks with MNA.

This document uses the term participating node for a node that is expected to process a given NAS, i.e., a node that is MNA-capable, supports the scope of that NAS, and is intended by the operator to act on the network actions it carries.
The participating nodes are, in general, a subset of the nodes on the path, and the subset differs per NAS.
Only the capabilities of the participating nodes of a NAS constrain that NAS.
How the set is determined per scope is described in {{aggregation}}.

### Abbreviations
This document makes use of the terms defined in {{!rfc9994}} and in {{?rfc9789}}.

| Abbreviation | Name                      | Description                                                                                                  | Reference                     |
| ------------ | ------------------------- | ------------------------------------------------------------------------------------------------------------ | ----------------------------- |
| NAS          | Network Action Sub-stack  | A stack of related LSEs in the MPLS stack containing network actions and ancillary data.                     | {{?rfc9789}}                  |
| RLD          | Readable Label Depth      | The number of LSEs a node can parse.                                                                         | {{!rfc9994}}                  |
| MLD_NAS      | NAS Maximum Label Depth   | The maximum number of LSEs in a NAS that a node can process, defined per scope.                              | This document                 |
| MNC          | Maximum NAS Count         | The maximum number of NAS of a given scope that a node can process, defined per scope.                       | This document                 |
| PSMH         | Post-Stack MPLS Header    | The header after the BOS carrying post-stack network actions and ancillary data.                             | {{!I-D.ietf-mpls-mna-ps-hdr}} |
| PSD          | Post-Stack Data           | Network actions and data encoded after the MPLS label stack.                                                 | {{!I-D.ietf-mpls-mna-ps-hdr}} |
| ISD          | In-Stack Data             | Network actions and data encoded within the MPLS label stack.                                                | {{!rfc9994}}                  |
| MLD_PSMH     | Maximum PSMH Size         | The maximum size of a single PSMH a node can process.                                                        | This document                 |
| MPC          | Maximum PSMH Count        | The maximum number of PSMHs (one per scope) that a node can process.                                         | This document                 |
| RLD_PSMH     | RLD including PSMH        | The total parseable depth including label stack and PSMHs, but excluding the post-stack offset.              | This document                 |
| MPSO         | Maximum PSMH Start Offset | The largest start offset from the BoS at which a node can still locate and process a PSMH.                   | This document                 |
{: #table_abbrev title="Abbreviations."}


# MNA Capabilities {#capabilities}

This section defines the parameters that describe the MNA capabilities of an LSR and that the ingress LER needs to know to construct MPLS label stacks with MNA.
For each parameter, it motivates why the ingress LER needs to know it and states the constraint that the ingress LER has to observe once it knows the value.
How the ingress LER obtains the value is out of scope for this document.

## In-Stack MNA Capabilities {#in-stack}

### The Readable Label Depth (RLD)

The Readable Label Depth (RLD) is the number of LSEs an LSR can parse without performance impact {{?rfc9789}}.
An LSR is required to search the MPLS stack for NAS that have to be processed by the LSR.
To that end, the network actions must be within the RLD of the node.
For HBH-scoped network actions, the ingress LER that pushes the network actions MUST ensure that the actions are readable at each LSR on the path that is required to process them, i.e., that they are placed within the RLD of each such node.

An operator may decide that a node with a low RLD does not participate in the processing of a particular network action.
Such a node is then not among the nodes required to process that action, and its RLD does not constrain the placement of the corresponding NAS.
The requirement above applies only to the nodes that are expected to act on the network action: placing a NAS beyond the RLD of such a node means the node silently does not perform the requested action, which the ingress LER cannot detect.

#### Example

An example for the RLD parameter is given in {{fig-rld_example}}.
With an RLD of 5, an LSR is capable of reading labels A, B, C, D, and E but not F.
An RLD of 8 is required in this example to read the entire MPLS stack.

~~~~
{::include ./drawings/rld_example.txt}
~~~~
{: #fig-rld_example title="Example MPLS stack of 8 MPLS LSEs illustrating the concept of RLD."}

### Maximum NAS Sizes (MLD_NAS)
This section gives a motivation for knowing maximum NAS sizes and then introduces the NAS Maximum Label Depth (MLD_NAS).

#### Motivation
A NAS in the MNA header encoding is at least 2 LSEs and at most 17 LSEs large {{!rfc9994}}.
At an LSR, one or more NAS, e.g., a select-scoped and a hop-by-hop-scoped NAS, are possible.
With two maximum-sized NAS, an LSR is required to reserve 34 LSEs in hardware to be able to process network actions.
This consumes hardware resources that may be needed to encode other LSEs, e.g., forwarding labels for SR-MPLS paths, or are not available in less capable devices.

This worst case is not a property of a particular hardware architecture but follows from the in-stack encoding itself.
Without a means for an LSR to inform the ingress LER that it cannot process maximum-sized NAS, the ingress LER has no basis on which to assume anything smaller and may push NAS of up to 17 LSEs per scope.
Every LSR on the path therefore has to provision for the worst case, irrespective of the NAS sizes actually used in the deployment.
The scope-specific parameter defined in this section removes that assumption once the ingress LER can learn it.

Many use cases in the MNA framework {{?rfc9791}} do not require a maximum-sized NAS of 17 LSEs to encode network actions and their ancillary data.
Therefore, a NAS can be up to 17 LSEs but nodes can also support smaller maximum NAS.
Knowing the maximum supported NAS size at the ingress LER prevents an LSR from receiving packets with a larger NAS than it supports.
This way, the allocated resources for NAS can be reduced if smaller maximum NAS are supported.
More resources are available for other purposes, and hardware with a low RLD can be made MNA-capable {{IhMe25}}.

#### NAS Maximum Label Depth
The maximum supported number of LSEs in a NAS that an LSR can process is referred to as NAS Maximum Label Depth (MLD_NAS) in this document.
For each scope in MNA, a separate parameter for the MLD_NAS exists, called MLD_NAS_Select, MLD_NAS_HBH, and MLD_NAS_I2E.
These parameters include the Format A, B, C, and D LSEs from {{!rfc9994}} in a NAS.
The MLD_NAS is defined per scope because the number of nodes that constrain a NAS differs by scope: a select-scoped NAS is processed by a single specified node, an HBH-scoped NAS by every node on the path, and an I2E-scoped NAS by the egress node.

Based on the maximum supported NAS sizes, the ingress LER has to observe the following when pushing the MPLS stack and NAS on a packet:

- The ingress LER MUST NOT push a select-scoped NAS that is larger than the MLD_NAS_Select value of the node that will process the select-scoped NAS.
- The ingress LER MUST NOT push an HBH-scoped NAS that is larger than the minimum of the MLD_NAS_HBH values of all participating nodes of that NAS.
- The ingress LER MUST NOT push an I2E-scoped NAS that is larger than the MLD_NAS_I2E value of the egress node.

These constraints apply to values that the ingress LER has actually learned.
If the ingress LER does not know a value for a scope at a participating node, it does not derive a constraint from the values it does know, and the conservative fallback described in {{aggregation}} applies.

The constraints are normative because exceeding a maximum supported value does not degrade performance gracefully.
An LSR that receives a NAS larger than it can process is unable to complete the network action in the forwarding path.
Depending on the implementation, the packet is either dropped or punted to the slow path, so that constructing such stacks results in packet loss or, at sufficient traffic rates, in a denial of service against the control plane of the LSR.
Since the ingress LER has the information needed to avoid this once it knows the MLD_NAS values, it is required to do so.

#### Example

{{fig-nas_sizes_example}} illustrates the different MLD_NAS sizes in an MPLS stack.
In this example, a select-scoped NAS has a maximum size of 4 LSEs, a hop-by-hop-scoped NAS of 7 LSEs, and an I2E-scoped NAS of 4 LSEs.

~~~~
{::include ./drawings/nas_sizes_example.txt}
~~~~
{: #fig-nas_sizes_example title="Example MPLS stack illustrating the different NAS sizes."}

### Maximum Number of NAS per Scope (MNC)
{{!rfc9994}} allows more than one NAS of the same scope to be present in the MPLS label stack.
For a node, processing several separate NAS of a scope is not equivalent to processing a single NAS of the same total size.
Each NAS carries its own Format A and Format B LSEs and requires separate parsing state, so a node may be able to process a NAS of a given size but only a limited number of distinct NAS of that scope.
For example, parsing two select-scoped NAS of 4 LSEs each can be more demanding for a node than parsing a single select-scoped NAS of 8 LSEs, even though the total number of LSEs is the same.

The maximum number of NAS of a given scope that an LSR can process is referred to as Maximum NAS Count (MNC) in this document.
As for the MLD_NAS, a separate parameter exists per scope, called MNC_Select, MNC_HBH, and MNC_I2E.
The MNC constrains the number of NAS of a scope independently of the MLD_NAS, which constrains the size of each individual NAS.

Based on the MNC values, the ingress LER has to observe the following when pushing the MPLS stack and NAS on a packet:

- The ingress LER MUST NOT push more select-scoped NAS for a node than that node's MNC_Select.
- The ingress LER MUST NOT push more HBH-scoped NAS than the minimum of the MNC_HBH values of all participating nodes of these NAS.
- The ingress LER MUST NOT push more I2E-scoped NAS than the egress node's MNC_I2E.

As for the MLD_NAS, these constraints apply only to values that the ingress LER has actually learned, and exceeding a value does not degrade gracefully: a node that receives more NAS of a scope than it can process is unable to complete the corresponding network actions in the forwarding path, with the same consequences as described for the MLD_NAS.

### Supported In-Stack Network Action Opcodes {#in-stack-opcodes}
An LSR does not necessarily support every In-Stack network action opcode.
Unlike the size and depth parameters above, an unsupported opcode has a defined outcome in the data plane: Section 5.4 of {{!rfc9994}} specifies that a node that does not understand an opcode within a NAS acts according to the Unknown Network Action Handling (U) bit of that NAS, i.e., it skips to the next network action when U is 0 and drops the packet when U is 1.
The U bit exists so that this case is handled deterministically rather than being left to the implementation, and an ingress LER that is aware of it can use it deliberately, e.g., to fail closed for a network action whose omission would be unsafe.

However, neither outcome achieves what the ingress LER intends when it pushes a network action.
With U set to 0, the network action is not performed and the packet is forwarded as if it had never been requested, which the ingress LER cannot detect.
With U set to 1, the packet is dropped, so that the traffic is lost rather than merely unprocessed.
Unknown Network Action Handling therefore makes the failure mode predictable, but it does not remove the need to know which opcodes a node supports.
This is consistent with Section 2.2 of {{?rfc9789}}, which recommends that care be taken not to construct an LSP that traverses nodes that do not support a network action that has to be processed at every hop.

The ingress LER thus needs to know the set of In-Stack network action opcodes that each participating node supports.
Knowing them, the ingress LER SHOULD refrain from pushing an In-Stack network action for a node that does not support the corresponding opcode, and instead use a network action that is supported on the path, select a different path, or refrain from using MNA on that path.
If the support for an opcode at a participating node is not known, the ingress LER treats the opcode as not supported by that node, as described in {{aggregation}}.

## Post-Stack MNA Capabilities
The Post-Stack MNA solution {{!I-D.ietf-mpls-mna-ps-hdr}} allows network actions and their ancillary data to be encoded after the bottom of the MPLS label stack in a Post-Stack MPLS Header (PSMH).
A packet may carry more than one PSMH: a separate Post-Stack MPLS Base Header is added for each scope {{!I-D.ietf-mpls-mna-ps-hdr}}, so post-stack network actions of different scopes result in several PSMHs after the bottom of the label stack, analogous to the several NAS that may appear in the label stack for ISD.
Section 5.3 of {{!I-D.ietf-mpls-mna-ps-hdr}} requires that each participating node signals its Post-Stack capabilities to the encapsulating node.
This section defines the parameters for that purpose.

{{fig-psmh_sizes_example}} gives an overview of the post-stack sizes defined in this section.
It shows how the MLD_PSMH, the RLD_PSMH, and the MPSO relate to the MPLS label stack, the In-Stack NAS that carries the P bit, the post-stack offset, and a Post-Stack MPLS Header.
The MLD_PSMH bounds the size of each PSMH itself, excluding its Post-Stack MPLS Base Header.
The RLD_PSMH bounds what a node reads, i.e., the MPLS label stack and the PSMHs; the post-stack offset in between is skipped rather than read and is bounded by the MPSO instead, which is why the RLD_PSMH bracket in the figure is interrupted there.
When several PSMHs are present, the figure repeats accordingly, with one In-Stack NAS and one Post-Stack MPLS Base Header per scope, all covered by the RLD_PSMH.

~~~~
{::include ./drawings/psmh_sizes_example.txt}
~~~~
{: #fig-psmh_sizes_example title="Overview of the post-stack parameters MLD_PSMH, RLD_PSMH, and MPSO. The RLD_PSMH does not cover the post-stack offset, which is bounded by the MPSO."}

### In-Stack Data Required for Post-Stack MNA {#psd-isd}
Post-Stack MNA is not a self-contained alternative to In-Stack MNA.
The presence of a PSMH is indicated by the P bit, which {{!I-D.ietf-mpls-mna-ps-hdr}} defines as bit 20 of the Format B LSE of the corresponding In-Stack NAS, and which MUST be set to 1 when the corresponding PSMH is added to the packet.
The scope (IHS) and the Unknown Network Action Handling (U) that apply to the Post-Stack network actions are those of that NAS as well.
A node that does not support the P bit skips the processing of the PSMH altogether {{!I-D.ietf-mpls-mna-ps-hdr}}.

A packet that carries Post-Stack network actions therefore always carries an In-Stack NAS for each scope for which a PSMH is present, even when no In-Stack network action is needed.
Such a NAS is at least 2 LSEs large, consisting of a Format A LSE and a Format B LSE that carries the No-Operation opcode {{!rfc9994}} with the P bit set.
It grows when further In-Stack opcodes are needed: if the PSMH does not start immediately after the BoS, the PSMH Start Offset opcode is carried in the NAS, and the PSMH End Offset opcode may be carried so that a node can determine the size of the PSMH without parsing it {{!I-D.ietf-mpls-mna-ps-hdr}}.
Since only the first opcode of a NAS is encoded in the Format B LSE, each further opcode adds at least one Format C LSE to the NAS.

Consequently, the In-Stack capabilities of {{in-stack}} also constrain a deployment that uses Post-Stack network actions exclusively.
For each scope for which a PSMH is present, the participating nodes have to support that scope, their MLD_NAS has to accommodate the accompanying NAS, their MNC has to admit it, and the NAS has to be placed within their RLD.
An implementation that supports Post-Stack MNA but has an MLD_NAS below 2 LSEs for a scope, or an MNC of 0 for that scope, cannot process Post-Stack network actions of that scope.

### Post-Stack MNA Support
A node MAY support Post-Stack MNA processing.
Per {{!I-D.ietf-mpls-mna-ps-hdr}}, the encapsulating node does not add a Post-Stack MPLS Header to a packet whose decapsulating node does not support Post-Stack MNA processing.
Therefore, the ingress LER needs to know whether each node on the path that would process a PSMH supports Post-Stack MNA.

### Maximum Post-Stack MPLS Header Size (MLD_PSMH)
The PSMH-Len field in the Post-Stack MPLS Base Header indicates the length of the Post-Stack MPLS Header, excluding the 4-octet Post-Stack MPLS Base Header itself {{!I-D.ietf-mpls-mna-ps-hdr}}.
Hardware implementations may have limits on the maximum PSMH size they can process.
The maximum supported PSMH length is referred to as MLD_PSMH in this document, analogous to MLD_NAS for ISD.
It is expressed in the same units as the PSMH-Len field and therefore likewise excludes the Post-Stack MPLS Base Header.
The MLD_PSMH applies to each PSMH individually, not to their combined size; the combined post-stack depth is bounded by the RLD_PSMH instead.
Based on the MLD_PSMH values, the ingress LER has to observe the following:

- The ingress LER MUST NOT add a PSMH with a PSMH-Len exceeding the MLD_PSMH of any node that will process that PSMH.

### Maximum Number of PSMHs (MPC)
As described above, a packet may carry more than one PSMH, one per scope.
Processing several PSMHs is not equivalent to processing a single larger PSMH: each PSMH has its own base header and requires separate parsing state, analogous to the Maximum NAS Count for In-Stack NAS.
A node may therefore support Post-Stack MNA but be able to process only a limited number of PSMHs in a packet.
The maximum number of PSMHs that an LSR can process is referred to as Maximum PSMH Count (MPC) in this document.
The MPC constrains the number of PSMHs independently of the MLD_PSMH, which constrains the size of each individual PSMH.

Based on the MPC values, the ingress LER has to observe the following:

- For each node, the number of PSMHs whose scope reaches that node MUST NOT exceed that node's MPC. An HBH-scoped PSMH is processed by every node on the path, an I2E-scoped PSMH only by the egress node, and a select-scoped PSMH only by the node it targets.

As for the other post-stack parameters, exceeding a node's MPC does not degrade gracefully: a node that receives more PSMHs than it can process is unable to complete the corresponding post-stack network actions in the forwarding path.

### Readable Label Depth Including Post-Stack MPLS Header (RLD_PSMH)
Section 5.3 of {{!I-D.ietf-mpls-mna-ps-hdr}} requires each participating transit and decapsulating node to signal its "Readable Label Depth including PSMH", i.e., the RLD of {{!rfc9994}} extended to include the length of the PSMH.
This parameter is referred to as RLD_PSMH in this document.

The RLD_PSMH covers the parts of the packet that the node has to read, i.e., the MPLS label stack and the PSMHs, as illustrated in {{fig-psmh_sizes_example}}.
It does not cover the post-stack offset, i.e., the octets between the BoS and the beginning of the first PSMH that are occupied by other post-stack headers, for example a pseudowire control word or a Generic Associated Channel Header {{!I-D.ietf-mpls-mna-ps-hdr}}.
Those octets are not parsed as part of the PSMH processing but skipped, and their number is not inferred by the node but signaled explicitly in the packet by the PSMH Start Offset opcode in the corresponding In-Stack NAS.
How far a node can skip is a capability of its own and is expressed by the MPSO ({{mpso}}) rather than by the RLD_PSMH.
Keeping the two apart means that neither value has to be recomputed when the other changes, e.g., when a deployment starts or stops using a control word before the PSMH.

Based on the RLD_PSMH values, the ingress LER MUST ensure that the combined depth of the MPLS label stack and of all PSMHs intended for a node does not exceed that node's RLD_PSMH.

The RLD_PSMH is a capability of its own and is not derivable from the RLD: a node that can read a given number of LSEs of the label stack cannot necessarily read the same number of LSEs and a PSMH in addition.
Where an implementation has a single parsing budget from which the label stack, the skipped offset, and the PSMHs are all served, it accounts for the offset it can skip when it determines the values that it reports, so that its RLD_PSMH and its MPSO can be observed simultaneously.

### Maximum PSMH Start Offset (MPSO) {#mpso}
The RLD_PSMH bounds what a node reads, but it does not express where after the BoS a node can locate a PSMH.
A node may be able to process a PSMH that starts immediately after the BoS but not one that starts at an arbitrary offset, for example because its parser recognizes only a fixed set of intervening post-stack headers or because it can skip only a fixed number of octets after the BoS.
The largest start offset from the BoS at which a node can still locate and process a PSMH is referred to as Maximum PSMH Start Offset (MPSO) in this document.
An MPSO of 0 means that the node can process a PSMH only when it starts immediately after the BoS, which is the default placement per {{!I-D.ietf-mpls-mna-ps-hdr}}.

Whether a node understands the PSMH Start Offset opcode at all is expressed by its set of supported In-Stack opcodes ({{in-stack-opcodes}}).
The MPSO expresses how large a value of that opcode the node can act upon.
Since the offset is signaled explicitly in the packet, the ingress LER can check it against the MPSO independently of the depth check against the RLD_PSMH.
This lets the ingress LER distinguish a node that cannot read deeply enough from a node that can read the required depth but cannot skip the intervening post-stack headers.

Based on the MPSO values, the ingress LER has to observe the following:

- The ingress LER MUST NOT encode a PSMH at a start offset larger than the MPSO of any node that will process that PSMH.

### Supported Post-Stack Network Action Opcodes
The Post-Stack network action opcode space (MNA-PS-OP) is 7 bits wide, so up to 128 Post-Stack opcodes can be defined {{!I-D.ietf-mpls-mna-ps-hdr}}, and an LSR does not necessarily support all of those that are defined.
As for In-Stack opcodes ({{in-stack-opcodes}}), an unsupported Post-Stack opcode has a defined outcome: a transit node MUST respect the Unknown Network Action Handling flag encoded in the corresponding NAS when it processes the PSMH {{!I-D.ietf-mpls-mna-ps-hdr}}, i.e., it skips the unknown Post-Stack network action or drops the packet.
The U bit of a NAS applies jointly to the In-Stack and the Post-Stack network actions of that scope, so the ingress LER cannot choose a different failure mode for the two.
The considerations of {{in-stack-opcodes}} apply accordingly: the handling is predictable, but neither skipping nor dropping performs the requested network action.

The ingress LER therefore needs to know the set of Post-Stack network action opcodes that each participating node supports, and it SHOULD refrain from adding a Post-Stack network action for a node that does not support the corresponding opcode.
If the support for a Post-Stack opcode at a participating node is not known, the ingress LER treats the opcode as not supported by that node, as described in {{aggregation}}.

The Post-Stack opcode space is separate from the In-Stack opcode space; a node may support a network action In-Stack, Post-Stack, or both.
Support for a network action in one space therefore MUST NOT be assumed to imply support in the other.

# Deriving Path-Wide Constraints {#aggregation}
The capabilities in {{capabilities}} are per-node properties, but the ingress LER constructs a single label stack for a path.
It therefore has to combine the per-node capabilities into path-wide constraints.

## Participating Nodes {#participating}
The capabilities that constrain a NAS are those of the participating nodes of that NAS, not those of all nodes on the path.
Which nodes participate follows from the scope of the NAS and from the deployment:

- A select-scoped NAS is processed by the node that brings it to the top of the label stack {{!rfc9994}}, so exactly that node participates.
- An I2E-scoped NAS MUST NOT be processed by any node except the egress node {{!rfc9994}}, so only the egress node participates.
- An HBH-scoped NAS is processed by the MNA-capable nodes on the path that are expected to act on the network actions it carries. An operator may decide that a node does not participate in a particular network action, for example a node with a low RLD, and a node that is not MNA-capable at all cannot participate in any case.

A node that does not participate contributes no capability values to the constraints of that NAS, but it may still constrain the label stack in other ways.
Per Section 7 of {{!rfc9994}}, a NAS MUST NOT appear at the top of the label stack at an MNA-incapable node, which restricts where a NAS may be placed independently of any capability value.
Per Section 2.2 of {{?rfc9789}}, care should be taken not to construct an LSP that traverses nodes that do not support a network action that has to be processed at every hop, which the ingress LER can only do if it knows which nodes these are.
Consequently, the ingress LER needs to know which nodes on a path are MNA-capable and which of them participate in a given network action, in addition to the capability values of the participating nodes.

## Combination of the Capabilities {#combination}
Independently of how the ingress LER learns the per-node capabilities, their combination follows from the scope of each parameter.
In the following, the participating nodes are those of the NAS in question, as defined in {{participating}}:

- The path-wide RLD for a NAS is the minimum RLD of its participating nodes.
- The path-wide MLD_NAS_HBH is the minimum MLD_NAS_HBH of the participating nodes of the HBH-scoped NAS.
- The MLD_NAS_Select for a specific node is the value of that node.
- The MLD_NAS_I2E is the value of the egress node.
- The MNC values are combined per scope in the same way as the MLD_NAS values: the path-wide MNC_HBH is the minimum over the participating nodes, the MNC_Select for a specific node is the value of that node, and the MNC_I2E is the value of the egress node.
- A scope is available on the path only if every node that would process a NAS of that scope supports it. In particular, an HBH-scoped NAS cannot be used if a node that is expected to process it does not support the HBH scope.
- The path-wide supported opcodes for an HBH-scoped NAS are the intersection of the opcodes supported by its participating nodes. The supported opcodes for a select-scoped NAS are those of the node processing that NAS, and for an I2E-scoped NAS those of the egress node.
- Post-Stack capabilities are combined analogously, but taking into account that different nodes process different PSMHs: Post-Stack MNA is available only if the decapsulating node supports it; the size of each PSMH (MLD_PSMH), the number of PSMHs at a node (MPC), the start offset of each PSMH (MPSO), and the combined depth of the label stack and the PSMHs at a node (RLD_PSMH) are each constrained by the nodes that process the respective PSMH(s), as defined in {{capabilities}}; and the supported Post-Stack opcodes are combined per scope like the In-Stack opcodes.
- Since a PSMH is accompanied by an In-Stack NAS with the P bit set ({{psd-isd}}), the In-Stack constraints of the corresponding scope apply to that NAS in addition.

The RLD is combined into a single path-wide minimum only if a single copy of the NAS is used.
If several copies of an HBH-scoped NAS are placed in a deep label stack, as described in Section 7 of {{!rfc9994}}, the RLD constraint applies to each copy and to the participating nodes that read that copy, rather than to a single path-wide minimum.

## Unknown Capabilities {#unknown}
For the combinations above to be correct, the ingress LER has to know the capabilities of exactly the participating nodes of each NAS.
For an HBH-scoped NAS, and for the RLD that applies to it, these are all nodes that are expected to process it, so missing information about a single participating node invalidates the path-wide constraint.
A mechanism that provides the per-node capabilities therefore has to make it possible for the ingress LER to determine whether it has the capabilities of all participating nodes, and to identify the nodes for which a value is unknown.

If a capability of a participating node is unknown, the ingress LER MUST NOT derive a bound from the values that it does know, since the unknown value can be more restrictive than any of them.
Instead, it falls back conservatively, which in this document means that it does one of the following:

- It refrains from using the network action, the scope, or the encoding that depends on the unknown value on that path, up to refraining from using MNA on that path altogether.
- It restricts itself to values that are known to be supported by every node in the domain by other means, e.g., a domain-wide minimum established by configuration or by the hardware baseline of the deployment.
- It selects a different path or egress node whose capabilities it does know, if it is able to steer the traffic as described in {{path-selection}}.

In particular, an opcode or a scope whose support at a participating node is unknown is treated as not supported by that node, and a size or depth parameter that is unknown at a participating node yields no bound at all rather than a bound derived from the other nodes.

# Considerations for Capability Discovery {#considerations}
This section describes problems that any mechanism for discovering or advertising MNA capabilities has to address.
These considerations motivate the requirements in {{requirements}}.

## Relationship to Path Selection {#path-selection}
Knowing the MNA capabilities is, in general, not an input to path computation.
Paths are selected by existing mechanisms, and the ingress LER then needs to know the MNA constraints of the selected path before network actions are pushed onto it.

Only coarse feasibility information may need to be available at path selection time, and such information already has a home in the IGP: the RLD can be advertised as MSD-Type 3 {{?rfc9789}} using the MSD advertisements of IS-IS {{?rfc8491}} and OSPF {{?rfc8476}}.
In contrast, the detailed capability set, i.e., the supported opcodes, the per-scope NAS size limits, and the Post-Stack parameters, is voluminous and only relevant for the specific paths on which an ingress LER intends to use MNA.
Making all of this information available to every node in the domain, for example by flooding it through the IGP, would burden all nodes with information that most of them never use.
This tension between the amount of information distributed and the number of nodes that need it is a central consideration for a discovery or advertisement mechanism and is reflected in {{requirements}}.

If the capabilities of a selected path are not sufficient for the intended network actions, the ingress LER can adapt the NAS construction to the constraints, or refrain from using MNA on that path.
If the ingress LER can steer traffic, e.g., using SR-TE candidate paths or RSVP-TE explicit routes, it can also select an alternate candidate path or egress node.
In particular, when multiple candidate egress nodes exist, the ingress LER can select an egress node that supports the required network actions, e.g., the required MLD_NAS_I2E or Post-Stack decapsulation support.

## MNA Capabilities in the Presence of ECMP {#ecmp}
When the LSP traverses one or more Equal-Cost Multipath (ECMP) sets, the nodes that a particular data packet of a flow visits are not necessarily the same across the ECMP set.
Capabilities that hold for one traversal of the path are therefore not sufficient to constrain the NAS that the ingress LER pushes onto the LSP, because a different member of the ECMP set may have more restrictive capabilities.

To construct a NAS that is processable by any packet of the flow, the ingress LER has to apply the path-wide constraints of {{aggregation}}, with the set of participating nodes extended to the participating nodes on every branch of each ECMP set.
The minima and the scope and opcode intersections defined there are then taken across all branches rather than over a single traversal, while the select-scoped and I2E-scoped constraints remain those of the node that processes the corresponding NAS.

This requires that the mechanism makes the capabilities of all members of an ECMP set available to the ingress LER, or otherwise allows the ingress LER to determine that it has covered every branch.
If the ingress LER cannot establish that it has the capabilities of every branch of an ECMP set, it MUST treat the capabilities of the nodes on the branches that it has not covered as unknown, in the sense of {{unknown}}.
This means in particular that the ingress LER derives no bound from the branches that it does know, because a branch that it does not know may be more restrictive: a minimum taken over the known branches is not a valid path-wide constraint, and an opcode or a scope supported on the known branches is not thereby supported on the LSP.
The ingress LER then falls back conservatively as described in {{unknown}}, i.e., it refrains from using MNA on that LSP or restricts itself to the network actions, scopes, and NAS sizes that are supported throughout the domain by other means.

Capabilities of nodes that are on every branch of the ECMP set remain usable.
For example, the egress node terminates all branches of an ECMP set within the LSP, so its capabilities constrain an I2E-scoped NAS as they do without ECMP, provided that the placement restrictions of Section 7 of {{!rfc9994}} are met on every branch, e.g., that no MNA-incapable node exposes the NAS at the top of the label stack when it pops the forwarding label.

Load balancing based on an entropy label {{?rfc6790}} or on fields of the payload means that the member of an ECMP set taken by a given flow cannot, in general, be predicted or reproduced from outside the flow.
This does not change the requirement: the ingress LER does not need to establish the capabilities of the branch taken by an individual flow, but the capabilities that hold for every branch a flow may take.
The ingress LER constructs NAS that are processable on all branches of the ECMP set, so per-flow fidelity is not required.
What a mechanism has to enable is either the coverage of every branch of an ECMP set, or the ingress LER's ability to detect that coverage is incomplete, not the identification of the branch that a given flow takes.

## Changes of Capabilities over Time {#change}
The MNA capabilities of a path can change after the ingress LER has learned them, for example when the path changes due to IGP reconvergence or Fast Reroute activation, when a node is reconfigured, or when a node is replaced by hardware with different capabilities.
Stale capability information can be worse than none, because the ingress LER may construct a stack that a node on the new or reconfigured path cannot process.
A mechanism therefore has to make the ingress LER aware of relevant changes in a timely manner, either by notifying it of changes or by allowing it to re-establish the capabilities of a path when the path may have changed.
The ingress LER SHOULD re-establish the capabilities of a path when it has reason to believe that the path or the capabilities of a node on it may have changed.

## Detecting MNA-Incapable Nodes {#incapable}
Not every node on a path necessarily supports MNA at all.
A node that does not support MNA cannot process any NAS, so its presence on the path constrains which network actions can be used, in the same way as a node with limited capabilities.
A mechanism therefore has to allow the ingress LER to distinguish, for each node on the path, between a node that supports MNA and reports its capabilities, a node that supports MNA but for which a particular value is unknown, and a node that does not support MNA at all.
A node that does not support MNA MUST be treated as not supporting any scope, opcode, or Post-Stack processing, and it therefore never participates in a NAS in the sense of {{participating}}.
Not participating does not mean that such a node can be disregarded: it still restricts where a NAS may be placed in the label stack, as described in {{participating}}.

## Per-Interface and Per-Node Capabilities {#per-interface}
The capabilities in {{capabilities}} are described as properties of a node, but in some implementations they can differ per interface.
When capabilities vary per interface, the ingress LER needs the capabilities that apply to the forwarding path that the flow actually takes through the node, i.e., the incoming and outgoing interfaces used on that path.
A mechanism has to make it possible to associate the reported capabilities with the relevant interfaces, or to report the most conservative capabilities of the node when a per-interface association is not possible.

# Requirements {#requirements}
This section summarizes the requirements on a mechanism for making MNA capabilities available to the ingress LER.
The requirements follow from the capabilities in {{capabilities}} and the considerations in {{considerations}}.

- REQ-1: The mechanism MUST allow the ingress LER to learn the RLD of each node on a path.
- REQ-2: The mechanism MUST allow the ingress LER to learn, per scope (select, HBH, I2E), both the maximum NAS size (MLD_NAS) and the maximum number of NAS (MNC) that each participating node can process, and to learn which scopes a node supports.
- REQ-3: The mechanism MUST allow the ingress LER to learn the set of supported In-Stack network action opcodes of each participating node.
- REQ-4: The mechanism MUST allow the ingress LER to learn, for each participating node, whether it supports Post-Stack MNA and, if so, its MLD_PSMH, the maximum number of PSMHs it can process (MPC), its RLD_PSMH, its MPSO, and its supported Post-Stack network action opcodes.
- REQ-5: The mechanism MUST allow the ingress LER to associate the learned capabilities with the forwarding path and to determine whether it has the capabilities of all nodes a flow may traverse, including across ECMP sets. Full coverage of every ECMP branch is desirable but not required; where it is not achieved, the ingress LER falls back conservatively ({{ecmp}}).
- REQ-6: The mechanism MUST allow the ingress LER to distinguish a node that does not support MNA, a node that supports MNA but for which a value is unknown, and a node that reports a value ({{incapable}}), so that the ingress LER can determine the participating nodes of a NAS ({{participating}}) and apply the conservative fallback where a value is unknown ({{unknown}}).
- REQ-7: The mechanism SHOULD allow the ingress LER to become aware of changes of the capabilities of a path in a timely manner, or to re-establish them when the path may have changed ({{change}}).
- REQ-8: The mechanism SHOULD scale with the number of paths on which MNA is used, rather than with the size of the domain, so that capability information is not imposed on nodes that do not need it ({{path-selection}}).
- REQ-9: Where capabilities can vary per interface, the mechanism SHOULD allow the ingress LER to obtain the capabilities relevant to the forwarding path through a node ({{per-interface}}).

# Example
Consider an SR-MPLS path with three LSRs: R1, R2 (transit), and R3 (egress).
The ingress LER R0 intends to use MNA on this path, both In-Stack (an HBH-scoped NAS and a select-scoped NAS for R2) and Post-Stack (an HBH-scoped and an I2E-scoped PSMH).
The derivation below illustrates the path-wide constraints for all parameters, including those for scopes that R0 does not use in this particular case.
Suppose that, by some mechanism, R0 has learned the following capabilities of the nodes on the path:

| Node | RLD | MLD_NAS_Select | MLD_NAS_HBH | MLD_NAS_I2E    | PS_Supported | MLD_PSMH | RLD_PSMH |
| ---- | --- | -------------- | ----------- | -------------- | ------------ | -------- | -------- |
| R1   | 20  | 9              | 9           | 0 (not egress) | Yes          | 16       | 36       |
| R2   | 51  | 9              | 3           | 0 (not egress) | Yes          | 8        | 59       |
| R3   | 35  | 9              | 9           | 9              | Yes          | 16       | 51       |
{: #table_example title="Example MNA capabilities of the nodes on a path."}

All three nodes are MNA-capable and participate in the network actions that R0 intends to use, so the participating nodes of the HBH-scoped NAS are R1, R2, and R3.
For simplicity, each node in this example can process a single NAS per scope, i.e., MNC_Select = MNC_HBH = MNC_I2E = 1, consistent with the single HBH-scoped and single select-scoped NAS that R0 intends to push.
Likewise, each node can process the PSMHs it sees: the transit nodes at least the HBH-scoped PSMH and the egress node both the HBH-scoped and the I2E-scoped PSMH, i.e., MPC is 1 at R1 and R2 and 2 at R3.
R1 and R3 can process a PSMH that starts up to 4 octets after the BoS, i.e., MPSO = 1, whereas R2 can process a PSMH only when it starts immediately after the BoS, i.e., MPSO = 0.

Applying the derivation of path-wide constraints from {{aggregation}}, R0 determines:

- Path-wide RLD: min(20, 51, 35) = 20 LSEs
- Path-wide MLD_NAS_HBH: min(9, 3, 9) = 3 LSEs
- MLD_NAS_Select for R2: 9 LSEs
- MLD_NAS_I2E: 9 LSEs (from R3)
- Post-Stack MNA is supported on all nodes.
- Path-wide MLD_PSMH for an HBH-scoped PSMH: min(16, 8, 16) = 8 (in 4-octet units).
- MLD_PSMH for an I2E-scoped PSMH: 16 (from R3).
- Path-wide RLD_PSMH: min(36, 59, 51) = 36 (in 4-octet units).
- Path-wide MPSO for an HBH-scoped PSMH: min(1, 0, 1) = 0, so the PSMHs have to start immediately after the BoS and no other post-stack header can precede them.

R0 can now construct a label stack ensuring that all NAS are within each node's RLD and do not exceed the per-scope MLD_NAS and MNC constraints.
The two PSMHs require an In-Stack NAS with the P bit set for each of their scopes ({{psd-isd}}).
For the HBH-scoped PSMH, this is the HBH-scoped NAS that R0 pushes anyway, which leaves the path-wide MLD_NAS_HBH of 3 LSEs for the In-Stack network actions of that scope.
For the I2E-scoped PSMH, R0 has to add an I2E-scoped NAS even though it does not use In-Stack network actions of that scope; a NAS of 2 LSEs carrying the No-Operation opcode with the P bit set suffices and is well within the MLD_NAS_I2E of 9 LSEs at R3.
For Post-Stack MNA, R0 ensures that each PSMH does not exceed the MLD_PSMH of the nodes that process it, that no node receives more PSMHs than its MPC, that no PSMH starts at an offset beyond the MPSO of a node that processes it, and that the combined depth of the label stack and of the PSMHs does not exceed any node's RLD_PSMH.

# Security Considerations
The MNA capabilities defined in this document reveal information about node capabilities, which could potentially be exploited by an attacker to craft targeted attacks against nodes with limited MNA support.
A mechanism that makes these capabilities available SHOULD therefore support configuration options to enable or disable the exposure of MNA capabilities, and SHOULD limit the exposure to within an MNA-capable MPLS domain by default.

A mechanism also has to protect against forged or manipulated capability information, because incorrect capabilities can cause the ingress LER to construct label stacks that a node cannot process, leading to silently dropped network actions, packet loss, or punting of packets to the slow path, and thus to a denial of service against the control plane of an LSR (see {{capabilities}}).
The specific security considerations depend on the mechanism and are therefore left to the document that specifies it.
The security considerations of {{!rfc9994}} and {{?rfc9789}} also apply.

# IANA Considerations
This document has no IANA actions.

--- back
