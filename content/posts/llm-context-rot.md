---
date: '2025-08-10T19:50:09-04:00'
title: 'LLM Context Rot'
draft: true
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

LLMs are context driven services. You give them some context, they provide a response. The context represents the entire chat conversation, or as much of it as it fits within the
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
  "> The maximum number of tokens it can process at once</span></span>, and any additional files attached to that conversation.

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

Excluding the response from the initial prompt, all other responses are a result of the current prompt plus the chat history. This means that the LLM is using it's own previous responses as context for generating new responses.


```python
context = []
while user_prompt := get_user_prompt():
    context.append({"role": "user", "content": user_prompt})
    prompt_response = get_llm_response(context)
    context.append({"role": "llm_assistant", "content": prompt_response})
    print(prompt_response)
```

It's well known that LLMs sometimes hallucinate.

<img src="/img/hallucination_doors_of_stone.jpg" alt="Doors of Stone Hallucination" width="400" height="400">

A definition that I like better is that LLMs, in fact, always hallucinate, it's just that sometimes they also get things right. The accuracy of their responses is probabilistically determined. When you include incorrect data in the context, the probability of getting incorrect responses increases.

Additionally, I've seen LLMs reuse old, incorrect answers, even after explicitly being told they were incorrect.

> 1. Do this thing
> 2. "Incorrect approach"
> 3. No, do it this way
> 4. "Correct approach"
> 5. Now do this other thing
> 6. "Uses a similar incorrect approach as 2"

At some point the context gets polluted to the point of becoming unusable. It has been my experience that all LLM chats eventually converge towards context rot. I've had to ditch chats countless times because the LLM started spewing nonsense and no corrective prompts could put it back on track. Worth noting that I'm far from being a [good prompt engineer](https://addyo.substack.com/p/the-prompt-engineering-playbook-for) and I'm sure my prompting skills, or lack thereoff, didn't help.

The solution is simple. Open a new chat and rework the context using the lessons learned from previous chats. It works great.
