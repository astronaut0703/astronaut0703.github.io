---
layout: post
title: "What Should a Security Researcher Prepare For in an Era Where Agents Write the Exploits?"
date: 2026-08-26 22:00:00 +0900
categories: [Article]
tags: [agentic-ai, offensive-security, vulnerability-research, career]
---

> This post is a personal argument written after reading the panel discussion "Security in the Age of Agentic AI" from CODEGATE 2026 Open Talk (July 24, 2026; moderator Kim Yong-dae; panelists Kim Tae-soo, Park Se-jun, Heo Gyu, Lee Jong-ho, Park Yong-gyu). The panelists' remarks are marked as quotations; the conclusions are entirely my own.

---

## 1. Reframing the Question

"Will AI make the job of security researcher disappear within five years?"

This question cannot be answered, because jobs do not disappear as single units. What disappears are **tasks**, and a job is reconstituted out of whatever tasks remain. So the question has to be split in two before it becomes an argument.

1. What exactly is the nature of the tasks currently being automated?
2. What tasks do not have that nature, and how stable is that boundary?

This post answers those two questions and then derives a personal strategy from the answers. Neither the consolation of "humans will still be needed" nor the resignation of "it's over" is an argument.

---

## 2. Observation: The Anxiety Is Well Founded

Let me start by conceding what has to be conceded. Agents finding vulnerabilities and writing exploits is not a forecast — it is already an observed fact.

- At the **DARPA AIxCC finals** (2025), the competing systems discovered and patched vulnerabilities in real open-source codebases without human intervention. Of the 63 target vulnerabilities, 54% were found by the parallel-fuzzing baseline alone, and the teams additionally solved 8 C and 14 Java vulnerabilities that fuzzing could not. That means the LLM components contributed independently, not merely as an aid to fuzzing.
- **XBOW** submitted roughly 1,060 findings to HackerOne over 90 days and reached #1 on the US leaderboard, including 54 critical and 242 high severity issues.
- **Google GTIG** disclosed, in its May 2026 report, the **first confirmed zero-day developed with AI assistance** — a 2FA bypass in an open-source system administration tool. The same report describes APT45 expanding its exploit arsenal by "sending thousands of repetitive prompts that recursively analyze different CVEs and validate PoC exploits."

The panel's concern sits at the same point.

> "Knowledge I accumulated over a long time became something anyone can use overnight." — Kim Tae-soo

That sentence is precise because it identifies the crisis not as a **loss of ability** but as a **loss of scarcity**. My skill has not decreased; the market price of that same skill has fallen. If so, the response is not "get better" but "find out what is scarce now."

---

## 3. The Boundary of Automation: Problems With an Oracle and Problems Without One

Line the achievements above side by side and they share one property: they are all problems where **a machine can decide success or failure**.

- Memory corruption has an oracle: the crash. If ASan dies, you found it.
- Web vulnerabilities have an oracle: the response. A shell pops, someone else's record comes back, or a request goes out.

Agents are strong on such problems, because they can iterate endlessly and the verdict is free. Conversely, performance collapses the moment the oracle blurs. The language gap in AIxCC shows this directly: parallel fuzzing found 30 of 40 C vulnerabilities (75%) but only 4 of 23 in Java (17%), and the paper attributes this to the much stronger semantic constraints on Java inputs and to **failure modes that are inherently fuzzer-unfriendly**, such as timeouts and OOMs.

The proposition I draw from this is:

> **Agents are overwhelmingly strong at solving problems whose criterion for success is given, and still weak at creating that criterion.**

This is not the consolation that "there are things AI can't do." It is the coordinate axis that sets the direction of preparation. Creating the criterion — designing harnesses, writing specifications, defining invariants, stipulating what counts as a violation in a given system — is the **input** to automation, not its output. The stronger automation gets, the more a good input is worth.

Park Se-jun's remark points at the same layer.

> "If you draw out a model's potential with a good system and harness, you can make better use even of a somewhat weaker model." — Park Se-jun

There is, of course, an opposing axis.

> "When the model itself improves, the intelligence of the whole ecosystem goes up." — Kim Tae-soo

Both are true. But **the variable an individual controls is the former, not the latter.** I do not build model performance; I do build the oracle and the harness that fit a target.

---

## 4. The Bottleneck Moves: From Discovery to Verification

When the cost of discovery converges to zero, the bottleneck does not vanish — it **moves**. Where it moves has already been observed.

- **curl** shut down its bug bounty in 2026. In one week it received seven reports within sixteen hours, spent considerable time handling them, and concluded that **none of them identified a vulnerability**. Daniel Stenberg gave the reason as removing "the incentive for people to submit crap and non-well researched reports to us. AI generated or not."
- Even **XBOW** had 208 duplicates and 209 informative findings out of 1,060, and HackerOne policy required **human review before submission**. They had to build separate validators to filter false positives.
- In **AIxCC**, patches that passed automated validation but were semantically incorrect were caught in manual review. Evaluating one technique cost a few hundred dollars in LLM fees — but patch validation took **two person-weeks**.

