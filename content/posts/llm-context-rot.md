---
date: '2025-08-11'
title: 'LLM Context Rot'
draft: false
tags: []
comments: true
showToc: false
TocOpen: false
hidemeta: false
# description: "Desc Text."
# canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: true
disableHLJS: false
hideSummary: true
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: false
ShowPostNavLinks: false
ShowWordCount: false
ShowRssButtonInSectionTermList: true
UseHugoToc: false
---

LLMs are context-driven services. You feed them some context, they generate a response. The context represents the entire chat conversation (or as much of it as fits within the
<span class="fancy-tooltip" style="position:relative;cursor:pointer;border-bottom:1px dotted #888;">context window<span class="fancy-tooltip-text" style="
    display:inline-block;
    visibility:hidden;
    opacity:0;
    transition:opacity 0.5s;
    position:absolute;
    left:50%;
    transform:translateX(-50%);
    bottom:125%;
    background:#333;
    color:#fff;
    padding:6px 10px;
    border-radius:4px;
    white-space:nowrap;
    z-index:10;
    font-size:0.95em;
    pointer-events:none;
  "> The maximum number of tokens it can process at once</span></span>), and any additional files attached to that conversation.

  Context rot is when the LLM context has degraded so much that it becomes unusable.

<style>
.fancy-tooltip:hover > .fancy-tooltip-text {
  visibility: visible !important;
  opacity: 1 !important;
  pointer-events: auto !important;
  transition-delay: 0s, 0s;
}
.fancy-tooltip > .fancy-tooltip-text {
  transition: opacity 0.5s;
  transition-delay: 0s, 0.5s;
}
</style>

Excluding the response from the initial prompt, all other responses are a result of the current prompt plus the chat history. This means that the LLM is using its own previous responses as context for generating new responses.


```python
context = []
while user_prompt := get_user_prompt():
    context.append({"role": "user", "content": user_prompt})
    prompt_response = get_llm_response(context)
    context.append({"role": "llm_assistant", "content": prompt_response})
    print(prompt_response)
```

It's well known that LLMs sometimes hallucinate. I once read that LLMs, in fact, always hallucinate; it's just that sometimes they also get things right. I like that definition more. The accuracy of their responses is probabilistically determined. When you include incorrect data in the context, the probability of getting incorrect responses increases.

Additionally, I've seen LLMs reuse previous incorrect approaches, even after explicitly being told they were incorrect.

<div class="chat-list">
  <div class="chat-bubble user">Do this thing</div>
  <div class="chat-bubble assistant"><i>Incorrect approach</i></div>
  <div class="chat-bubble user">That's incorrect, do it this way</div>
  <div class="chat-bubble assistant"><i>Correct approach</i></div>
  <div class="chat-bubble user">Now do this other thing</div>
  <div class="chat-bubble assistant"><i>Reuses the same initial incorrect approach</i></div>
</div>

<style>
.chat-list {
  display: flex;
  flex-direction: column;
  gap: 0.5em;
  margin: 1em auto;
  max-width: 450px;
  align-items: stretch;
}
.chat-bubble {
  max-width: 90%;
  padding: 0.6em 1em;
  border-radius: 1.2em;
  font-size: 1em;
  line-height: 1.4;
  box-shadow: 0 1px 4px rgba(0,0,0,0.07);
  word-break: break-word;
}
.chat-bubble.user {
  align-self: flex-start;
  background: #e0e7ff;
  color: #222;
}
.chat-bubble.assistant {
  align-self: flex-end;
  background: #f1f5f9;
  color: #333;
  border: 1px solid #cbd5e1;
}
</style>

<style>
@media (prefers-color-scheme: dark) {
  .chat-bubble.user {
    background: rgb(65, 66, 72);
    color: #f4f4f5;
  }
  .chat-bubble.assistant {
    background: rgb(40, 41, 46);
    color: #e0e0e0;
    border: 1px solid rgb(55, 56, 62);
  }
}
</style>

In my opinion, all LLM chats inevitably converge toward context rot as their context becomes increasingly polluted. I've had to ditch chats countless times because the LLM started spewing nonsense and no corrective prompts could put it back on track. It's worth noting that I'm far from being a [good prompt engineer](https://addyo.substack.com/p/the-prompt-engineering-playbook-for) and I'm sure my noob level prompting skills didn't help.

The solution is simple, open a new chat and rework the context using the lessons learned from previous chats. Your prompts will likely be better, and you'll probably get better results this time around.
