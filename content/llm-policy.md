---
title: Generative AI Policy
date:  "2026-09-08T09:00:00+10:00"
draft: false
aliases:
  /slop
---

# Generative AI (LLM) Policy
Generative AI tools using large language models (LLMs) are widespread
in software development today. While we acknowledge that these tools
do have uses cases, LLMs engender harms to the pillars of Asahi Linux:
software freedom, the open-source community, the open Internet, the
environment, digital privacy, user consent, and software developers
themselves. For more information on the ethical issues of LLM use
in open source, we recommend reading [SourceHut's policy](https://sourcehut.org/blog/2026-08-27-tos-changes-and-llms/)

Furthermore, Asahi Linux relies on [clean room reverse engineering](https://asahilinux.org/copyright)
to ensure our work is legal. We have strict guardrails around
binary disassembly and decompilation, and we absolutely forbid
the use of leaked materials. While general questions around
LLMs and copyright remain unsettled, LLMs pose unique legal risks to
reverse engineering projects, as these systems are likely to violate
the clean room requirements and taint the resulting code. This risk
is amplified with "agentic" approaches, where the human may
be unaware of the reverse engineering process. In reverse engineering
and science _how_ knowledge is obtained is just as important as the
knowledge itself.

Due to these legal and ethical issues, we broadly forbid the use of
generative AI tooling for material contributions to Asahi Linux.
Enforcement may vary depending on the seriousness of the infraction.
A GitHub issue drafted by an LLM may simply be closed with reference
to this policy. A contributor using an LLM to interpret a trace
made using the m1n1 hypervisor may be issued with a first and final
warning. A contributor found to have concealed extensive LLM use will
be banned immediately, particularly if they may have accessed unreleased
Apple material.
