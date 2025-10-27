---
title: "I Built This Website by Talking to an AI"
date: "2025-10-26T00:00:00Z"
description: "Some observations about building a personal website using Claude Code, including what worked and what was annoying."
tags: ["ai", "rust", "web"]
---

## The Setup

I built this website by having conversations with [Claude Code](https://claude.ai/). Not because I wanted to make a point about AI, but because I was curious if it would actually work for a real project. The result is a Rust web server that pulls [blog posts](https://github.com/willfindlay/blog) from git, renders markdown in both light and dark themes, and runs in Kubernetes. It's just a personal website, but it has enough moving parts to be interesting.

## What Worked

The best part was not having to look things up. When I needed syntax highlighting, Claude knew about [syntect](https://crates.io/crates/syntect) and how to integrate it with the markdown parser. When I wanted a file watcher for hot-reload, it knew about the [notify](https://crates.io/crates/notify) crate and correctly handled the async/sync boundary. This breadth of knowledge meant I could focus on what I wanted rather than how to implement it.

Code generation was solid. The Rust code follows proper conventions: error handling with [anyhow](https://crates.io/crates/anyhow), async patterns with [tokio](https://tokio.rs/), appropriate use of [Arc](https://doc.rust-lang.org/std/sync/struct.Arc.html) and [RwLock](https://doc.rust-lang.org/std/sync/struct.RwLock.html) for shared state. There were no obvious quality issues. It just worked.

Iteration was fast. I could describe a problem, get a solution, test it, and provide feedback all within a few minutes. When something didn't work, Claude usually diagnosed it correctly. The file watcher not triggering reloads? "The notify crate requires a synchronous thread, but your reload is async. Capture the tokio runtime handle before spawning." That's the right answer, delivered immediately.

## What Was Annoying

Context retention is a problem. After dozens of messages, Claude would sometimes suggest things that contradicted earlier decisions. "We already decided pages use a file watcher, not git sync" became a recurring correction. I worked around this by maintaining a comprehensive [CLAUDE.md](https://github.com/willfindlay/website/blob/main/CLAUDE.md) file with architectural decisions, but that added overhead.

Claude doesn't ask clarifying questions when it should. When faced with ambiguity, it picks a reasonable default and runs with it. This led to some wasted work. The checkbox persistence debate is a good example: Claude implemented localStorage persistence, I realized I didn't want that, we removed it. If Claude had asked "should checkbox state persist?" upfront, we could have skipped that iteration.

I also felt a low-level anxiety about understanding the codebase. When you write code yourself, you develop an intimate familiarity with every detail. With AI-generated code, that familiarity is shallower. I understand the architecture, but I couldn't tell you every edge case without looking. This might be fine—we work with code we didn't write all the time—but it feels different when the code was generated on-demand.

### The CLAUDE.md Problem

Maintaining documentation became essential. [CLAUDE.md](https://github.com/willfindlay/website/blob/main/CLAUDE.md) grew to over 800 lines covering everything from startup sequence to CSS color palettes. This served as persistent context across conversations, but it also became a maintenance burden. Every feature required documentation updates.

On the other hand, the documentation is valuable beyond AI assistance. It's a specification of the system as built. When I come back to this project in six months, CLAUDE.md will remind me how everything works. So maybe it's not wasted effort.

## Observations

Building with AI feels like working with a junior developer who has encyclopedic knowledge but needs clear direction. Claude can generate correct implementations quickly, but I made every architectural decision and reviewed every change. The division of labor was clear: I own the design, Claude owns the typing.

Is the code as good as what an expert could write? Probably not. Is it good enough? Yes. Does it work? Yes. Would it have taken longer to build without AI? Definitely.

The interesting question isn't whether AI can write code—it can. The question is what working this way feels like, and whether that feeling is sustainable. For a personal website, it was fine. For a larger project with multiple people, I'm not sure. Context retention issues would compound, and the documentation overhead would grow.

But for solo projects where you want to move fast and don't mind reviewing generated code, AI assistance is genuinely useful. Not revolutionary, not magical, just useful.

The code is [on GitHub](https://github.com/willfindlay/website) if you want to see what AI-assisted Rust looks like in practice. The [blog content](https://github.com/willfindlay/blog) and [resume](https://github.com/willfindlay/resume) live in separate repositories.
