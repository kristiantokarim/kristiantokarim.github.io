+++
date = '2026-03-17T09:00:00+07:00'
draft = false
tags = ["thinking", "engineering"]
title = 'Acceptable Impurity'
summary = "The FDA allows 60 insect fragments per 100 grams of chocolate. You wash your hands until acceptable, not until clean. This changed how I build software."
+++

Three years ago I saw an Instagram post about insect contamination in food and went looking for the actual numbers. The [FDA publishes a handbook](https://www.fda.gov/food/current-good-manufacturing-practices-cgmps-food-and-dietary-supplements/food-defect-levels-handbook) for this. Up to 60 insect fragments per 100 grams of chocolate. Ground cinnamon gets 400 insect fragments and 11 rodent hairs per 50 grams.

A [study on handwashing](https://pmc.ncbi.nlm.nih.gov/articles/PMC3037063/) found that even after scrubbing with soap, about 8% of samples still had fecal-origin bacteria. You wash your hands anyway. You just don't think about what's left.

You could scrub with surgical-grade antiseptic. Soak your hands in alcohol. They'd be cleaner. Your skin would crack and bleed within a week.

You don't wash your hands until they're clean. You wash until what's left won't make you sick. I'd always done this. I just never noticed.

#### The Deployment Log

We run a monorepo with multiple services. Every week, someone spends a few hours cooking a deployment log. Go through 10 to 20 commits, figure out which services each commit impacts, write it up. Manual and tedious.

I built a tool to automate it. Compare the current production release tag with the latest main branch, analyze the commits in between, output which services need deployment. Top-level folder changes are easy to map. For cross-service dependencies, `go list` handles it. If service A changes and service B depends on it, the tool flags both.

It worked. Until I hit the common package.

Changes to the shared package affected everything. So did `go.mod` updates. Any commit touching either of those would flag every service as impacted. I had two options. Build a deeper analysis layer using `go/ast` to trace actual usage, which would take weeks of work for diminishing returns. Or use an LLM as a filtering layer, which worked but cost more than the problem justified.

I stepped back from both. Then I checked the data.

In the last three months, the common package had been changed about 10 times total. `go.mod` changed more often but the changes were usually low impact and easy to eyeball. The edge case that was blocking me barely happened.

Before that Instagram post, I'd have kept grinding. Trying to get the tool to handle every path correctly, feeling like shipping it with known false positives meant shipping something broken. I wouldn't have checked the data. I would have kept solving.

So I shipped it as-is. `go list` handles the common case. Occasionally it produces a false positive when someone touches the shared package or `go.mod`. When that happens, we do the manual analysis on those specific commits. Takes a few minutes.

The release log went from a few hours to under an hour. The remaining impurity was small enough to live with.

#### The Line

The hard part was never accepting impurity. Every engineer already does that. You skip edge cases, leave TODOs, ship code that handles 99% of the paths. The hard part is being deliberate about where you draw the line.

It means knowing the number. Not assuming the edge case is rare. Checking the logs, counting the occurrences, deciding with data what you're willing to leave behind.

The difference between broken and acceptable isn't the number of edge cases you missed. It's whether you counted them.

