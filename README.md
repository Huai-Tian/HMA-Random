# HMA-Random

[简体中文](README_ZH.md) | English | [AI collaboration doc](README_AGENT.md)

HMA-Random is a Zygisk module that stops other apps from enumerating what you have installed. In one sentence: it doesn't hide your apps — it gives them a new identity to live under.

> A separate [README_AGENT.md](README_AGENT.md) carries the full set of design invariants, adjudication records, and red lines. If you are developing this project with an AI assistant, have it read that document first — most of this project's constraints live in documents, not in code.

## The core idea

Defending against app enumeration has long been a game of catch-up: query entry points are scattered across dozens of APIs, bypasses keep appearing (raw binder calls, the native layer, system-uid apps reading files directly), and interception-based projects are forever patching holes.

HMA-Random declines to play that game. At the moment system_server registers a hidden app, the module writes in a randomly generated package name R and equips that new identity with a complete set of mutually consistent credentials: a random signature, a random label and icon, fake component names, and parseable decoy APKs on disk. From then on, "package P is installed" is not a fact anywhere in the system — every query path returns clean results, including paths nobody has thought of yet.

Think of it as witness protection for apps: a new name, a matching fabricated history, and only trusted parties knowing the real one. The essential difference from interception: **they conceal apps from the system; this module makes the system itself an accomplice.**

The cost is keeping the lie self-consistent. The bulk of the engineering is not in the randomization — it is in the word "complete."

## Relationship to HMA-OSS

