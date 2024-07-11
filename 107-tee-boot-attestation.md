<!--
**Note:** When your enhancement is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Create an issue in keylime/enhancements**
  When filing an enhancement tracking issue, please ensure to complete all
  fields in that template.  One of the fields asks for a link to the enhancement.  You
  can leave that blank until this enhancement is made a pull request, and then
  go back to the enhancement and add the link.
- [ ] **Make a copy of this template.**
 name it `NNNN-short-descriptive-title`, where `NNNN` is the issue number (with no
  leading-zero padding) assigned to your enhancement above.
- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary", and "Motivation" sections.
  These should be easy if you've preflighted the idea of the enhancement with the
  appropriate SIG(s).
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the enhancement clarified and merged quickly.  The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.
-->
# enhancement-107: TEE Boot Attestation

<!--
This is the title of your enhancement.  Keep it short, simple, and descriptive.  A good
title can help communicate what the enhancement is and should be considered as part of
any review.
-->

<!--
A table of contents is helpful for quickly jumping to sections of a enhancement and for
highlighting any additional information provided beyond the standard enhancement
template.
-->

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (optional)](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Notes/Constraints/Caveats (optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [keylime/enhancements] referencing this enhancement and targeting a release**.

For enhancements that make changes to code or processes/procedures in core
Keylime i.e., [keylime/keylime], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

- [ ] Enhancement issue in release milestone, which links to pull request in [keylime/enhancements]
- [ ] Core members have approved the issue with the label `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place
- [ ] User-facing documentation has been created in [keylime/keylime-docs]

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

## Summary

<!--
This section is incredibly important for producing high quality user-focused
documentation such as release notes or a development roadmap.  It should be
possible to collect this information before implementation begins in order to
avoid requiring implementers to split their attention between writing release
notes and implementing the feature itself. Reviewers
should help to ensure that the tone and content of the `Summary` section is
useful for a wide audience.

A good summary is probably at least a paragraph in length.
-->

Trusted Execution Environments (TEEs) are a confidential computing and
virtualization technology that allow for the protection/encryption of guest VM
RAM, cache memory, and CPU registers for running sensitive applications on
potentially untrusted hosts within the cloud/edge. With TEEs, data written to
RAM, caches, or registers are encrypted with keys maintained by secure
processors embedded within a CPU, thus protecting confidential VM guests from
other guests on a host, or even the host system itself. With encryption fully
managed by the hardware, only the confidential VM (CVM) itself would be able to
read/write to its own memory, protecting against buggy/malicious hosts spying
on or tampering with the CVM.

TEEs are for protecting applications/users on untrusted platforms. In-fact, the
confidential computing threat model assumes that the host in which a CVM is
running on top of is actively attempting to tamper or spy on guest memory. With
this, users must *ensure* that a host has launched their guest with all proper
TEE protections and didn't perform any other nefarious actions to tamper with
a CVM. That is, CVMs must *attest* their boot environment and prove the
*integrity* of their workload before any sensitive operations can be performed
within a CVM.

This enhancement will introduce the notion of workloads using TEE secure
processors as their root-of-trust, rather than TPM devices. With this, TPM state
will be derived from successful TEE attestation reports, and certain TPM PCRs
will be extended to reflect successful TEE attestation.

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->

As CVMs are deployed on untrusted systems, it is reasonable to assume that a
CVM user would also like to take advantage of system integrity monitoring
provided by keylime *in addition to* TEE technology. That is, it is reasonable
to expect that keylime agents will be deployed on CVMs. With that, TEE
attestation support for environments in which keylime will already be running
for system integrity monitoring purposes will likely be desired.

### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->
 * Extend keylime registrar to perform TEE attestation to bootstrap an agent's
   TPM state.
 * On CVMs, extend TPM measurements to account for TEE boot attestation within
   keylime.
 * Fully support confidential computing systems' need for boot *and* runtime
   attestation within keylime.

### Non-Goals

<!--
What is out of scope for this enhancement?  Listing non-goals helps to focus discussion
and make progress.
-->
 * TODO

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation.  The "Design Details" section below is for the real
nitty-gritty.
-->

### User Stories (optional)

<!--
Detail the things that people will be able to do if this enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system.  The goal here is to make this feel real for users without getting
bogged down.
-->

With TEE secure processors acting as the ultimate root-of-trust, much of the
registration and attestation process's behavior should be identical to that of
if TPM devices were the root of trust. However, some registration steps will be
handled before the a keylime agent is initialized.

It is critical to establish trust in a CVM as early as possible. As such,
typical CVMs encrypt their root disks, and require a key stored on an
attestation server to unlock the root disk and begin running any operating
systems or applications on the CVM. To get this key from the server, a CVM would
have to fetch its TEE evidence from the secure processor and send it as input to
the server. If the server is able to validate the TEE evidence, it will then
release the key. At this point, CVMs could decrypt their root disks and begin
running their OS and applications trusting that they are in-fact running
confidentially.

[SVSM](https://github.com/coconut-svsm/svsm) is a firmware that runs in CVMs and
is responsible for performing TEE attestation. SVSM performs TEE attestation to
unlock the CVM's root disk and establish the initial vTPM state. SVSM will
perform the initial communication with the keylime registrar.

TPMs will be present to ensure the integrity of software running on a CVM (i.e.
runtime attestation of CVMs). Furthermore, initial vTPM state will be used to
derive the key to unlock the CVM root disk. Specifically, encrypted VM images
will be sealed against the initial vTPM state of the CVM. The vTPM state key
will then be encrypted with a TEE attestation key stored in the keylime
registrar. Upon CVM boot, the CVM will have access to the encrypted vTPM state
key. To unlock the root disk, the CVM will need the decrypted vTPM state key.
The only way for the CVM to decrypt its vTPM state is to attest its TEE evidence
with the registrar. Upon successful TEE attestation, the registrar will decrypt
the vTPM state and send it back to the CVM. With this, vTPM state is tied to TEE
attestation, and vTPM state will need to be available to run any workloads on
CVMs. Thus, TEE attestation will be required for running any OS/applications and
initializing TPM state on a CVM.

#### Story 1

The use-case of this can be broken up into two main phases, being pre-boot TEE
attestation and post-boot TPM attestation.

Pre-boot TEE attestation is performed from SVSM to unlock the CVM's root disk
and establish initial vTPM state for the machine.

Post-boot TPM registration is performed by the keylime agent and no functional
changes are required. However, the agent enrollment policy will need to account
for TEE attestation (as will be explained).

Pre-boot Attestation
--------------------

1. Pre-boot VM Provisioning

As mentioned before, CVM root disks are encrypted, with decryption reliant upon
successful TEE attestation. In the VM provisioning phase, a user provisions an
encrypted VM image sealed with vTPM state, which is in-turn encrypted with a key
stored within keylime. This step involves a new keylime registrar endpoint to
fetch the public components of its "TEE attestation key" to then encrypt the
vTPM state.

- User creates an encrypted disk image instance from a plain text image
  template.
- User creates vTPM state for the CVM.
- User seals encrypted disk image key against the vTPM state.
- User makes POST request to new keylime registrar endpoint to fetch TEE
  attestation key. Keylime registrar replies with public components of TEE
  attestation key.
- User encrypts vTPM state with public TEE attestation key.
- User stores encrypted vTPM state along with encrypted root disk.

2. Pre-boot VM TEE Attestation

With the image created and the CVM now running, SVSM is able to attest the CVM's
TEE evidence, unlock its vTPM state, and decrypt its root disk. This step
involves another new registrar endpoint to perform attestation on the TEE
evidence.

- SVSM fetches the TEE attestation report from the secure processor.
- SVSM generates a key pair to receive encrypted results from the attestation
  server without the host being able to read/snoop on the results.
- SVSM hashes the public key pair into a part of the TEE attestation report.
- SVSM POSTs the pre-boot attestation data (TEE report, key pair, wrapped disk
  key) to the keylime registrar's TEE attestation endpoint.
- Registrar attests TEE evidence.
- If TEE attestation successful, registrar decrypts disk key, re-encrypts the
  disk key with the SVSM key, and sends the re-encrypted key back to SVSM along
  with a success message. The registrar will also extend one of the vTPMs PCR
  registers (PCR 0 perhaps, although this is up for debate) to show proof of TEE
  attestation within the vTPM state itself.
- If unsuccessful, registrar informs SVSM of attestation failure. SVSM cannot
  decrypt root disk.

Post-boot Attestation
---------------------

With the CVM's OS booted and keylime agent started, no functional changes are
requried by the agent. The agent has a view of the vTPM and will begin the
keylime registration/activation/enrollment/runtime attestation phases as normal.

However, recall that upon a successful pre-boot TEE attestation, the vTPM state
will be extended (via a PCR) to show proof of successful pre-boot TEE
attestation. The agent enrollment policy must account for this PCR extension and
check for it during the enrollment phase to ensure that the agent's pre-boot
environment was attested.

### Notes/Constraints/Caveats (optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above.
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->
 * TODO

### Risks and Mitigations

<!--
What are the risks of this proposal and how do we mitigate.  Think broadly.
For example, consider both security and how this will impact the larger
enhancement ecosystem.

How will security be reviewed and by whom?
-->

Keylime will store a TEE attestation key used to encrypt initial vTPM state for
all encrypted guests attesting with the server. If this key is leaked, all of
the images could potentially be decrypted and secrets can be leaked.

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->


### Test Plan

<!--
**Note:** *Not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy.  Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).
-->

 * New tests will need to be written specifically for TEE boot attestation
   extension scenarios.

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->

TODO

### Dependency requirements

<!--
If your new change requires new dependencies, please outline and demonstrate that your selected dependency 
is well maintained and packaged in Keylime's supported Operating Systems (currently Debian Stable
and as of time writing Fedora 32/33). 

During code implementation you will also be expected to add the package to CI , the keylime ansible role and 
keylimes main installer (`keylime/installers.sh`).

If the package is not available in the supported Operated systems, the PR will not be merged into master. 

Adding the package in `requirements.txt` is not sufficent for master which is where we tag releases from. 

You may however be able to work within an experimental branch until a package is made available. If this is
the case, please outline it in this enhancement.

-->
No additional dependencies should be required.

## Drawbacks

<!--
Why should this enhancement _not_ be implemented?
-->
No drawbacks are known of.

## Alternatives

<!--
What other approaches did you consider and why did you rule them out?  These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->
 * There are other TEE attestation servers released now, such as the [trustee]
   service. However, using trustee would require keylime and trustee to work
   in-tandem and communicate with each other. It would also require two separate
   servers to be run for attesting one confidential VM (trustee for boot
   attestation, and keylime for runtime attestation and integrity monitoring).
   Keylime already offers the sufficient runtime services needed for integrity
   monitoring, so it is reasonable for it to include extensions for boot
   attestation rather than require another solution to be deployed alongside it.
   This would also make keylime a one-stop solution for running sensitive CVM
   workloads on the cloud or edge.

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->
No infrastructure changes needed.

[SVSM]: https://github.com/coconut-svsm/svsm/blob/main/README.md
[trustee]: https://github.com/confidential-containers/trustee/blob/main/README.md