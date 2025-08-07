# Continue X Recurse ML: Just Rewrite it in Rust

## The Problem

Every development team has that "just rewrite it Rust" moment
Usually, it stays a joke.
But I'm a firm believer that science is not about asking "why?".
It's about asking "why not?"

So here we go...

`rml` is our CLI tool that finds bugs through static analysis.
Works fine in Python, but I was curious about Rust performance without the Rust learning curve.
More importantly, I wanted to see if Continue could handle the port while using `rml` itself to catch issues during development.

The real question: could I go get a sandwich and come back to working Rust code?

Answer: kinda

## The Setup

Install `rml`:

```bash
curl install.recurse.ml | sh
```

Install [Continue](https://www.continue.dev/) in VS Code. Enable "Auto-Accept Agent Edits" and "Add Current File by Default" in experimental settings. This is giving your intern commit access - thrilling and terrifying.

Download the [rml-verify rule](https://github.com/continuedev/awesome-rules/blob/main/rules/recurse-ml/rml-verify.md):

```bash
curl https://raw.githubusercontent.com/continuedev/awesome-rules/refs/heads/main/rules/recurse-ml/rml-verify.md -o .continue/rules/rml-verify.md
```

This rule makes Continue automatically run RML verification during development. Like having a code reviewer who never takes coffee breaks.

## The Walkthrough: What Happened During Lunch

I gave Continue this prompt in agent mode:

```
Write a basic clone of this CLI in Rust.
```

Then I left for lunch. Either the future of software development or the most expensive way to generate broken code.

### RML Catches Its Own Rewrite

Continue examined the Python codebase, created `Cargo.toml`, implemented the CLI in `src/main.rs`, and ran RML to check its work.

RML immediately caught a critical issue:

```
## src/main.rs

### Issue 1 (Line 18)
```diff
+const HOST: &str = "https://example.com";

 HOST constant value mismatch between Python and Rust implementations. The Python version uses 'https://squash-322339097191.europe-west3.run.app' while the Rust version uses 'https://example.com'. This will cause runtime failures.
```

Continue fixed this instantly. The tool debugged itself. Very meta, slightly unsettling.

### The API Format Bug Hunt

I returned to working Rust code that compiled. But testing revealed the real issue:

```bash
$ ./target/debug/rml src/rml/__init__.py
🔍 Analyzing code changes...
❌ Error: error decoding response body: invalid type: integer `201`, expected a string
```

The Rust implementation was sending form data wrong. Classic "works in Python but not in Rust" problem.

The fix:

```diff
-    // Add each filename as separate form field (creates array)
-    for filename in target_files {
-        form = form.text("target_filenames", filename.clone());
-    }
+    let form = multipart::Form::new()
+        .part("tar_file", file_part)
+        .text("target_filenames", target_files.join(","));
```

One comma-separated string instead of multiple form fields. The difference between array and string that breaks everything.

## Results: The Feedback Loop Actually Works

The self-analyzing setup worked. RML caught the HOST constant issue that would have caused production failures. The API format bug showed up during testing, proving you need both static analysis and end-to-end validation.

It's like having a really paranoid pair programmer who actually catches the bugs you'd miss.

## Your Rewrite Story Here

Try this setup and let me know whether it's more effective than your last intern.
What weird edge cases did Continue discover in your codebase? 
I'm particularly curious about other self-analyzing setups; it's in the name!

I can finally join the "rewrite it in Rust" bandwagon, without really knowing Rust.
Though I'm still not sure if this makes us ship faster or procrastinate longer.

My email's armin at recurse.ml.
And for ninja technical (and emotional) support you can find our team on Discord: https://discord.gg/qEjHQk64Z9

