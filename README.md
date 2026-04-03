# Akash Enhancement Proposals (AEP)

Akash Enhancement Proposals (AEPs) describe standards for the [Akash](https://akash.network) Decentralized Cloud platform, including core protocol specifications, client APIs, and SDL standards.

See [INDEX.md](INDEX.md) for an index of all AEPs.

## Roadmap

Akash's [roadmap](ROADMAP.md) outlines the high-level goals and priorities for the Akash Network. Some AEPs are associated with the roadmap.

## Contributing

1. Review [AEP-1](spec/aep-1) and the Best Practices: Akash Governance Proposals Discussion.
2. Ensure you have adequately vetted your idea and proposal content to make certain that your idea is unique and will better the network and community. 
3. Fork the repository by clicking "Fork" in the top right.
4. Add your AEP to your fork of the repository. There is a [template AEP here](aep-template.md).
5. Submit a Pull Request to Akash's [AEPs repository](https://github.com/akash-network/aeps).

Your first PR should be a first draft of the final AEP. It must meet the formatting criteria enforced by the build (largely, correct metadata in the header). An editor will manually review the first PR for a new AEP and assign it a number before merging it. Make sure you include a discussions-to header with the URL to a discussion forum or open GitHub issue where people can discuss the AEP as a whole.

If your AEP requires images, the image files should be included in a subdirectory of the assets folder for that AEP as follows: `assets/aep-N` (where N is to be replaced with the AEP number). When linking to an image in the AEP, use relative links such as `../assets/aep-1/image.png`.

Once your first PR is merged, we have a bot that helps out by automatically merging PRs to draft AEPs. For this to work, it has to be able to tell that you own the draft being edited. Make sure that the `author` line of your AEP contains either your GitHub username or your email address inside . If you use your email address, that address must be the one publicly shown on your GitHub profile.

When you believe your AEP is mature and ready to progress past the draft phase, open a PR changing the state of your AEP to 'Final.' An editor will review your draft and ask if anyone objects to its being finalized. If the editor decides there is no rough consensus - for instance, because contributors point out significant issues with the AEP - they may close the PR and request that you fix the issues in the draft before trying again.

There are three types of AEP:

A Standard describes any change that affects most or all Akash implementations, such as a change to the network protocol, a change in block or transaction validity rules, proposed application standards/conventions, or any change or addition that affects the interoperability of applications using Akash. Furthermore, Standard AEPs can be broken down into the below categories. Standards Track AEPs consist of three parts, a design document, implementation, and finally if warranted an update to the formal specification.
- Core - improvements requiring a consensus fork, as well as changes that are not necessarily consensus critical but may be relevant to "core dev" discussions. (AEP-6, 8, 9)
- Economics - includes improvements to Akash Network's Economic Model with regards to Staking or Subsidy distribution. (AEP-5, 10, 22)
- Interface - includes improvements around client API specifications and standards and/ or to the primary deployment clients (Akash CLI and Akash Console). (AEP-3, 28, 31)
  
A Meta AEP describes a process surrounding Akash or proposes a change to (or an event in) a process. Process AEPs are like Standards Track AEPs but apply to areas other than the Akash protocol itself. They may propose an implementation, but not to Akash's codebase; they often require community consensus; unlike Informational AEPs, they are more than recommendations, and users are typically not free to ignore them. Examples include procedures, guidelines, changes to the decision-making process, and changes to the tools or environment used in Akash development. Any meta-AEP is also considered a Process AEP. (AEP-4, 7, 16)

An Informational AEP describes an Akash design issue, or provides general guidelines or information to the AEPs community, but does not propose a new feature. Informational AEPs do not necessarily represent AEPs community consensus or a recommendation, so users and implementers are free to ignore Informational AEPs or follow their advice.(AEP-1, is the closest current example of an Informational AEP; Informational AEPs should be considered as minor, whereas, Meta AEPs are major in terms of community guidelines and information)

It is highly recommended that a single AEP contain a single key proposal or new idea. The more focused the AEP, the more successful it tends to be.

## AEP Status Terms

* **Draft** - an AEP that is undergoing rapid iteration and changes.
* **Last Call** - an AEP that is done with its initial iteration and ready for review by a wide audience.
* **Accepted** - a core AEP that has been in Last Call for at least 2 weeks and any technical changes that were requested have been addressed by the author. The process for Core Devs to decide whether to encode an AEP into their clients as part of a hard fork is not part of the EIP process. If such a decision is made, the AEP will move to final.
* **Final (non-Core)** - an AEP that has been in Last Call for at least 2 weeks and any technical changes that were requested have been addressed by the author.
* **Final (Core)** - an AEP that the Core Devs have decided to implement and release in a future hard fork or has already been released in a hard fork.

## AEP Development Walkthrough 

#### 1. Idea Development
You (or your team) have an idea for adding a new feature to the Akash Network. 
You read over AEP-1 and the Best Practices: Akash Governance Proposals Discussion to gather some ideas about how you will define the feature you are attempting to integrate and how you will manage the implementation. You publish your idea in a new discussion in the Akash Network Github and on the Akash Network Discord channel to gather some additional perspective. You may even introduce your idea to the Steering Committee to get first hand feedback on the relevancy and realism of your idea. The community then has the opportunity to engage with you about your idea. You get to see how your proposed feature may affect other existing frameworks and features and learn how original your idea may be.
#### 1.1. Competing for An AEP Proposal 
If you find that an AEP proposal already exists that defines the work that you want to perform, you may begin competing for ownership of the AEP proposal. This can be done by first outlining a discussion with the following content: 

  a. A title pertaining to the AEP proposal you want to assume (“Proposal to solve AEP-X”). 

  b. A contributor summary of the background and skill set of your team or yourself as well as why your background and skills make you a good fit for assuming responsibility of this AEP proposal.
  
  c. A detailed timeline and budget of how you intend to accomplish this AEP proposal in a better fashion than those already performing the work. What makes the process that you will enact different from what is already being done? How can you do it better, in less time, for less expense; and if you cannot, why should the community fund you over the solution it has already chosen? 
  
  d. A detailed specific breakout of the plan you will execute to achieve the end goal of the AEP.
  
  e. Finally, any outstanding questions that need community or higher engineering input. 

Once your reason for assuming another entity's AEP proposal has been heard and assessed by the community, you should present it to the Steering Committee. The Steering Committee will then decide, given all of the information, if you will be allowed to continue competing for the AEP proposal in question. It is important to note, there are many reasons that an AEP proposal may be competed ranging from a group or individual failing to meet the schedule that they publicized in their AEP proposal to improperly utilizing the funding they were granted during their development effort. Sometimes a group may lose an AEP proposal simply because the competition made the community believe they had a better idea for achieving the desired outcome. The goal of any AEP proposal is complete transparency to the community. 

#### 2. Draft AEP Proposal  
You write out a draft AEP proposal using an existing template, like Akash’s “Technical AEP Template”. You determine which areas of the Akash Network you are impacting and how your AEP proposal should be designated (Standard/Core, Standard/Economics, Standard/Interface, Meta, Informational). You assess the considerations your peers posed to you during your discussions, you gather estimations on how much this design and implementation will cost, and provide the benefits and drawbacks of your system, highlighting why it is a necessary addition to the Akash Network. You generate 4-6 comprehensive milestones that you plan to complete by specific dates, this will both better help you estimate the amount of work you have to undertake to complete your design and integration effort, and help the community and administration track your progress as you move forward after you have been funded. You assess the “Prepare your Prerequisites” section of the Best Practices Discussion to confirm that your AEP proposal meets all of the existing standards. Finally, you fork a new AEP repository, enter your AEP proposal into your fork, and submit a Pull Request (PR) that the editor and, separately, the Steering Committee can review and discuss in their monthly meeting. Ultimately, it is the choice of the Steering Committee to merge your fork and your AEP proposal into the main repository, or if it needs more work. Your AEP proposal is in the Draft stage (as is reflected in the header of your document), during which the community will provide official criticisms and commentary on the state and content of your AEP proposal. 
During the Draft stage, the community has the opportunity to directly contribute to your AEP proposal by forking your forked repository and submitting their own PRs to you. Many different vectors contribute comments and commits to your work, essentially, a commit is a PR on a PR, a way for the people reviewing your Draft to get their considerations directly in front of you by submitting and updating their own PRs to your PR.   
#### 3. Pull Request
After you have implemented the changes and edits posed by the community and refined your AEP proposal so that it meets the best standard you can attain, you submit an update to your PR and ensure that there are no more changes to be made. Your AEP proposal undergoes a 14-day review period. Your AEP proposal is in the Last Call phase. A goal is for your AEP proposal to only enter Last Call once, so be sure it meets all of the considerations and standards that were brought up during the Draft phase. 
#### 4. Acquiring Funding and On-Chain Voting 
When your AEP proposal has completed its review phase and the community and editors are satisfied with the content and structure of the AEP proposal, it can be submitted on-chain for voting, assuming that you require community pool funding to integrate your feature. The validators and stakeholders will vote whether or not to approve or reject your AEP proposal. Should your AEP proposal be approved, you will be granted funding and you will submit a new PR to move your AEP proposal from “Last Call” to “Final” (again, as is reflected in the header of your document). From here, your fork will be merged with the Main AEP Repository, confirming its validity, and you may begin work implementing your idea. 

## Generating the AEPs Index and Roadmap

To generate the AEPs index and roadmap, run the following commands:

```bash
node scripts/index.js
```

To install dependencies, run: `npm install`
