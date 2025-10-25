---
title: "Building a Git-Backed Blog with Rust and Axum"
date: "2025-10-25T21:17:46Z"
description: "How I built a personal website with automatic content updates by treating Git as a content management system, eliminating the need for databases or manual deployments."
tags: ["rust", "web", "axum", "git", "systems"]
---

## Introduction

Most static site generators follow a familiar pattern: write content in Markdown, run a build command, and deploy the generated HTML. This works well, but it creates friction. Every content change requires rebuilding and redeploying the entire site. For a personal blog, this overhead feels unnecessary.

I wanted something different. I wanted to push a new blog post to a Git repository and have it appear on my website within minutes, without touching the server or triggering any build pipelines. This post describes how I built that system using Rust, Axum, and Git as a content management system.

## The Core Concept

The architecture rests on a simple idea: treat a Git repository as the source of truth for content, and have the web server periodically pull updates. When new content appears, parse it and serve it immediately. No database, no build step, no deployment ceremony.

This approach works because blog content has specific characteristics. Posts are immutable once published. Updates are infrequent. Read operations vastly outnumber writes. These properties make caching straightforward and eliminate most complexity around consistency.

## Technology Choices

I chose Rust and Axum for the backend. Rust provides memory safety without garbage collection overhead, making it well-suited for long-running servers. Axum builds on top of Tokio's async runtime and provides excellent ergonomics for web applications.

The git2 library handles repository operations. It provides Rust bindings to libgit2, enabling clone, fetch, and reset operations without shelling out to the git command. pulldown-cmark handles Markdown parsing, supporting GitHub-flavored extensions like tables and strikethrough. syntect provides syntax highlighting for code blocks, using Sublime Text's language definitions to support over 200 programming languages.

For the frontend, I kept things minimal. Custom CSS provides a light and dark theme using CSS custom properties. A small JavaScript file handles theme persistence. No framework, no build step, just plain HTML and CSS served directly.

## Server Architecture

The application follows a straightforward structure. At startup, the server loads configuration from environment variables, initializes an in-memory content cache, and spawns a background task for Git synchronization. The main thread runs the Axum HTTP server.

Application state consists of two components wrapped in an Arc for shared ownership. The configuration holds server settings and repository URLs. The content cache stores parsed blog posts in a HashMap protected by an RwLock. This allows concurrent reads while ensuring exclusive access for updates.

The router defines endpoints for the home page, blog listing, individual posts, and static assets. Each handler receives the application state and can read from the content cache without blocking other requests. This design keeps request handling fast and predictable.

## Git Synchronization System

The synchronization system runs in a background Tokio task, separate from request handling. Every five minutes, it performs a fetch from the remote repository followed by a hard reset to the remote's main branch. If the remote commit differs from the local commit, the task triggers a content reload.

The hard reset strategy treats the remote repository as the single source of truth. Any local modifications are discarded. This simplifies the synchronization logic considerably, eliminating concerns about merge conflicts or divergent histories. For a single-author blog where content only flows in one direction, this approach works perfectly.

The synchronization code runs in a blocking context using `tokio::task::spawn_blocking` because libgit2 performs blocking I/O operations. This prevents Git operations from blocking the async runtime's thread pool. The blocking task clones the repository path and operates independently.

## Content Parsing and Caching

Blog posts are Markdown files with YAML frontmatter. The frontmatter includes the title, publication date, description, and tags. The gray_matter crate parses this format reliably, separating metadata from content.

When the synchronization system detects changes, it scans the posts directory and parses each Markdown file. For each file, it extracts the slug from the filename, parses the frontmatter, converts the RFC3339 date string to a DateTime, and renders the Markdown to HTML using pulldown-cmark.

The parser also calculates an estimated reading time by counting words and dividing by 200 words per minute. This provides readers with an expectation of time commitment before clicking through.

All parsed posts get stored in a HashMap keyed by slug. The reload operation constructs a new HashMap and performs an atomic swap under a write lock. This ensures readers never see a partially updated cache. Either they see the old content or the new content, never a mix.

## Syntax Highlighting

Code blocks receive syntax highlighting during the Markdown rendering phase using the syntect library. Syntect provides access to over 200 language definitions from Sublime Text's syntax files, covering everything from common languages like Rust and Python to specialized formats like configuration files and assembly.

The implementation takes an unusual but effective approach to theme compatibility. Rather than highlighting code once and trying to make it work with both light and dark themes, the system renders each code block twice with different color schemes. Light mode uses the InspiredGitHub theme, which provides clean, readable highlighting similar to GitHub's interface. Dark mode uses the base16-ocean.dark theme, featuring muted oceanic colors that remain comfortable during extended reading.

