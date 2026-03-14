---
# Page settings
layout: default
keywords:
comments: false

# Hero section
title: Web Extensions for feed reranking
description: Run feed reranking experiments on closed platforms without platform collaboration.

# Author box
author:
    title: About Author
    title_url: '#'
    external_url: false
    description: Tiziano Piccardi

# Micro navigation
micro_nav: true
---

Web extensions are the most practical approach for running feed experiments on major closed platforms such as Facebook, X (Twitter), Instagram, and Threads, where researchers have no access to backend systems or APIs. Rather than waiting for platform collaboration, a web extension allows researchers to intercept and modify what participants see in real time while they browse their real feeds using their own accounts.

## Why This Approach

The dominant platforms have progressively restricted research access. APIs have been shut down or paywalled, and the era of large-scale academic collaborations exemplified by the 2020 Facebook and Instagram Election Study has largely ended. This leaves researchers with a practical problem: how do you run a controlled experiment on a feed algorithm you cannot access?

Web extensions solve this by operating entirely on the client side. Instead of modifying the platform, the extension modifies what the participant's browser renders. The platform never needs to cooperate with the experiment. This enables naturalistic experiments in which participants use their real accounts, interact with their real social graph, and engage with authentic content while researchers retain fine-grained control over what appears in the feed and in what order.

## How It Works

A web extension can intervene in a participant's feed in two complementary ways, which are typically combined in practice.

### Intercepting the Network Response

The most powerful approach is to override the browser's network request layer (for example `XMLHttpRequest` or `fetch`) before the page loads. When the platform requests a batch of posts from its servers, the extension intercepts the raw JSON response, pauses rendering, applies the experimental transformation (reranking, filtering, content editing), and then releases the modified payload back to the page. Because the intervention happens at the data layer before the frontend framework processes the response, the rendered feed reflects the modified ranking without visual glitches or consistency issues.

The interception logic must be injected into the main page scope rather than the content script scope by loading it as a web-accessible resource in the extension manifest. The content script and injected script communicate via the browser's message-passing API. If the reranking logic is simple enough, it can run entirely in the browser using TensorFlow.js, WebAssembly, or ONNX, preserving participant privacy. For more complex models such as LLM-based classifiers, the extension forwards the post batch to an external backend server, waits for the scored and reranked response, and substitutes it before rendering resumes.

This architecture concentrates most of the latency on the initial feed load. Subsequent scroll batches are preloaded in the background by the platform's infinite-scroll pipeline, so the intervention delay is absorbed before the user reaches them. In a 1,256-participant field study on X, the extension introduced an average delay of approximately three seconds on the initial feed load, which went unnoticed by 98% of participants.

### Manipulating the DOM

For interface-level changes such as injecting in-feed surveys, adding warning labels, or inserting custom widgets, DOM manipulation is appropriate. After the feed renders, a content script observes DOM mutations via `MutationObserver` and modifies or augments post elements directly.

```js
const articles = document.querySelectorAll('article');
articles.forEach(article => {
  if (matchesCriteria(article.innerText)) {
    article.style.display = 'none';
  }
});
```

DOM manipulation is simple for filtering and annotation but unsuitable for reranking because frameworks such as React dynamically recycle DOM nodes as the user scrolls. The DOM tree is therefore incomplete and unstable at any given moment. For reranking interventions, the network interception approach described above should be used instead.

### The Mixed Approach

Most serious experiments combine both layers: **network interception for data manipulation** (reranking, filtering, content editing) and **DOM manipulation for interface changes** (surveys, labels, visual indicators). This architecture is used in the FeedMonitor reference implementation.

```
Original feed (JSON)
    → Intercepted by extension
    → Sent to backend / processed locally
    → Reranked payload returned
    → Rendered by platform frontend
    → DOM layer adds in-feed widgets
```

## What You Can Change

Working at the network layer provides access to the full post payload before rendering and enables four classes of intervention:

- **Rerank** — rescore posts according to any objective (toxicity, sentiment, topic, engagement) and reorder the batch  
- **Remove** — exclude specific posts from the rendered feed  
- **Edit** — modify post text, social metrics (likes, shares), or attachments before display  
- **Add** — inject new posts into the feed from an external inventory and blend them with platform-ranked content  

Up-ranking is harder than down-ranking because the extension only sees the posts the platform has already selected. Expanding the working set requires pre-fetching additional batches by simulating scroll requests in the background. In the FeedMonitor experiment this approach expanded the candidate set from 30 to 90 posts per session.

## Reference Implementation

The **FeedMonitor** repository provides a production-tested blueprint for the full stack: network interception, feed reranking, behavioral event logging, and a Python backend. It was used in a 10-day preregistered field experiment on X involving 1,256 participants.

→ https://github.com/StanfordHCI/FeedMonitor

The repository includes:

- `injected.js` — network interception logic injected into the page scope  
- `logic.js` — core reranking logic; calls the backend for scoring  
- `events.js` — behavioral event logging (visibility, clicks, likes, dwell time, tab focus)  
- `launcher.js` — extension initialization and configuration  
- Python backend — scoring and reranking server (Flask; Gunicorn recommended for production)

The experimental deployment is described in:

Piccardi, T., Saveski, M., Jia, C., Hancock, J. T., Tsai, J., & Bernstein, M. S. (2025).  
**Reducing partisan animosity through algorithmic feed reranking.** *Science*.  
https://www.science.org/doi/10.1126/science.adu5584

## Browser Compatibility

Chrome and Edge are the primary targets and fully compatible with the Manifest V3 extension standard. Firefox is compatible via its Add-Ons store. Safari requires submission to the App Store and involves a stricter review process.

Chrome dominates desktop browser usage, accounting for roughly 70% of the market, making it the practical default for participant recruitment.

## Limitations

### Desktop Only

Web extensions run in desktop browsers and do not run inside native mobile apps, which is where most social media consumption occurs. This is the primary practical limitation of the approach and a threat to the external validity of studies that do not account for it.

On mobile devices, certificate pinning prevents network interception, official app stores do not distribute unauthorized clients, and there is no extension runtime environment. Partial workarounds exist. For example, a React Native WebView wrapper can inject similar JavaScript logic, and Edge Canary for Android has experimental extension support. However, none of these solutions are currently suitable for large-scale production studies. On iOS, TestFlight limits distribution to 100 participants before Apple review is required.

In practice, the desktop constraint should be addressed in the study design. Recommended mitigations include instructing participants to access the platform through their desktop browser during the study period, logging behavioral traces to detect mobile usage leakage, and comparing logged activity against participants' public activity to estimate treatment dilution.

### Platform API Instability

Extensions that intercept network responses depend on the stability of the platform's frontend API endpoints. These endpoints can change without notice. Long-running deployments therefore require automated monitoring to detect when interception logic stops functioning.

### Feedback Loops with Platform Algorithms

Down-ranking content reduces engagement signals on that content. This may cause the platform's recommendation algorithm to further suppress it in subsequent sessions. Depending on the research question, this may reinforce the intervention or complicate causal interpretation. Researchers should account for this possibility in their analysis.

### Inventory Constraint

The extension can only operate on posts that the platform has already selected to show the user. True up-ranking, meaning surfacing content that the platform would not have displayed, requires either pre-fetching additional batches or maintaining a separate content inventory. Both approaches introduce additional infrastructure complexity.