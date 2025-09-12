+++
date = '2025-09-12T17:03:33+01:00'
draft = true
title = 'Clean Code at High Velocity'
+++

<!-- TODO: the article uses the term Clean Code to refer to general software engineering principles, what's a more appropriate term? -->

Most advice on writing maintainable software ignores the pressure of shipping quickly.
Proponents of clean code would argue that writing the best software one can pays off in the long run.
After all, code debt needs to be paid off eventually.
Why not pay it off immediately?
We can get to work with beautiful software from day one.

This line of thinking works well at companies that have firmly found product market.
When majority of code written is likely to live in the codebase long after its author has left the company.

However, this advice feels out of place when working under the pressure of a pre-PMF startup.
The codebase should facilitate rapidly learning from users.
The product is likely to change.
Large parts of the codebase are likely to be discarded, upon pivoting.
Hence, investing in making the code clean feels like a sidequest.

> My code is a dumpster-fire but we ship fast!
-- Every YC Founder

Experienced lead engineers know when to compromise the "best practices" and when to embrace them.
In this blog post I describe my principles for achieving the Goldilocks zone of clean code and velocity.


Key principles:

1. Solve problems iteratively.
2. Implement best practices top-down.
3. Notice the smells.

## Solve problems iteratively

You might be smart but you're not smart enough to predict the future.
Focus on understanding and solving the problems you're experiencing today, rather than predicting the future.
Then, iteratively improve on the foundations you've created as future issues arise.

Paul Graham compares the nature of iterating on a software product to working on an oil painting.
Oil paint is forgiving to mistakes.
It allowed the painter to work on the same surface for weeks or even months.
That's why invention of oil paint correlates with significant improvements in the quality of produced paintings.

We should apply the same mindset when thinking about clean code.
Instead of trying to follow every single "best practice" upfront, look out for things that are slowing you down.
Try finding the work that will yield 80% of the speedup with 20% of the effort.
Repeat this process as your code and team expands.
The focus should be on ensuring a trend of gradual improvement.

Refactoring should never be a major stand-alone project.
Instead, continuously ship improvements in response to code-smells.
Here's the mental model I use when refactoring.
Organize your own refactoring work in such way, that if you'd have to switch priorities and drop the project, you could commit the existing work and improve the codebase.

### Failure Recovery Over Prevention

An iterative approach means you should optimize for failure recovery over prevention.
There is a natural temptation to prevent production outages.
However, failure isn't preventable.
Consequently, most effort aimed at preventing production failures slows down production deployments and instils a false sense of confidence in the deployed software.

Instead, optimize for fast failure identification and resolution.

**Identification:**
1. Logging: you should be able to just make sense of your logs.
2. Alerting: if you don't know whether your production app is working or not you should implement basic alerting.
    For example, triggering an alert upon any `error` log should do the trick.
    5xx error codes are another good source.
    With increased traffic, keeping the rate of failures at exactly zero unfortunately becomes unfeasible.
    If alerts become too noisy, either your user experience has taken a hit or you should filter them more carefully.
    Make sure not to ignore alerts.

**Resolution:**
1. A basic CI/CD should ensure that you can quickly ship a bugfix or at the very least roll back the faulty deployment.
2. This is the core component of what I mean by optimizing for error recovery instead of prevention.
    You want to create a culture where your team feels comfortable making changes to production quickly.
    Why:
    1. Avoids known issues staying in the code.
        When a developer sees a typo in product's copy or a minor functional bug, they should be able to fix it at the speed of thought.
        If the process of shipping solution involves multiple manual steps, writing the fix will be a significant detour from the previous issue they were working on.
        This means the issue is likely to be backlogged and is unlikely to be fixed.
        This in turn, creates a cascading effect where known minor issues are left in the product.
        Like broken windows, lead to increase in crime, dangling known product issues lead to eventual enshitification.
    2. Ensures codebase doesn't become stale.
        Decreasing the time required to ship minor refactors, ensures that a poor variable name or convoluted function logic can be fixed with a quick PR.



## Implement Best Practices Top-Down

To identify high-impact improvements focus on small changes that cover as much of your codebase as possible.
For example, in code review it's tempting to focus on immediately obvious nit-picks.

Same principle applies to code review.
It's easy to focus the review conversation on low-level nitpicks.
Spending the time debating whether to use a list comprehensions or a `map` has minimal impact on code quality.
On the other hand, deciding whether to use multiple workers with a redis queue or a single one has major impact.
In code review focus on major decisions first.
Here's a rough priority rank:
1. infrastructure choices
2. DB Model Design
3. Class and/or key function design
4. Class and/or function implementation
5. Formatting

TODO: describe how low level discussions can be removed through automatic tooling incl. Recurse ML.

CI/CD: CI/CD streamlines the single most important part of programmer's job -- shipping to prod.
Therefore, it is unsurprising how much positive impact a decent CI/CD has on engineering team's productivity.
It should be equally unsurprising that engineers have a temptation to over-engineer it.

<!-- TODO: create a comic:
PM: what can we ship in a month?
Dev: basic payments and cart and a basic CI/CD
PM: what about three months?
Dev: basic payments and cart with an excellent CI/CD?
-->

Here's the CI/CD priority stack rank:
1. CD: ensure that you automatically build and deploy from your main branch (or however you choose to organize your git workflow).
    Your build will already pick up the
2: CI: run the compiler and linter.
3. Implement automated testing.


Automated testing should also follow a similar priority ranking.
If you don't have any tests, begin with E2E tests.
A single well-designed E2E test, means you should be able to spot major bugs in any component on your hot-path.
They are more realistic than unit-tests, so the number of bugs prevented through a single inference run is higher as well.


Notice the smells:

1. Extensive use of mocks is a red flag for three reasons:
    (i) not really testing anything.
        Mock is an assumption.
        A test with a lot of mocks, indicates that it makes a lot of assumptions.
        Instead a unit-test ought to test internal logic.
    (ii) poor abstraction.
        If you tried your best to test internal logic but needed to mock it, you should split the code responsible for fetching the data from the processing logic.
        Then you can test the processing logic without the burden of mocks.
    (iii) poor dev experience: mock heavy tests are annoying to write and frustrating to maintain as they break due to implementation changes.
2. Simple features require changing 5+ files.
    This is a sign of poorly designed abstractions.
    The job of an abstraction is to limit the amount of context programmer needs to hold in their head at any one time.
    Having to juggle multiple abstractions means they are not doing their job.


How does a good abstraction look like?