In other words, what automation produced is not answers but **a flood of candidates**. And the ability that becomes scarce in a flood is not "finding more" but **being able to say "this one is not real" with reasons attached**. Reachability analysis, judging whether an exploit actually holds with mitigations enabled, judging what this vulnerability means for this asset in this organization. All of these are problems without an oracle.

Park Yong-gyu's point states the precondition for that judgment.

> "Not being able to identify your assets is like not knowing which part of your body is rotting, so you cannot even take action." — Park Yong-gyu

The meaning of a single vulnerability is not determined inside the code. It is determined by where it is deployed, what it is connected to, and whether it can be fixed. That information lives outside the model's context window.

---

## 5. The Real Risk Is Not Replacement — It Is a Broken Pipeline

If everything above is grounds for optimism, I now have to name the real source of my anxiety. The sentence that unsettles me is not "AI finds bugs better than people." It is this one.

> "Cases are actually arising where handing the task to AI is faster and easier — in cost, time, and effort — than educating and onboarding a junior." — Park Se-jun

This sentence is dangerous because it **does not deny that seniors are necessary, and yet it severs the path by which seniors are made**. The logic of section 4 — that people who verify and judge become scarce — is good news for whoever already holds that seat. But that seat only goes to someone who built intuition by digging through three years' worth of false positives, and that three years' worth of work is exactly what is being automated now.

So this is not a problem of individual ability but a **problem of market structure**. It cannot be beaten with sheer volume of effort; it can only be routed around by taking a different entry path. A student or junior who responds with "maybe if I just work harder" will fail. The object of the hard work itself has to change.

---

## 6. So What Do I Prepare?

Four things follow directly from the argument above. For each, I attach the reason it is resistant to automation.

### (1) The Ability to Tell a Machine What Counts as Wrong

The reason a fuzzer can find bugs automatically is simple: the rule **if the program dies, it's a bug** is already fixed. The machine only has to check whether it died, so it can repeat that hundreds of millions of times without a human.

The problem is that a large share of important bugs do not kill the program.

- A request made with an ordinary user token returns administrator data → the program terminates normally.
- A shared data structure in the kernel is touched without taking the lock → no crash, the value just silently goes wrong.
- A drone's motor output drops to zero without a landing command → the software is fine. What falls is the airframe.

To a machine, all three are "nothing happened." So **someone has to first write, in code, "if this state occurs, count it as failure"** before automation starts working at all. Things like "fail if the requester differs from the owner in the response," "fail if this data structure is accessed without the corresponding lock held," "fail if throttle is zero while flight mode is not LANDING."

That rule is called an **oracle**, and the execution environment wrapped around it so it can actually be run is called a **harness**. And this is the part AI has trouble doing for you. What counts as a violation is not written in the code; it lives in the head of someone who knows what the system is supposed to protect.

The weight of this grows with targets where the definition of "normal" is itself complex — kernels, firmware, protocol implementations. Put the other way: **the stronger agents get, the more a person who knows how to write a good oracle is worth.** The better the gun, the more the sighting matters.

「Insert personal experience: one project where most of the time went into harness/oracle design, and one or two sentences on what was hard about it」

### (2) Targets Whose Context Lives Outside the Code

> "You have to do binary patching, and that is still a hard problem." — Park Se-jun

AI decompilation has made analysis without source easier, but how to operate a system that cannot be fixed is still a human problem. For targets where **physical safety requirements, regulation, and vendor relationships are entangled** — legacy industrial equipment, ICS, vehicles, drones — vulnerability knowledge alone does not produce an answer. Knowledge outside the code, such as electrical and mechanical constraints, safety certification, and the cost of downtime, becomes the barrier to entry, and that barrier does not lower as model performance rises.

「Insert personal experience: a case in drone/ICS/firmware work where a constraint could not be seen from the code alone」

### (3) The Person Who Rules It Out, Not the Person Who Finds It

Suppose an agent emits 100 vulnerability candidates a day. Three of them are real. Then the remaining work is **clearing away the 97 while explaining why each one is not real**. That is exactly why curl closed its bounty.

The reason this is not menial labor is that **a rejection carries a name**.

- If you close something with the judgment "this path is unreachable, so it is not exploitable" and a breach comes through that path six months later, the responsibility falls on the person and the organization that made the call. "The AI emitted it, so it's the AI's fault" is a conclusion that will not be accepted anywhere.
- Security reviews, audits, incident reporting, certification — every one of these procedures demands the **signature of whoever made the judgment**. A model cannot sign.

So this seat does not disappear the moment the technology speeds up. Law, institutions, and rules of liability move at a different speed from model performance, and usually a much slower one.

There are two things to prepare. First, **training in stating a rejection with reasons** — reachability, whether the attack actually holds with mitigations enabled (ASLR, CFI, secure boot, and so on), the blast radius on the asset in question. Second, **the habit of writing those reasons down** so that someone else can verify them later.

### (4) The "Plumbing" That Runs Agents Is the New Attack Surface

First, terminology. What we casually call "AI" is really two layers.