The HMA in the name marks this as a lineage project: it works on the same problem as [Hide My Applist](https://github.com/Dr-TSNG/Hide-My-Applist) (Dr-TSNG) and [HMA-OSS](https://github.com/frknkrc44/HMA-OSS) (frknkrc44), with a different paradigm.

| Dimension | HMA-OSS | HMA-Random |
|---|---|---|
| Where the defense sits | Query layer: filter results inside the caller's process | Data layer: rewrite the registered facts in system_server |
| What you maintain | A list of interception points (grows with the attack surface) | A list of identity-consistency channels (grows with OS releases) |
| Unknown query paths | A threat | Not a threat |
| system-uid file reads | Hard to cover | Mount-namespace covering + decoys |
| Configuration shape | Interception rules | Identity map + policy |

The two projects share no code, and configurations are mutually incompatible — "can I import HMA rules" is permanently *no*; the configuration models are fundamentally different. Before the first release we will introduce this project to the HMA-OSS maintainers.

## Design

Four mechanisms:

1. **One identity, rolled once.** R, the random signature, and the fake component names are generated once at hiding time and persist across reboots. No per-query randomization (it would tear system state apart), and no casual re-rolling (persistent grants, accounts, and shortcuts are all name-indexed; a re-roll is a different app — the only legal form is a full unhide-and-hide-again).

2. **Zero interception.** The module translates exactly two kinds of processes: the hidden app itself (it must go on believing it is P, or it will misbehave) and allowlisted processes (the launcher and Settings need the real name and icon). Every other process — including vendor apps holding system privileges — gets no hooks at all. They don't need to be intercepted, because the facts they can query are already R.

3. **A complete false identity.** The file layer is the main battlefield: any app can open another app's APK by the exact path PMS hands out (APK extracters needing no root are the proof). So every hidden app has a decoy inside each observer's mount namespace — a real APK, properly signed with the random signature, its manifest saying R, padded to the same order of size as the real app, with a directory structure isomorphic to a genuine install. An observer reconciling PMS metadata, file contents, and signature verification still finds a world without cracks.

4. **The signature gate.** Updates and probes share the install path, and the criterion is cryptographic: only a package signed with the real certificate is recognized as an update, translated, and merged; a forged probe lands as a "fresh install" whose every observable behavior is identical to a device that never had the app. The module notifies you that someone is probing — and never changes the probe's outcome, because any differential handling would reopen the channel.

## Threat model

**In scope**: device-side enumeration defense from the moment of enabling. Package queries, signature and metadata reads, file enumeration and direct reads, process side channels, install probes — ordinary apps and system-uid vendor apps alike all see the randomized identity.

**Out of scope**, stated plainly:

- Server-side records do not vanish. Push tokens, cloud backups, and advertising identifiers recorded "user has P" remotely long before you enabled this module. The promise starts at enablement; it does not reach back.
- Root-level adversaries and physical access are out of scope.
- Anti-detection (mountinfo mount traces, injection traces) is a long-running battlefield inherited from the Magisk ecosystem: we follow, borrow, and try our best, but make no promise. The security story is deliberately written in two layers — identity inversion as a structural promise, anti-detection as a continuing arms race — and the two are never blended.
- Some functionality of hidden apps is impaired; see below.

## Known costs

- **Store updates stop.** The store reads R plus a random signature and concludes "not installed"; in-app self-update still works.
- **Cross-app protocols suffer.** App links need a local bypass; Play billing, and any handshake where the peer verifies the caller's package name, will break — device-side translation cannot fool someone else's server. That is the paradigm's fixed boundary.
- **Maintenance.** PMS internals are refactored with every major Android release; the adaptation layer is this module's most expensive component.
- **Soft-brick risk.** A crash inside system_server is a soft brick. Safe-mode verification is a hard development requirement; on the user side, know how to recover before you install.
- The full residual ledger (the dumpsys side channel, blind spots over non-zygote daemons, probe packages really landing, and more) is maintained in the exposure_register of [README_AGENT.md](README_AGENT.md).

## Status

Blueprint frozen 2026-11-03; currently zero code. This document describes design, not shipped features; implementation progress lives in the pending_work board of README_AGENT.md.

## 🚫 Non-commercial declaration

HMA-Random is a non-commercial project: it exists for personal interest and technical research, and it intends to stay that way.

- **Free forever**: no paid features, no memberships, no subscriptions, no donation entry — and none will be added. The project's direction cannot be changed by money, because it never touches money.
- **The author takes nothing**: no direct or indirect economic benefit is derived from this project; the author runs no sponsorship or fundraising channel, and authorizes none in the project's name.
- **The license is GPL-3.0, no exceptions**: this project is provided under GNU GPL-3.0 (see [LICENSE](LICENSE)), and any use must comply with it in full — distributions must carry the corresponding source, and derivatives must be released under the same license; that is the entire meaning of using this project. The author does not negotiate dual licensing, does not accept commercial exceptions, and does not permit closed-source redistribution; reselling build artifacts in violation of GPL obligations for profit constitutes copyright infringement.
- **The declaration travels with the code**: redistributions must retain this declaration intact, with correct attribution; redistributing it stripped or doctored is not a fork of this project — it is impersonation.
- **Official channels only**: binaries and source are authoritative in this repository and its Releases; builds from any other source are unverified and used at your own risk.

## Disclaimer

- This module is a privacy tool, built to counter app profiling and installed-list collection. It must not be used to evade parental controls, enterprise compliance (MDM), or law enforcement; the consequences of such use are the user's alone.
- Provided under GPL-3.0 with no warranty. Modules that modify system_server behavior carry inherent soft-brick risk; make sure you are comfortable with Magisk safe mode and fastboot recovery before installing.
- Banking and risk-control apps may refuse to run in a modified environment; Play Integrity verdicts are not guaranteed.

## License

[GPL-3.0](LICENSE). You are free to use, modify, and redistribute this project; distributions and derivatives must be released under the same license and keep the source open.

## Acknowledgements

- [Hide My Applist](https://github.com/Dr-TSNG/Hide-My-Applist) (Dr-TSNG) — the founder of this category of work.
- [HMA-OSS](https://github.com/frknkrc44/HMA-OSS) (frknkrc44) — the control group, and a continuing reference on anti-detection.
- The Magisk / Zygisk / Shamiko ecosystem — the entire foundation of the injection and anti-detection layers.