During content parsing, the Markdown renderer processes each fenced code block through syntect twice, once for each theme. The resulting HTML includes both versions wrapped in containers marked with content-light and content-dark classes. CSS controls visibility based on the current theme attribute, showing only the appropriate version.

This approach trades disk space and memory for rendering performance. Each code block consumes roughly twice the storage it would with a single color scheme. However, theme switching becomes instantaneous with no client-side processing required. Users experience no delay or flash of incorrectly styled code when toggling themes.

The pre-rendering strategy also simplifies the frontend. No JavaScript library needs to load. No syntax definitions ship to the browser. No client-side parsing or highlighting occurs. The browser simply displays pre-formatted HTML, which works reliably across all devices and connections.

## Request Handling

The blog listing handler retrieves all posts from the cache, sorts them by date in descending order, and generates HTML inline. Each post card displays the title, date, description, and tags. The HTML includes data attributes for client-side filtering.

Individual post handlers look up posts by slug in the cache. If found, the handler generates HTML with the post's title, date, reading time, rendered content, and tags. If not found, it returns a 404 status code.

The HTML generation currently uses string formatting rather than a template engine. For this scale of application, the simplicity outweighs the benefits of templates. The code remains readable and maintainable without introducing additional dependencies or compilation steps.

## Theme System

The website supports both light and dark themes with instant switching and no flash of unstyled content. The implementation uses CSS custom properties to define colors, and the `data-theme` attribute on the document element to toggle between theme sets.

On page load, JavaScript checks localStorage for a saved theme preference, falling back to the system preference indicated by the `prefers-color-scheme` media query. It applies the theme by setting the data attribute before the page renders, preventing any visual flash.

The theme toggle button updates both the DOM attribute and localStorage. CSS transitions provide smooth color changes. The icon changes between a moon and sun to indicate the current theme and the available toggle action.

## Search and Filtering

The blog listing page includes client-side search and tag filtering. JavaScript reads data attributes from post cards to perform filtering without server round trips. The search input uses debouncing to avoid excessive filtering operations while typing.

Tag filters support multi-selection with delayed activation. Clicking a tag enters a pending state, and after 600 milliseconds the filter activates. This prevents accidental filter activation from quick clicks while providing visual feedback.

All filtering happens in memory. The implementation uses Set data structures for efficient membership testing. Posts smoothly fade in and out as filters change, providing visual continuity.

## Deployment Strategy

The application deploys as a Docker container. The Dockerfile uses a multi-stage build, compiling dependencies in one stage and the application in another, then copying only the binary and static assets to a minimal Alpine Linux runtime image.

The container runs as a non-root user for security. It includes a health check endpoint that Kubernetes uses to verify the application is responding. The content directory gets mounted as a volume where the container clones the blog repository.

Environment variables configure the repository URL, sync interval, and server binding address. This keeps the container image generic and deployable to different environments with different configuration.

## Performance Characteristics

The server handles requests efficiently. Reading from the cache requires only acquiring a read lock, which multiple requests can hold simultaneously. Parse and rendering happen once during content reload, not on every request.

Memory usage remains low. The cache stores pre-rendered HTML for each post in both light and dark theme versions, trading memory for response time. The dual rendering approximately doubles the HTML storage per post, but with a typical blog of 50-100 posts, this still amounts to only a few megabytes. The Rust binary itself is small, and the Alpine runtime adds minimal overhead.

Startup time is fast. The application clones the repository and parses all posts in a few seconds. The server begins accepting requests as soon as initialization completes. There is no warm-up period or lazy loading.

## Tradeoffs and Limitations

This architecture makes specific tradeoffs. The five-minute sync interval means new content is not instantly visible. For a personal blog, this delay is acceptable. If instant updates were required, webhooks could trigger immediate synchronization.

The hard reset strategy assumes single-author, unidirectional content flow. If multiple authors were pushing content simultaneously, this approach would discard commits. A merge-based strategy would handle that case better, at the cost of additional complexity.

Content lives in memory rather than a database. This works well for blogs with hundreds or even thousands of posts, but would eventually hit memory limits. The tradeoff favors simplicity and performance for the typical case.

The inline HTML generation becomes cumbersome as pages grow more complex. Future work might migrate to a template engine. For now, the simplicity of string manipulation keeps the code straightforward and easy to modify.

## Conclusion

Building a blog with Git as a content management system provides a unique combination of benefits. Content updates require only a git push. The server requires no database or external dependencies. The application remains simple and maintainable.

The architecture demonstrates that not every web application needs complex infrastructure. Sometimes the right solution is a single binary, a Git repository, and a few hundred lines of Rust. The result is fast, reliable, and easy to reason about.

The complete source code is available on GitHub. The patterns described here extend beyond blogs to any content-driven website where updates are infrequent and content flows in one direction.
