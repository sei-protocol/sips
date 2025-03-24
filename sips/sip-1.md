**SIP-1: Sei Improvement Proposal Process**

| SIP-Number | 1 |
| ----- | ----- |
| Title | SIP Guidelines |
| Description | A set of guidelines and requirements for the submission and lifecycle of SIPs. |
| Author | Chad Hugghins \[chad@seifdn.org\] |
| Type | Process |
| Created | 2025-03-21 |
| Status | Living |

## **What is an SIP?**

SIP stands for Sei Improvement Proposal. An SIP is a design document that details a new feature for Sei, its processes, or the ecosystem. It provides the basis for the Sei community to propose a new idea and to assess its merits, its feasibility, and the implementation details.

The goal of the SIP project is to standardize and provide high-quality documentation for Sei and its ecosystem. SIPs serve as the primary mechanism for proposing new features, collecting community input on issues, and documenting the design decisions that have gone into Sei.

Anybody can create an SIP. However, it is strongly encouraged to discuss the potential SIP with the Sei community before creating a proposal. The SIP process is not intended to sort and select the best ideas, but to assess the feasibility of each SIP on its own merit. By seeking feedback from the community before submitting, the author can gauge the extent to which the SIP is needed, and avoid wasting time on unnecessary SIPs.

In addition to the proposal itself, SIPs track the implementation status through the "Implemented" and "Activated" statuses, providing visibility into the complete lifecycle from idea to production deployment.

## **Sei Protocol Overview**

Sei is a high-performance, low-fee, delegated proof-of-stake blockchain designed for developers. It supports optimistic parallel execution of both EVM and CosmWasm, opening up new design possibilities. With unique optimizations like twin turbo consensus and SeiDB, Sei ensures consistent 400ms block times and a transaction throughput that's orders of magnitude higher than Ethereum. This means faster, more cost-effective operations. Plus, Sei's seamless interoperability between EVM and CosmWasm gives developers native access to the entire Cosmos ecosystem, including IBC tokens, multi-sig accounts, fee grants, and more.

Sei introduces four major innovations:

1. **Twin Turbo Consensus**: This feature allows Sei to reach the fastest time to finality of any blockchain at 400ms, unlocking web2-like experiences for applications.  
2. **Optimistic Parallelization**: This feature allows developers to unlock parallel processing for their Ethereum applications, with no additional work.  
3. **SeiDB**: This major upgrade allows Sei to handle the much higher rate of data storage, reads and writes which become extremely important for a high-performance blockchain.  
4. **Interoperable EVM**: This allows existing developers in the Ethereum ecosystem to deploy their applications, tooling and infrastructure to Sei with no changes, while benefiting from the 100x performance improvements offered by Sei.  
1. 

## **SIP Type**

There are three types of SIP, with procedural differences between each:

### **Standard**

A Standard SIP presents a feature which will impact most implementations of Sei, and therefore requires community consensus. These SIPs include consensus changes, networking protocol changes, RPC changes, EVM/CosmWasm changes, and smart contract standards. Standard SIPs must also specify a SIP category.

### **Process**

A Process SIP presents a change that does not impact the Sei protocol itself, but still requires some level of community consensus. These include developer tooling, node and validator procedures, and changes to the SIP process itself.

### **Informational**

An Informational SIP presents guidelines or information to the community without proposing a specific change or feature. Such SIPs do not result in changes to the Sei protocol or development environment and therefore do not require community consensus for approval.

## **SIP Category**

A SIP category must be specified when the SIP type is Standard, but not otherwise.

There are five categories of SIP:

1. **Core**: Changes or additions to core features of Sei, including consensus, execution, storage, and account signatures  
2. **Networking**: Changes or additions to Sei's mempool or network protocols  
3. **Interface**: Changes or additions to RPC specifications or lower-level naming conventions  
4. **Framework**: Changes or additions to EVM and CosmWasm contracts and primitives included within the Sei codebase  
5. **Application**: Proposals of new EVM/CosmWasm standards or primitives that would not be included within the Sei codebase but are of significant interest to the developer community

## **SIP Workflow**

