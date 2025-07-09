---
title: "Custom Semantic Code Checks for PRs"
date: 2025-07-09T13:16:50+01:00
draft: false
toc: false
tags: 
  - untagged
---

_[Find the full docs for Rules here](https://docs.recurse.ml/gh/configs/rules)_

![Recurse ML PR Comment on a redundant comment with proposed suggestion (removal of comment)](header.png)

Maintaining consistent code quality while scaling the team is difficult.
Well-intentioned Notion style guides are time-consuming to write and are often disregarded as soon as they're written.
In this post I'll describe how you can use Recurse ML's new Rules Feature to put the code style guide inside of your PR process.

Recurse ML is a GitHub App that identifies bugs introduced in PRs.
It focuses on semantic issues that slip past static analysers.

We realised that every team's "source of paranoia" is different.
While some universal bugs exist, the most valuable catches are often team-specific.
That's why we built rules - a way to teach Recurse ML about your codebase's unique requirements.

## Quickstart

First off, install Recurse ML [GitHub app](https://github.com/apps/recurseml/) if you haven't already.

Then, create a `.recurseml.yaml` file (you can place it anywhere in your project):

```yaml
# put this in .recurseml.yaml anywhere in your repo
rules: .cursor/rules
```

Recurse ML rules are compatible with [Continue's](https://docs.continue.dev/customize/deep-dives/rules) [Cursor's](https://docs.cursor.com/context/rules) rule formats.

Next, create your rules directory and add your first rule. Create a file called `effective-comments.md` in your `.cursor/rules` directory:

```md
---
Name: effective-comments
Description: Explain WHY not WHAT the code does
Globs: 
  - "**/*.js"
  - "**/*.ts"
  - "**/*.py"
  - "**/*.go"
  - "**/*.java"
  - "**/*.rb"
  - "**/*.cs"
---

# Effective Code Comments

Explain WHY not WHAT the code does. Document complex business logic, clarify non-obvious implementations, warn about gotchas, and provide context for maintainers. Use proper documentation comment format for functions/methods. Keep TODO comments specific with assignees. Update comments when code changes.

See: https://blog.codinghorror.com/code-tells-you-how-comments-tell-you-why/
```

The full docs for rules can be found [here](https://recurse-ml.notion.site/Custom-Rules-21106defaa718059beabfb583f2ad066).

Now, every time a PR is submitted in your repo, a check will run, that verifies whether comments in the newly introduced code adhere to these guidelines.
If they don't, Recurse ML will leave a review comment.

For tips on how to write high quality rules, I recommend reading [bdougie's](https://x.com/bdougieYO) excellent blogpost [The Anatomy of Rules](https://blog.continue.dev/the-anatomy-of-rules-writing-effective-boundaries-for-ai-agents-in-ruby/).
Even though his examples are in Ruby, the principles are universal.

## Inspiration Gallery

If you're still wondering what rules to introduce into your repo, I've gathered some examples.

### Performance & Scaling

These rules catch the performance killers that don't show up until you're under real load.
For most applications, they're probably overkill - but if you're one of the lucky few startups with actual users, these patterns become critical.
They're designed to prevent issues that work fine in development but can bring down your entire API when traffic picks up.

Create `blocking-operations.md`:

```md
---
Name: avoid-blocking-operations
Description: Don't use synchronous operations in request handlers
Globs:
  - "routes/**/*.js"
  - "api/**/*.js"
  - "**/*handler*.js"
---

# Avoid Blocking Operations in Request Handlers

Don't use synchronous operations (fs.readFileSync, crypto.pbkdf2Sync) in request handlers as they block the event loop and kill performance under load. Use async alternatives or move to worker threads. One blocking operation can take down your entire API response time.
```

Create `database-queries.md`:

```md
---
Name: database-query-optimization
Description: Avoid N+1 queries and missing database indexes
Globs:
  - "**/*.js"
  - "**/*.ts"
---

# Database Query Optimisation

Avoid N+1 queries and missing database indexes. Use eager loading, batch queries, or data loaders. Flag loops that contain database queries or ORM calls. These issues don't show up in development but will crash production performance as data grows.
```

## Operational Reliability

Production systems have a way of failing in the most inconvenient moments.
These rules focus on the defensive coding practices that make the difference between a system that gracefully handles errors and one that silently loses data or crashes under pressure.
Most codebases won't need this level of rigour, but when reliability matters to your users (and your sleep schedule), these patterns become essential.

Create `error-handling.md`:

```md
---
Name: proper-error-handling
Description: Critical operations must have comprehensive error handling
Globs:
  - "**/*.js"
  - "**/*.ts"
---

# Proper Error Handling in Critical Paths

Critical operations (payments, user registration, data processing) must have comprehensive error handling with logging, monitoring, and graceful degradation. Avoid silent failures that could lose revenue or corrupt user data. Include error context for debugging production issues.
```

Create `memory-leaks.md`:

```md
---
Name: memory-leak-prevention
Description: Clean up resources to prevent memory leaks
Globs:
  - "**/*.js"
  - "**/*.ts"
---

# Memory Leak Prevention

Clean up event listeners, timers, and streams to prevent memory leaks. Use weak references for caches, clear intervals/timeouts, and properly close database connections. Memory leaks in Node.js can silently kill server performance and require expensive restarts.
```

### Wisdom from the Elders

What if instead of sending a link-dump of classic blogposts to new-hires, we could encode the knowledge right inside of the repo?
Wonder no more.
As you probably noticed, the example rule in Quickstart session was inspired by [Jeff Atwood's](https://blog.codinghorror.com/about-me/) [classic](https://blog.codinghorror.com/code-tells-you-how-comments-tell-you-why/).
I decided to distil some more classic software engineering blogposts into rules.

Create `dont-dry-prematurely.md`:

```md
---
Name: dont-dry-prematurely
Description: Consider carefully if code duplication is truly redundant
Globs:
  - "**/*.js"
  - "**/*.ts"
  - "**/*.py"
  - "**/*.go"
  - "**/*.java"
  - "**/*.rb"
  - "**/*.cs"
---

# Don't DRY Your Code Prematurely

Consider carefully if code duplication is truly redundant or just superficially similar before applying DRY principles. Functions or classes may look the same but serve different contexts and business requirements that evolve differently over time. Avoid premature abstractions that couple behaviours which may need to evolve separately. When in doubt, keep behaviours separate until enough common patterns emerge over time that justify the coupling. Think about long-term evolution, not just making code shorter. Apply YAGNI principle - tolerate a little duplication in early development stages and wait to abstract until clear patterns emerge.

See: https://testing.googleblog.com/2024/05/dont-dry-your-code-prematurely.html
```

Create `make-wrong-code-look-wrong.md`:

```md
---
Name: make-wrong-code-look-wrong
Description: Use naming conventions that make security vulnerabilities visible
Globs:
  - "**/*.js"
---

# Make Wrong Code Look Wrong

Use naming conventions that make security vulnerabilities and type mismatches immediately visible. For web applications, prefix all user-supplied strings with 'unsafe_' or 'us' and all sanitized strings with 'safe_' or 's'. For coordinate systems, use prefixes like 'window_x' vs 'layout_x' to distinguish semantic differences. Apply Apps Hungarian notation principles - use prefixes that indicate the semantic meaning or origin of data, not just the data type. Make code self-documenting by ensuring that dangerous or incorrect operations look obviously wrong at the point of use. Collocate all information needed to verify correctness in the same line of code.

See: https://www.joelonsoftware.com/2005/05/11/making-wrong-code-look-wrong/
```

Now, the rules act as a JIT learning tool for new-joiners, too!
And if your team is already using Cursor Rules for AI-assisted development, these same patterns will help maintain consistency across both human and AI-generated code.


## Your Usecase Here

Do you have a pet peeve that you fight for like it's the Battle of Verdun?
Is there part of your project that requires extra care during every PR?
Maybe you just found a weird and wacky way to use this feature?
How does this compare to your existing agent rule setup?
I'd like to hear all about it.
My email's armin at recurse.ml.
And for higher bandwidth comms, you can find our team on Discord: https://discord.gg/qEjHQk64Z9

