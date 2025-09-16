+++
date = '2025-09-14T15:45:21-07:00'
draft = true
title = 'How do LLM and human coding mistakes compare?'
+++



# Mediocre Architecture Decisions

## Intro

A common trap I've noticed while working with LLMs on code architecture decisions is that they'll come up with plausibly incorrect architectural decisions.
LLM's architectural choices would produce a mediocre but passing design at best.
At worst, it's a plausibly incorrect design that ignores external considerations the LLM is unaware of.

## Mediocre but passing design

Traits:
1. Internal bloat
    -> methods or even entire classes that are not necessary for the given problem
2. Ignores existing code design structure
    -> duplication
    -> inconsistency
3. Lack of taste and elegance.
    I think this is largely the byproduct of the two others.
    Designing a specific solution that fits a given problem is beautiful.
    It can be brief and specific.
    Whereas, LLMs have a bias towards following generic class structures.
    This makes the code feel bland.

Dangers:
1. The bloat is self-propogating. <!-- TODO: mention DHH's objections to IDEs here -->
    When code is bloated, making even basic changes becomes slow.
    This means a developer is likely to rely on LLM for it.
    The LLM is likely to produce a more verbose solution that a human would.
    This gets us back to square one.
2. Lack of internal consistency within the project.
    This makes the project's code fragmented and hard to reason about.
    Again, this means it's harder for a human to make changes in a project without LLM assistance.
    Therefore, the issues propagate further and further.
3. This makes it harder to 

## Solution

During review, focus on high-impact decision points.
Determining the class structure has high leverage.
This decision propagates down into method definitions and implementations.

Decisions around implementation of particular methods have significantly lower impact across the entire project.
LLMs are also less likely to make mistakes at that level.


## Worked Example: Implementing Email Notifications

