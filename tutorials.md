---
# Page settings
layout: default
keywords:
comments: false

# Hero section
title: Tutorials
description: List of resources and tutorials focused on social media experiments.

# # Author box
# author:
#     title: About Author
#     title_url: '#'
#     external_url: true
#     description: Author description

# Micro navigation
micro_nav: true

# Page navigation
# page_nav:
#     prev:
#         content: Previous page
#         url: '#'
#     next:
#         content: Next page
#         url: '#'
---

## IC2S2 2026

**Social Media Feed Ranking Algorithms: Guide to Field Experiments**  
Tiziano Piccardi (Johns Hopkins University) · Martin Saveski (University of Washington)  
July 28, 2026 · Burlington, Vermont, US

Coming soon...

---

## ICWSM 2025

**Social Media Feed Ranking Algorithms: Guide to Field Experiments**  
Tiziano Piccardi (Stanford University) · Martin Saveski (University of Washington)  
June 23, 2025 · Copenhagen, Denmark

→ [Tutorial website](https://socialmediafeed25.github.io/)

### About

A four-hour hands-on tutorial introducing researchers and students to the theory and practice of feed ranking experiments. The tutorial covers how feed ranking algorithms work, how to run experiments on closed platforms using browser extensions and middleware, how to design and analyze randomized controlled trials on social media feeds, and how to build custom feeds on open platforms like Bluesky.

The tutorial is organized around a concrete case study — the field experiment on AAPA reranking and affective polarization published in *Science* (2025) — which is used to illustrate each methodological step from pilot design to final analysis.

### Schedule

| Part | Topic | Duration |
|---|---|---|
| 1 | History & foundations of feed ranking algorithms | 45 min |
| 2 | Feed experiments using middlewares | 45 min |
| 3 | Planning & analyzing experiments | 45 min |
| 4 | Hands-on exercise: Build your own Bluesky feed | 1 hour |

### What you will learn

**Part 1 — History & foundations** traces the evolution of feed ranking from Facebook's first algorithmic feed in 2011 to modern systems that score posts across hundreds of signals. It covers how engagement-maximizing algorithms work, what the research literature says about their societal effects (polarization, misinformation, mental health), and why the design space explored by platforms so far is far narrower than what is possible — and what researchers can contribute.

**Part 2 — Feed experiments using middlewares** explains the technical architecture for running feed reranking experiments on closed platforms (X, Facebook, Instagram) without platform cooperation. This includes network-level feed interception via browser extensions, real-time LLM-based scoring, feed reranking and filtering, and behavioral event logging. The FeedMonitor reference implementation is used as a worked example.

**Part 3 — Planning & analyzing experiments** covers the full experimental workflow: running pilots, conducting power analyses, writing a pre-analysis plan, checking covariate balance via permutation tests, and diagnosing attrition. All steps are illustrated using data from the AAPA reranking experiment.

**Part 4 — Hands-on exercise** walks participants through building a custom feed algorithm on Bluesky using its open Feed Generator API, from setting up a feed server to defining ranking logic and deploying the feed.

### Prerequisites

- Familiarity with Python and HTTP APIs
- A Bluesky account (recommended)

### Resources

| Resource | Link |
|---|---|
| Slides: History & foundations | [PDF](https://socialmediafeed25.github.io/slides_pdfs/L1.pdf) |
| Slides: Feed experiments using middlewares | [Google Slides](https://docs.google.com/presentation/d/1717jL_sNQ1NUb-sI40KfjPSAc5z8fzTSsADga3mgq_M/edit?usp=sharing) |
| Slides: Planning & analyzing experiments | [PDF](https://socialmediafeed25.github.io/slides_pdfs/L3.pdf) |
| Hands-on exercise (GitHub) | [socialmediafeed25/ICWSM25BlueskyTutorial](https://github.com/socialmediafeed25/ICWSM25BlueskyTutorial) |