Follow the contribution guidelines to submit a SIP. The person who submits a SIP is automatically the SIP Author. A SIP Editor will be assigned to the SIP and will assess the formatting and structure. If the SIP is correctly formulated:

1. The status will be moved to Draft and a SIP number will be assigned. At this point, it has officially entered the SIP process.  
2. The SIP Editor will create an official comments thread and update the Comments-URI header field to point to this thread.  
3. The SIP Editor will guide the SIP Author through the process, including how to gather feedback from all relevant stakeholders. See the SIP Author section for more details.

## **How to Contribute**

To contribute a SIP:

1. Read SIP-1 (this document).  
2. Fork the SIPs repository.  
3. Copy the SIP template, using a temporary filename in the format `sips/sip-temporary_title.md`, into the sips directory.  
4. Fill in the template, ensuring you follow the comments inside.  
5. Assets may be embedded into the SIP. Include these assets in the assets directory, in a subdirectory named `sip-temporary_title`.  
6. Submit a Pull Request to the SIPs repository from your forked repository.

### **Next Steps**

If your PR is correctly formulated, the status will be moved to Draft and a SIP number will be assigned. At this point, it has officially entered the SIP process. Continue to follow the instructions in SIP-1 as your SIP makes its way through the process.

## **SIP Status**

There are ten official statuses for a SIP. The typical progression for a successful SIP would be: Draft → Review → Last Call → Final → Implemented → Activated.

Show Image

### **Idea**

This is not an official status, but represents the SIP before it has been assigned a status. At this stage, the SIP Author writes the proposal and submits it as a PR to the SIP repository. If the SIP's formatting and structure do not meet the requirements when it is assessed, it will remain without a status until these issues have been addressed.

### **Draft**

Once the formatting of the SIP meets the requirements, a SIP Editor will change the status to Draft and fill in other necessary fields, including the name of the SIP Editor. The SIP has now formally entered the process.

At this point, it is the responsibility of the SIP Author to present their proposal to the community. The SIP Editor will help the SIP Author find the appropriate avenues of discussion, which may include forums, community calls, and/or presentations.

It is also important in the Draft stage to consider which stakeholders the SIP will affect, in particular those who would be required to implement it, and to seek their feedback at the earliest stage.

It is the SIP Author's responsibility to ask the SIP Editor to assess whether the SIP is ready to be moved to Review status. The SIP Editor will not actively monitor this progress. There is a time factor involved in making this decision. The more significant and consequential the SIP is, the longer is needed before the SIP is ready to move to Review.

### **Review**

If the SIP Editor is satisfied that comments and feedback from the community and relevant stakeholders have been adequately addressed during the Draft stage, the SIP moves into the Review stage.

Once the SIP reaches Review, reviewers will be assigned to the PR in order to perform an official review. The SIP Author may request such reviewers, but the SIP Editor is responsible for deciding if they meet the requirements for an official review. A minimum of three reviews are required, with all feedback addressed, for the SIP to proceed to Last Call.

Generally, there is no minimum time period for SIPs to remain in Review status, but the SIP Editor may decide to enforce one for a particular SIP. In this case, the minimum time period will be stated when the status is changed to Review.

### **Fast Track**

In some circumstances, a SIP may be moved immediately into Fast Track status instead of Draft. This is entirely at the SIP Editor's discretion and is reserved for proposals that have already reached community consensus outside of the SIP process.

Once a SIP reaches Fast Track status, the process is the same as the Review stage.

### **Last Call**

Once a SIP has been successfully reviewed and all feedback has been addressed and accepted, the SIP moves into Last Call. Typically, this stage lasts a minimum of one week without further comment, but this duration is at the discretion of the SIP Editor and will be clearly stated.

The purpose of this stage is to give a final chance for dissenting opinions to be aired and to give time for any feedback that has been addressed in the Review stage to be digested by all parties.

### **Final**

The SIP has now been accepted. It represents a specification that the community has reached consensus on as being technically sound and beneficial for the Sei ecosystem. Every accepted SIP should have an associated tracking issue in the Sei repository. If the implementation requires the feature to be activated using a feature flag, a feature-gate issue for tracking across networks should also be created.

