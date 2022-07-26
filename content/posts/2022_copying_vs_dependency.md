---
title: "A little copying is better than a little dependency"
date: 2022-07-24T08:00:08+08:00
---

quote from https://go.dev/blog/supply-chain:

> The final and maybe most important software supply chain risk mitigation in the Go ecosystem is the least technical one: Go has a culture of rejecting large dependency trees, and of preferring a bit of copying to adding a new dependency. It goes all the way back to one of the Go proverbs: “a little copying is better than a little dependency”. The label “zero dependencies” is proudly worn by high-quality reusable Go modules. If you find yourself in need of a library, you’re likely to find it will not cause you to take on a dependency on dozens of other modules by other authors and owners.

That’s enabled also by the rich standard library and additional modules (the golang.org/x/... ones), which provide commonly used high-level building blocks such as an HTTP stack, a TLS library, JSON encoding, etc.

All together this means it’s possible to build rich, complex applications with just a handful of dependencies. No matter how good the tooling is, it can’t eliminate the risk involved in reusing code, so the strongest mitigation will always be a small dependency tree.


must watch this talk by rob pike: https://www.youtube.com/clip/UgkxWCEmMJFW0-TvSMzcMEAHZcpt2FsVXP65