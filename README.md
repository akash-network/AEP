# Akash Enhancement Proposals (AEP)

Akash Enhancement Proposals (AEPs) describe standards for the [Akash](https://akash.network) Decentralized Cloud platform, including core protocol specifications, client APIs, and SDL standards.

See [INDEX.md](INDEX.md) for an index of all AEPs.

## Roadmap

Akash's [roadmap](https://akash.network/roadmap/2025/) outlines the high-level goals and priorities for the Akash Network.

## Contributing

1. Review [AEP-1](spec/aep-1).
2. Fork the repository by clicking "Fork" in the top right.
3. Add your AEP to your fork of the repository.
4. Submit a Pull Request to Akash's [AEPs repository](https://github.com/akash-network/AEP/tree/main/spec).

Your first PR should be a first draft of the final AEP. It must meet the formatting criteria enforced by the build (largely, correct metadata in the header). An editor will manually review the first PR for a new AEP and assign it a number before merging it. Make sure you include a discussions-to header with the URL to a discussion forum or open GitHub issue where people can discuss the AEP as a whole.

If your AEP requires images, the image files should be included in a subdirectory of the assets folder for that AEP as follows: `assets/aep-N` (where N is to be replaced with the AEP number). When linking to an image in the AEP, use relative links such as `../assets/aep-1/image.png`.

Once your first PR is merged, we have a bot that helps out by automatically merging PRs to draft AEPs. For this to work, it has to be able to tell that you own the draft being edited. Make sure that the `author` line of your AEP contains either your GitHub username or your email address inside . If you use your email address, that address must be the one publicly shown on your GitHub profile.

When you believe your AEP is mature and ready to progress past the draft phase, open a PR changing the state of your AEP to 'Final.' An editor will review your draft and ask if anyone objects to its being finalized. If the editor decides there is no rough consensus - for instance, because contributors point out significant issues with the AEP - they may close the PR and request that you fix the issues in the draft before trying again.

## AEP Status Terms

* **Draft** - an AEP that is undergoing rapid iteration and changes.
* **Last Call** - an AEP that is done with its initial iteration and ready for review by a wide audience.
* **Accepted** - a core AEP that has been in Last Call for at least 2 weeks and any technical changes that were requested have been addressed by the author. The process for Core Devs to decide whether to encode an AEP into their clients as part of a hard fork is not part of the EIP process. If such a decision is made, the AEP will move to final.
* **Final (non-Core)** - an AEP that has been in Last Call for at least 2 weeks and any technical changes that were requested have been addressed by the author.
* **Final (Core)** - an AEP that the Core Devs have decided to implement and release in a future hard fork or has already been released in a hard fork.

## Generating the AEPs Index and Roadmap

To generate the AEPs index and roadmap, run the following commands:

```bash
node scripts/index.js
```

To install dependencies, run: `npm install`