I ran Claude Code in [FastAPI Example App](http://github.com/fastapi/fastapi-example-app) with the following prompt:

> Add a notification system to this FastAPI app that sends email alerts when users perform certain actions (user registration, password reset, important data changes).
> The system should support multiple notification channels in the future (SMS, push notifications) and handle failures gracefully with retry logic.

### Claude Code's Implementation

Claude found an existing function for email notifications `generate_new_account_email` and followed a similar pattern in defining the new functions:     `generate_admin_account_status_change_email`, `generate_admin_profile_update_email`, `generate_email_change_notification`, `generate_new_account_email`, and `generate_profile_update_email`.

Here's an example for one of them:

```python
def generate_admin_account_status_change_email(
    email_to: str,
    full_name: str | None,
    status_change: str,
    admin_email: str,
    timestamp: str,
    reason: str | None = None
) -> EmailData:
    project_name = settings.PROJECT_NAME
    subject = f"{project_name} - Account Status Update"
    html_content = render_email_template(
        template_name="admin_account_status_change.html",
        context={
            "project_name": project_name,
            "full_name": full_name or "User",
            "status_change": status_change,
            "admin_email": admin_email,
            "timestamp": timestamp,
            "reason": reason,
            "login_link": settings.FRONTEND_HOST,
            "support_link": f"{settings.FRONTEND_HOST}/support",
        },
    )
    return EmailData(html_content=html_content, subject=subject)
```

And here's how it's used:

```python
def update_user(
    *,
    session: SessionDep,
    user_id: uuid.UUID,
    user_in: UserUpdate,
    current_user: CurrentUser,
) -> Any:
    # 20 lines of endpoint handling code

    # All code until `return` is new notification logic
    if settings.emails_enabled:
        user_data = user_in.model_dump(exclude_unset=True)
        changed_fields = get_changed_fields_display(old_data, user_data)
        timestamp = format_timestamp()

        # Check if account status was changed
        if "is_active" in user_data and user_data["is_active"] != old_data["is_active"]:
            try:
                status_change = "Activated" if user_data["is_active"] else "Deactivated"
                email_data = generate_admin_account_status_change_email(
                    email_to=db_user.email,
                    full_name=db_user.full_name,
                    status_change=status_change,
                    admin_email=current_user.email,
                    timestamp=timestamp,
                )
                send_email(
                    email_to=db_user.email,
                    subject=email_data.subject,
                    html_content=email_data.html_content,
                )
            except Exception as e:
                logger.info(f"Failed to send status change email to {db_user.email}: {e}")

        elif changed_fields:
            try:
                email_data = generate_admin_profile_update_email(
                    email_to=db_user.email,
                    full_name=db_user.full_name,
                    changed_fields=changed_fields,
                    admin_email=current_user.email,
                    timestamp=timestamp,
                )
                send_email(
                    email_to=db_user.email,
                    subject=email_data.subject,
                    html_content=email_data.html_content,
                )
            except Exception as e:
                logger.info(f"Failed to send admin update email to {db_user.email}: {e}")

    return db_user
```

Here Claude has successfully understood the existing code structure and has created a new function that follows a similar pattern as before.
The functions to generate emails and their usage are nearly identical with the existing code.

However, if this were a PR I'd reject it.

## Abstractions

The existing approach to sending notification emails has reached its critical mass.
Now we have 6 duplicate functions that do the same thing.
~60% of endpoint code is responsible for sending notification emails.

## Logic

There are two dangerous gotyas in the code:

1. The code catches bare `Exception`s and logs them at an `info` level.
    This is a surefire way to leave undiscovered logical bugs in laying in the code forever.
2. The html templates used by generator methods (e.g. `admin_account_status_change.html` in the example above) don't exist.

## The Difficulty

1. Context switching between two modes of thinking is difficult.
    I use automated software to take the cognitive load of the low-level functional issues.
2. When programmers write code together with LLMs the architectural issues are more likely to persist over functional ones.

## Implications

This changes how I review code:
Spend less time thinking about the PR at individual line level and more time thinking about the high level architecture.
1. Does the PR solve the problem it reports to solve?
    It's possible that the author was mislead by the coding agent
2. Are the software architecture decisions sound?
3. Does the new architecture introduce duplication with existing code?
    Can we leverage the newly introduced classes to solve some existing challenges?
    Ironically, asking these questions to an LLM is not that bad of an idea.

The first line of defence are good ol' whiteboard meetings.
I've noticed myself pulling a colleague aside for a design discussion more frequently.
Especially as I know that once I have a solid high-level design, the (LLM-assisted) implementation will be easy.

Sometimes, just implementing a feature is faster than having a long discussion around it.
Normally, I'm hesitant to suggest drastic architectural changes in PRs.
The tradeoff between clean software architecture and time taken to implement it is rarely worth it in a startup.
Especially, if there's an existing solution.
However, as the code can be easily generated, that's no longer a valid concern.
That's why now, when reviewing PRs, I pay more attention to high-level design decisions.

As a reviewer, it is difficult to think about both the big picture design and line-by-line implementation all at once.
That's why I outsource most of the low-level thinking to automated tooling.
Tests and classic standard analyzers are helpful here.
Obviously we use Recurse ML as our safety net.

However, one issue I find myself battling with, is that LLMs tend to repeat the same mistakes over and over again.
This is where [custom rules](https://docs.recurse.ml/gh/configs/rules/) are particularly helpful.
I've learned from [Nate](https://natesesti.com) @ Continue.dev the following rule of thumb:

> Whenever I leave a comment in a PR, I ask myself, should this be an automatic rule?
> Most of the time the answer is "yes" and I create it.

## Conclusion

Embrace the promise of LLMs to free humans from route work towards creative high-level tasks.
Designing elegant software is hard.
It requires understanding existing code, having a mental model of people who will maintain it and the environment it will run in.
However, it's also fun.
It's creative in the same way writing or visual art is creative.
The space of viable solutions is wast.
Greatness is highly subjective and context-dependent.
But creating something, that other people appreciate and build upon is one of the most rewarding parts of the human experience.

# Relevant resources

1. https://cendyne.dev/posts/2025-03-19-vibe-coding-vs-reality.html
2. https://simonwillison.net/2025/Mar/11/using-llms-for-code/


# [llm draft] Potemkin problem


## 1. The "Locally Correct but Globally Inconsistent" Pattern

The paper provides strong empirical backing for your core insight. From the paper:

> "LLMs often produce code that's locally correct but globally inconsistent - each function works perfectly in isolation but they make incompatible assumptions about data structures or state management."

The concept of "potemkin understanding" directly maps to system design - LLMs can correctly explain architectural patterns (the keystone) but fail when actually implementing coherent systems. They exhibit what appears to be understanding through correct definitions and explanations, but this breaks down when they need to maintain consistency across a larger system.

## 2. The Incoherence Problem

The paper's finding about conceptual incoherence (Section 4.1) is particularly relevant. They found that models often contradict their own outputs - generating something and then later evaluating it as incorrect. This directly explains why LLMs produce architecturally inconsistent designs:

> "Their conceptual grasp is incoherent, with conflicting notions of the same idea... This indicates that conceptual misunderstandings arise not only from misconceiving concepts, but also from inconsistently using them."

For your blog post, this means LLMs don't just misunderstand system design principles - they apply them inconsistently. An LLM might correctly use microservices patterns in one part of the system while violating those same patterns elsewhere, not because it doesn't "know" the patterns, but because its understanding is fundamentally incoherent.

**How to incorporate these findings:**

You could frame LLM system design failures as "Potemkin Architecture" - designs that appear correct when examined piece by piece but fail to cohere into a working system. The paper's framework explains why code review practices designed for human errors fail for LLM code: humans make consistent mistakes based on misconceptions, while LLMs make inconsistent mistakes based on incoherent understanding. This makes traditional architectural review even more critical, as you need to check not just whether each component is correct, but whether the components share a coherent understanding of the system.



