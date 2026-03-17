---
# Page settings
layout: default
keywords:
comments: false

# Hero section
title: Feed reranking
description: Resources to modify the feed on social media.

# Micro navigation
micro_nav: true

---

## Overview

This page collects resources for running **real-time feed reranking experiments** on social media using browser extensions.  
The goal is to help researchers:

- Intercept and log live feeds from platforms such as X/Twitter.
- Apply custom reranking or content editing logic with minimal latency.
- Deploy ecologically valid field experiments on participants’ real feeds, without platform collaboration.
- Connect feed interventions to survey measures and behavioral data.

---

## Code: FeedMonitor

**Repository**

- **GitHub:** [StanfordHCI/FeedMonitor](https://github.com/StanfordHCI/FeedMonitor)

FeedMonitor is an open-source reference implementation of a browser extension and backend for feed experiments on X/Twitter. It is designed as a starting point that you can fork and adapt to your own interventions.

**What you get**

- **Browser extension skeleton**
  - Intercepts network calls that power the feed.
  - Restructures responses before they reach the page, so you can modify content server-side or client-side.
  - Works with the website’s regular interface, so participants continue to use the platform as usual.

- **Reranking pipeline**
  - Extracts posts from the feed payload.
  - Sends them to a scoring component (e.g., LLM API, custom classifier, or heuristic).
  - Reorders or filters posts according to your experimental objective (up-ranking, down-ranking, or more complex logic).

- **Data collection hooks**
  - Logging of impressions and scrolling.
  - Optional logging of engagement events (likes, opens, clicks), when permitted by your IRB protocol.
  - Support for matching feed events to survey responses.

- **Integration points**
  - Backend endpoints for scoring and logging.
  - Hooks to insert **in-feed surveys**, prompts, or other widgets.

Use this repo as an implementation template, and customize:

- The **selection logic** (which posts are targeted).
- The **scoring function** (keywords, models, LLMs, etc.).
- The **intervention** (reranking, hiding posts, inserting new posts, editing text or metrics).
- The **measurement strategy** (momentary, longitudinal, or behavioral).

---

## Paper reference

### Reranking Social Media Feeds: A Practical Guide for Field Experiments

**Citation**

> Tiziano Piccardi, Martin Saveski, Chenyan Jia, Jeffrey Hancock, Jeanne L. Tsai, Michael S. Bernstein.  
> *Reranking Social Media Feeds: A Practical Guide for Field Experiments*. arXiv:2406.19571, 2024. :contentReference[oaicite:0]{index=0}

**Read the full paper**

- [arXiv:2406.19571](https://arxiv.org/abs/2406.19571)

---

### What the guide covers

The paper provides a practical playbook for researchers who want to run **independent field experiments on social media feeds** using browser extensions, without needing platform cooperation. It covers:

- **Experiment paradigm**
  - Use browser extensions to intercept feed API responses or DOM content in real time.
  - Apply re-ranking or content editing before the user sees the feed.
  - Keep the interface as close as possible to the original platform, to maintain ecological validity. :contentReference[oaicite:1]{index=1}

- **Intervention types**
  - **Down-ranking**: push certain content further down the feed, or remove it completely (for example, harmful or unwanted content).
  - **Up-ranking**: increase exposure to specific content (for example, out-party posts, positive emotions, civic information). The paper discusses how to approximate the platform’s inventory and the limitations when you only see the already ranked subset of posts. :contentReference[oaicite:2]{index=2}
  - **Content editing**: manipulate visible fields on posts (text, social metrics, attachments, warnings) to study how different designs or framings change user responses. :contentReference[oaicite:3]{index=3}

- **Measurement strategies**
  - **In-feed Ecological Momentary Assessments (EMAs)**: embed short survey widgets inside the feed to capture real-time reactions, such as perceived interest, emotions, or perceived quality.
  - **Pre/post or repeated surveys**: measure outcomes like affective polarization, opinion change, or attitudes before and after the intervention, or at regular intervals during a study.
  - **Behavioral signals**: log detailed interaction traces (time on page, scrolling, likes, shares, link clicks) to understand trade-offs between outcomes of interest and engagement. :contentReference[oaicite:4]{index=4}

- **Implementation details**
  - How to override and extend the browser’s `XMLHttpRequest` (or equivalent) to intercept feed responses.
  - How to structure a **participant registration and onboarding flow** that:
    - Connects survey platforms, consent, and recruitment sources.
    - Passes participant IDs into the extension through a coordinating server.
  - How to design notification patterns (in-page messages, emails, banners) and handle error recovery.
  - Privacy and ethics considerations, including consent flows, secure storage, and uninstallation or automatic deactivation at the end of the study. :contentReference[oaicite:5]{index=5}

---

## Typical architecture at a glance

A standard reranking experiment based on the paper and FeedMonitor follows these steps:

1. **Intercept the feed**
   - The extension captures network responses (or DOM) that contain the feed.
2. **Select posts of interest**
   - Identify posts relevant to your intervention (for example, political, toxic, positive emotion, etc.).
3. **Score posts**
   - Apply a scoring model (rule-based, ML, or LLM) aligned with your experimental objective.
4. **Apply intervention**
   - Reorder, hide, insert, or edit posts according to the scores and the experimental condition.
5. **Render updated feed**
   - Replace the original feed data with the modified version, while preserving a smooth user experience and minimizing latency.
6. **Measure outcomes**
   - Log exposure and behavior; optionally insert EMAs and connect to pre/post surveys.

The paper and repository together give you a concrete, working example of this pipeline that you can adapt to other platforms and interventions.

---

## When to use these resources

Use this stack if you want to:

- Run **feed-level interventions in the wild**, on participants’ real accounts.
- Study the impact of ranking choices on:
  - affective polarization
  - emotional experience
  - exposure to certain topics (for example, politics, misinformation, prosocial content)
  - engagement and time use
- Prototype **alternative feed objectives** (for example, democratic values, well-being, prosocial behavior) on top of existing platforms.
- Build a reusable infrastructure for future feed experiments by your group or collaborators.

---

## How to cite

If you build on this work (paper, code, or design), please cite:

```bibtex
@article{piccardi2024reranking,
  title   = {Reranking Social Media Feeds: A Practical Guide for Field Experiments},
  author  = {Piccardi, Tiziano and Saveski, Martin and Jia, Chenyan and Hancock, Jeffrey and Tsai, Jeanne L and Bernstein, Michael S},
  journal = {arXiv preprint arXiv:2406.19571},
  year    = {2024}
}
