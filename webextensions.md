---
# Page settings
layout: default
keywords:
comments: false

# Hero section
title: Web Extensions for feed reranking
description: Run feed reranking experiments on closed platforms without platform collaboration.

# # Author box
author:
    title: About Author
    title_url: '#'
    external_url: false
    description: Tiziano Piccardi

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


Web extensions are the most practical approach for running feed experiments on major closed platforms—Facebook, X (Twitter), Instagram, Threads—where researchers have no access to backend systems or APIs. Rather than waiting for platform collaboration (which rarely happens and may never happen again at scale), a web extension lets you intercept and modify what participants see in real time, on their actual feeds, with their actual social context.

## Why This Approach

The dominant platforms have progressively restricted research access. APIs have been shut down or paywalled, and the era of large-scale academic collaborations—exemplified by the 2020 Facebook and Instagram Election Study—appears to be drawing to a close. This leaves researchers with a fundamental problem: how do you run a controlled experiment on a feed algorithm you cannot touch?

Web extensions solve this by operating entirely on the client side. Instead of modifying the platform, you modify what the participant's browser renders. The platform never knows the experiment is happening. This enables naturalistic, ecologically valid experiments—participants use their real accounts, see their real social graph, and interact with real content—while giving researchers fine-grained control over what appears in the feed and in what order.

## How It Works

A web extension can intervene in a participant's feed in two complementary ways, which are typically combined in practice.

### Intercepting the Network Response

The most powerful approach is to override the browser's `XMLHttpRequest` object before the page loads. When the platform requests a batch of posts from its servers, the extension intercepts the raw JSON response, pauses rendering, applies the experimental transformation (reranking, filtering, content editing), and then releases the modified payload back to the page. Because the intervention happens at the data layer—before the frontend framework has processed anything—the rendered feed reflects the modified ranking with no visual glitches or consistency issues.

The interception logic must be injected into the main page's scope (not the content script scope) by loading it as a web-accessible resource in the extension manifest. The content script and injected script then communicate via the browser's message-passing API. If the reranking logic is simple enough, it can run entirely in the browser using TensorFlow.js, WebAssembly, or ONNX, preserving participant privacy. For more complex models—such as LLM-based classifiers—the extension forwards the post batch to an external backend server, waits for the scored and reranked response, and substitutes it before rendering resumes.

This architecture means that the latency cost is paid only once per session: on the initial feed load. Subsequent scroll batches are preloaded in the background by the platform's own infinite-scroll pipeline, so the intervention delay is absorbed invisibly. In a 1,256-participant field study on X, an average added delay of approximately 3 seconds on initial load went unnoticed by 98% of participants.

### Manipulating the DOM

For interface-level changes—injecting in-feed surveys, adding warning labels, inserting custom widgets—DOM manipulation is the right tool. After the feed renders, a content script observes DOM mutations (via `MutationObserver`) and modifies or augments post elements directly.

```js
const articles = document.querySelectorAll('article');
articles.forEach(article => {
  if (matchesCriteria(article.innerText)) {
    // hide, reorder, or annotate
    article.style.display = 'none';
  }
});
```

DOM manipulation is simple for filtering and annotation but unsuitable for reranking: frameworks like React recycle DOM nodes as the user scrolls, so the tree is incomplete and unstable at any given moment. For reranking, use the network interception approach above.

### The Mixed Approach

Most serious experiments combine both layers: **network interception for data manipulation** (reranking, filtering, content editing) and **DOM manipulation for interface changes** (surveys, labels, visual indicators). This is the architecture used in the FeedMonitor reference implementation.

```
Original feed (JSON)
    → Intercepted by extension
    → Sent to backend / processed locally
    → Reranked payload returned
    → Rendered by platform frontend
    → DOM layer adds in-feed widgets
```

## What You Can Change

Working at the network layer gives you access to the full post payload before rendering, enabling four classes of intervention:

- **Rerank** — rescore posts according to any objective (toxicity, sentiment, topic, engagement) and reorder the batch
- **Remove** — exclude specific posts entirely; the user scrolls past them without knowing
- **Edit** — modify post text, social metrics (likes, shares), or attachments before display
- **Add** — inject new posts into the feed from an external inventory, blending them with platform-ranked content

Up-ranking is harder than down-ranking because the extension only sees the posts the platform has already selected. Expanding the working set requires pre-fetching additional batches by simulating scroll requests in the background—the FeedMonitor experiment expanded from 30 to 90 posts per session this way.

## Browser Compatibility

Chrome and Edge are the primary targets and fully compatible with the Manifest V3 extension standard. Firefox is compatible via its Add-Ons store. Safari requires submission to the App Store and involves a stricter review process.

Chrome dominates desktop browser share (65–79% across measurement sources as of late 2024), making it the practical default for participant recruitment.

## Reference Implementation

The **FeedMonitor** repository provides a production-tested blueprint for the full stack: XHR interception, feed reranking, behavioral event logging, and a Python backend. It was used in a 10-day preregistered field experiment on X involving 1,256 participants and is the basis for the methods paper described in this site.

→ [github.com/StanfordHCI/FeedMonitor](https://github.com/StanfordHCI/FeedMonitor)

The repository includes:
- `injected.js` — XHR override injected into the page scope
- `logic.js` — core reranking logic; calls backend for scoring
- `events.js` — behavioral event logging (visibility, clicks, likes, dwell time, tab focus)
- `launcher.js` — extension initialization and configuration
- Python backend — scoring and reranking server (Flask; Gunicorn recommended for production)

_Tiziano Piccardi, Martin Saveski, Chenyan Jia, Jeffrey T. Hancock, Jeanne Tsai, and Michael Bernstein. 2026. Reranking Social Media Feeds: A Practical Guide for Field Experiments. Trans. Soc. Comput. 2026._ [https://doi.org/10.1145/3800557](https://doi.org/10.1145/3800557)


## Limitations

### Desktop Only

Web extensions run in desktop browsers. They do not run in native mobile apps, which is where most social media consumption happens. This is the primary practical limitation of the approach and a threat to the external validity of any study that does not account for it.

On mobile, certificate pinning prevents network interception, official app stores do not distribute unauthorized clients, and there is no extension runtime. Partial workarounds exist—a React Native WebView wrapper can inject the same JavaScript logic, and Edge Canary for Android has experimental extension support—but none of these are production-ready for large-scale studies. iOS via TestFlight is capped at 100 participants before Apple review is required, and Apple may reject the app.

In practice, the desktop-only constraint should be addressed in the study design. The recommended mitigations are: explicitly instructing participants to access the platform through their desktop browser for the duration of the study, logging behavioral traces to detect mobile usage leakage, and comparing logged activity against participants' public activity to estimate treatment dilution.

### Platform API Instability

Extensions that intercept network responses depend on the platform's frontend API endpoints remaining stable. These endpoints can and do change without notice. Any long-running deployment needs automated monitoring to detect when the interception logic breaks.

### Feedback Loops with Platform Algorithms

Down-ranking content reduces engagement signals on that content. This may cause the platform's own recommendation algorithm to further suppress it in subsequent sessions—which could either reinforce the intervention's goals or complicate causal interpretation, depending on the research question. This should be accounted for in the analysis.

### Inventory Constraint

The extension can only work with posts the platform has already selected to show the user. True up-ranking—surfacing content the platform would not have shown—requires either pre-fetching additional batches or maintaining a separate content inventory, both of which add significant infrastructure complexity.