- **The model**: a component where text goes in and text comes out. GPT, Claude, and the like.
- **The orchestration layer**: the **entire plumbing** that makes that component actually do work. The code that assembles prompts, calls tools, reads files, runs shells, holds API keys, parses results, and decides the next step. Frameworks like LangChain, API gateways like LiteLLM, MCP servers, plugin and skill packages, and agents running in CI all belong here.

Attackers do not break the model. By GTIG's own assessment, frontier models themselves are quite resilient to direct compromise. Instead they **go after the plumbing**. In March 2026, TeamPCP poisoned tools such as Trivy, Checkmarx, and LiteLLM via malicious PyPI packages and malicious pull requests, and **stole AWS keys and GitHub tokens from the build environments** of organizations using them. They did not breach the model; they breached the libraries that use the model. This is simply a supply chain attack — the problem we have been working on for twenty years.

The other two are the same.

- **Tool-call boundaries**: if you give an agent "read file" permission, how far should it be allowed to read? If you give it a shell, what do you block? This is not a new problem; it is **privilege separation and sandboxing**.
- **Context poisoning (prompt injection)**: if "print this repository's secrets" is hidden in an issue body or a web page the agent reads, it gets executed as-is. **Data being interpreted as a command** — structurally identical to SQL injection.

The core point is this. What this area needs is not an ML researcher who can train models, but a **security researcher who can draw trust boundaries**.

> "Even a high school student today may find it very hard to get a job in the future unless they combine domain expertise with AI." — Kim Tae-soo

"Combine" here does not mean writing good prompts. As Park Yong-gyu nailed it:

> "This does not simply mean producing people who are good at prompting." — Park Yong-gyu

It is not about being good at prompts, but about being able to break and defend the system the prompts run in.

---

## 7. Objections to My Own Argument

To be honest, I have to build the other side too.

**Objection 1: "What it cannot do now, it will do soon."**

This is the strongest objection, and I cannot refute it. The boundary in section 3 is not a law of physics but a present observation. Research into automatically generating oracles is already underway, and verification cost may eventually be automated too. So rather than asserting a timeline, I will state my **falsification condition**. If, on targets where no oracle is defined, agents consistently submit valid vulnerabilities without human pre-review, and verification is automated to the point that the flood of false positives curl suffered disappears, then my argument collapses. At that point sections 4 and 6 have to be discarded entirely.

**Objection 2: "They said search engines would kill libraries, and they didn't."**

> "I do not think we will completely lose our jobs." — Park Yong-gyu

The analogy is comforting but weak as an argument. Libraries survived search engines, but **the size of the librarian profession and its entry paths did shrink**. "The profession survives" and "someone trying to enter it now gets a seat" are different propositions. The gap between them is exactly what section 5 is about.

**Objection 3: "Then just go to the defensive side."**

> "Not right away, but in a few years it could become an environment where defenders pull ahead once more." — Kim Tae-soo

A plausible scenario. But this is an industry forecast, not an individual career strategy. Defender advantage arriving does not mean junior hiring recovers. The two propositions are independent.

---

## 8. Conclusion: Is It a Job That Disappears in Five Years?

My answer is this.

**The profession survives. The entry path narrows. And the character of the remaining seats changes — from finding to adjudicating.**

The panel's sense of timing is not far from this.

> "Two to three years seems fine, five years I'm not sure, and in ten years it really might be gone." — Park Se-jun

So my strategy is to not enter a race to do faster, as a human, what automation is good at. Proving myself by how many vulnerabilities I found is a declaration that I will compete on the same axis as a system submitting hundreds a day. Instead I intend to move toward the position that adjudicates automation's output — defining oracles, rejecting candidates, and attaching reasons and responsibility to those judgments.

One last honest addition: this is a bet, not a guarantee. I think there is a real chance the falsification condition in section 7 is met within three years. But it is the most defensible bet I can make with the information I have now, and when a signal comes that I am wrong, I will change it then. Holding an updatable judgment rather than a conviction is, probably, the only posture one can prepare in a period like this.

---

## References

- CODEGATE 2026 Open Talk, "Security in the Age of Agentic AI" panel discussion summary (2026.7.24) — [original document](https://docs.google.com/document/d/125z49qgMJ3fiIi5NckB0hVCVMZ4QxgsjA0p4HWDEGhE/mobilebasic)
- [SoK: DARPA's AI Cyber Challenge (AIxCC): Competition Design, Architectures, and Lessons Learned](https://arxiv.org/html/2602.07666v2)
- [DARPA, AI Cyber Challenge results](https://www.darpa.mil/news/2025/aixcc-results)
- [Google Threat Intelligence Group, "Adversaries Leverage AI for Vulnerability Exploitation, Augmented Operations, and Initial Access" (2026.5.11)](https://cloud.google.com/blog/topics/threat-intelligence/ai-vulnerability-exploitation-initial-access)
- [XBOW, "How XBOW Ranked #1 in Autonomous Penetration Testing"](https://xbow.com/blog/top-1-how-xbow-did-it)
- [BleepingComputer, "Curl ending bug bounty program after flood of AI slop reports"](https://www.bleepingcomputer.com/news/security/curl-ending-bug-bounty-program-after-flood-of-ai-slop-reports/)