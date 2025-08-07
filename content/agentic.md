+++
date = '2025-07-09T16:01:07+01:00'
draft = false
title = 'Continue X RML'
+++


## The Problem

`rml` is our CLI tool that identifies bugs in local files.
The client code is open-source, so we'll be using it as an example.
I'll be using Continue with `rml` to edit `rml` [source code](https://github.com/recurse-ml/rml).
In particular, users should be able to provide directories in addition to filenames for analysis.
This involves a couple of moving parts, as we need to filter the changed files that match the directory.

1. I let Continue's agent run
2. While it's running, I go away for a lunch.
3. When I come back, the cost of it being stuck in an error is high, as it requires me to debug and restart the agentic workflow.
    I want to improve the success rate of Continue just getting

## Continue 

1. Open Source
2. Highly customizable
  1. Can use on-prem or ollama deployments
  2. You can select your own apply models
3. Beyond the tool, [Continue Hub](hub.continue.dev) is an ecosystem of rules, models, and tools.
  1. Recurse ML integrates into this ecosystem.
  2. I've already described our [rule support](rules).
  3. This post will focus on how Continue's agent mode can use our CLI tool [`rml`](https://docs.recurse.ml/rml/) for 

## Prerequisites

### RML: Recurse ML CLI Tool

You can install `rml` using the following one-liner:

```bash
curl install.recurse.ml | sh
```

_For more info check out our [docs](https://docs.recurse.ml/rml/) or [contact us](https://docs.recurse.ml/rml/help-and-support) for help._

### Continue

You can install [Continue](https://www.continue.dev/) as a plugin in VS Code or JetBrains.
It even supports your favourite VS Code fork 😉.

## The Setup



### Auto-Accepting Edits for Extra Speed

We want to allow Continue to autonomously run 

![Image of settings with Auto-Accept Agent Edits and Add Current File by Default enabled](configs.png)

1. Enable experimental settings:
  1. Go to Continue settings (the cog on the right hand side of the panel).
  2. Under Experimental settings enable Auto-Accept Agent Edits and Add Current File 

### Allowing Continue to Run `rml`

Download the [rml-verify rule](https://github.com/continuedev/awesome-rules/blob/main/rules/recurse-ml/rml-verify.md) from [awesome-rules](https://github.com/awesome-rules).

```bash
# Ensure .continue/rules/ exists
curl https://raw.githubusercontent.com/continuedev/awesome-rules/refs/heads/main/rules/recurse-ml/rml-verify.md -o .continue/rules/rml-verify.md
```