While it is not necessary for the SIP author to also write the implementation, it is by far the most effective way to see a proposal through to completion: authors should not expect that other project developers will take on responsibility for implementing their accepted feature.

Once a SIP has reached Final status, it cannot be withdrawn or modified in any way. A further SIP can be submitted to extend or replace features in the original SIP, or the standard could simply stop being used (e.g., for Application or Informational SIPs).

### **Implemented**

When all relevant teams have completed development of the SIP's feature, the SIP status changes to "Implemented". This status indicates that the technical work is complete but the feature has not yet been activated on the Sei mainnet.

### **Activated**

A SIP will have the status Activated once it has been implemented, tested, and finally activated on Sei mainnet. This status represents the completion of the SIP's journey from idea to production deployment.

### **Stagnant**

A SIP will move into Stagnant status if it remains inactive in Draft, Review, or Last Call status for four weeks. Once a SIP reaches Stagnant status, it is not usually able to be revived, and instead should be withdrawn and resubmitted as a new SIP. However, in exceptional circumstances, the SIP Editor may bring a Stagnant SIP back to its prior status if requested by the SIP Author.

### **Withdrawn**

The SIP Author may withdraw their SIP at any stage.

### **Living**

Living is a special status reserved for Process SIPs that will never reach Final status, such as this SIP.

## **SIP Author**

The SIP Author refers to the person or people that submit the SIP. The SIP Author is responsible for:

* Driving the process forward at all stages  
* Bringing the proposal to the attention of the relevant stakeholders, particularly in cases where they will be required to implement the proposal  
* Encouraging open debate and discussion  
* Addressing and resolving comments and feedback  
* Withdrawing a SIP when it is clear that the community cannot reach consensus or it is not feasible to implement

## **SIP Editor**

Although the SIP Author is responsible for driving the SIP, the SIP Editor is available at all stages to support the SIP Author. They can help the SIP Author to understand what is required at each stage, connect with appropriate stakeholders, and open community discussions through the appropriate mediums.

The current register of SIP Editors is as follows:

| Name | GitHub username | Email address | Affiliation |
| ----- | ----- | ----- | ----- |
| \[Editor 1\] | \[username1\] | \[email1\] | Sei Labs |
| \[Editor 2\] | \[username2\] | \[email2\] | Sei Foundation |

## **SIP Repository Access Policy**

The SIP repository has three levels of access:

1. **Triage**: Contributors with triage access can label and close issues, but cannot merge code.  
2. **Write**: Contributors with write access can review and merge PRs for SIPs.  
3. **Maintain**: Maintainers have full administrative access to the repository.

SIP Editors will typically have Write or Maintain access. Access to the repository is granted based on demonstrated contribution to the Sei ecosystem and technical expertise. To request access or report misuse, please contact one of the current SIP Editors.

## **SIP Comments**

Each SIP will have a Comments-URI header field set by the SIP Editor when a status is assigned. This location should be used as the primary place for official discussions about the SIP.

## **SIP Format**

The contribution guidelines provide a template, which should be used as the basis for creating the SIP.

### **Header**

#### **Title**

The Title field should:

* Follow AP style headline case, which capitalizes most words  
* Be no more than 64 characters  
* Not end in a period

#### **Description**

The Description field should:

* Be a single, informative sentence  
* Be no more than 160 characters

### **Linking to Other SIPs**

Other SIPs may be referenced in the format SIP-N where N is the number of the SIP being referenced. The first time a SIP is referenced within the document, it must be hyperlinked using a relative Markdown link. Subsequent references may also be linked but this is not required.

### **Linking to External Resources**

External links should not be included, except to the Sei repository.

### **Embedding or Linking to Assets**

Assets (including images, code, and data) may be embedded into or linked from the Markdown. Such assets should be included within the assets directory, in a subdirectory created for the SIP.

## **History**

This SIP has built on key concepts from both Bitcoin's BIP-0001, Ethereum's EIP-1, and Sui's SIP-1, and also Solana’s SIMD-1.

## **Copyright**

CC0 1.0.
