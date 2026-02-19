+++
date = '2025-09-22T09:00:00+08:00'
draft = false
tags = ["thinking", "ai"]
title = 'The Hard Part Was Never Writing Code'
summary = "The bottleneck in software engineering was never writing code. It was always knowing what to build. AI just makes that harder to ignore."
+++

Over the past year of working with AI as a daily tool, I've noticed a pattern: the engineers who get the most out of it aren't the ones with the best prompt tricks. They're the ones who can articulate exactly what they want.

#### What Was Always True

The bottleneck in software engineering was never execution. Writing code was never the hard part. The hard part was always figuring out what to build.

AI just makes this more obvious. The gap between "I know what I want" and "I have working code" keeps shrinking. You still need someone who knows what correct looks like, but the cost of turning a clear idea into working code is dropping fast.

So the engineer who can think clearly about what to build before building it has always had an edge. Now that edge is massive.

#### What "Clarity" Actually Means

When I say clarity, I don't mean writing detailed specs or Jira tickets. I mean the ability to:

- **Define the actual problem**, not the symptom. "The page is slow" is a symptom. "We're loading 100k rows into memory to paginate 20 results" is a problem.
- **State your constraints explicitly.** What can't change? What's negotiable? What does "good enough" look like?
- **Identify the tradeoffs before you start.** The engineer who can articulate tradeoffs upfront makes better decisions, and gives better instructions to both humans and AI.

Give a clear, constrained problem statement to an AI and you get back something useful. Give it something vague and you get slop.

#### What This Looks Like in Practice

When I review code and something doesn't sit right, I don't just flag it. I stop and figure out what specifically is wrong. A function name feels off. Is it misleading, or just abbreviated to the point where you need context to decode it? A piece of logic feels overcomplicated. Is it solving a problem that doesn't exist, or is the complexity genuinely needed and I'm just not seeing why?

That process of going from "this feels wrong" to "this is wrong because X" is the actual work. Do it enough times and you start to see patterns. The things that bother you in code reviews aren't random. They cluster: naming, race conditions, failure handling, unnecessary abstraction. Once you notice the clusters, you can articulate them. Once you can articulate them, you can delegate the work to a colleague or an AI, and they'll know exactly what you expect.

Same thing with writing. I read a doc and a sentence sounds off. I stop and ask: is it incoherent with what came before? Is the wording imprecise, "recently" instead of a date, "significant" without a number? Is it just too many words for what it's saying? I pin it down. Over time, those specific answers turned into a set of things I consistently care about: top-down flow, precise language, conciseness, coherence.

I didn't sit down one day and write a checklist. The checklist came from repeatedly doing the hard work of turning a vague instinct into a specific objection. That's the part most people skip. And it's the same part that matters when you're working with AI. When I give an AI a task, I treat it the way I'd treat a colleague: I say exactly what I mean, because I know interpretation gaps lead to wrong output. The clarity comes from having already done the thinking.

#### Why This Is What Seniority Actually Looks Like

This isn't just about reviewing code or writing docs. It's the same skill behind every part of the job that gets harder as you grow.

Scoping a project. Aligning a team on what to build. Deciding what not to build. Breaking a vague business ask into concrete technical work. These all require the same thing: the ability to take something fuzzy and make it specific.

People sometimes look at senior engineers and think the job got less technical. Fewer pull requests, more docs, more meetings. But the work didn't get less technical. It got less visible. The hard part moved from "how do I implement this" to "what exactly should we implement and why." The thinking is the work. It always was. It's just that earlier in your career, implementation is hard enough to absorb all your attention, so you mistake it for the real challenge.

#### The Practical Takeaway

Before jumping to a solution, articulate the problem. Before delegating to a person or an AI, define what good looks like. Before building, make sure you can explain what you're building and why in plain language.

And write clearly. I started picking this up at Amazon Japan, where a senior engineer drilled it into me, and I've been working on it since. Writing forces you to actually think things through. If you can't write it down clearly, you haven't thought it through.

These skills have always separated strong engineers from the rest. AI just makes that harder to ignore.

