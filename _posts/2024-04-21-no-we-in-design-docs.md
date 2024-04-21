---
title: "There is no 'we' in design docs"
date: 2024-04-21T14:05:00-04:00
classes:
categories:
  - communication
editors:
  - Bianca Homberg
tags:
  - lessons
  - writing
---

Once your software project reaches a certain size, you probably want to have a design document that describes how the system you're building should behave. Putting your design into natural language and diagrams before you put it into code helps you get to the correct design more quickly, since words and pictures are easier to read and change than code. A design doc also encourages everyone on the team to think in terms of the end goals and fundamental constraints of the system rather than focusing on details of the code. The downside is that natural language is less precise than code, which leads to some predictable failure modes.

# The worst-case scenario for design docs

The worst thing you can do when writing a design doc[^other-tech-writing] is to leave the reader confidently incorrect. This is substantially worse than leaving your reader confused: if they know they're confused, they know they need to seek out more information.

If your reader reads your doc, builds the wrong mental model of the system, and _believes_ they have the correct mental model, your design doc has failed to serve its purpose. You won't get good feedback about your proposed design, the reader will make assumptions about the system that aren't valid, and eventually things will break in surprising and frustrating ways.

# 'We'

There are lots of ways to let your reader build the wrong mental model, but for this post I'll focus on one that I've seen come up a lot in design docs I've reviewed: 'we'. As in "we will keep track of a mapping from users to documents so that we can enforce permissions" or "we will retry the query if it times out".

In both of those examples, 'we' hides the truth of how the system behaves. Assuming your design doc is discussing a typical webapp, is it the frontend that will keep track of the permissions mapping/retry the query? The backend? A new service you're going to build? These different answers have different design implications and would prompt different follow-up questions.

'We' is dangerous because different readers can read 'we' and fill it in with different parts of the system. They may not even be aware of the other possible interpretations that other readers will come away with (this is particularly likely if different people on your team tend to focus on different parts of the system or specialize in different disciplines).

# Be explicit

The solution is pretty straightforward: be explicit about which part of the system performs which action. The subject in each sentence of your design doc should be a specific, named part of your system: "_the webapp service_ will maintain a mapping from users to documents so that it can enforce permissions", "_the frontend code_ will retry the query if it times out".

Now everybody reading the doc has the same idea about how the system will behave and can ask meaningful follow-up questions ("if the frontend code retries the query, how does the backend handle the case where the frontend has asked to run the same query twice?").

# Exceptions

There are some cases where you actually want the ambiguity that 'we' provides. If you're communicating externally (e.g. public-facing documentation), you may not want to get into the details of how your application is implemented. If you're making a promise about the behavior of your system that you expect to be true over many changes to the particulars of your implementation (e.g. "for compliance reasons, we do not store user data"), you want your language to be general enough that it doesn't sound like you're trying to use technical precision to deceive your reader.

[^other-tech-writing]: Or blog, documentation, or other piece of technical writing
