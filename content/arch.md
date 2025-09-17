+++
date = '2025-09-14T15:45:21-07:00'
draft = true
title = 'Why Do LLMs Design Mediocre Architecture?'
+++

LLMs create mediocre architecture that compounds technical debt.
By "mediocre architecture" I mean code that's functionally correct but hard to maintain and expand.
In our example it's duplicated code without opinionated abstractions.

Where should we expect them to fail?
How can tech leads benefit from increases in velocity without eventually crumbling under tech debt?
How should teams change their working style as a result?

## Worked Example: Implementing Email Notifications

I ran Claude Code in the [FastAPI Example App](http://github.com/fastapi/fastapi-example-app) with the following prompt:

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

But here's the thing, if this were a PR, I'd reject it.

### Why This Doesn't Work

The existing approach to sending notification emails has reached its critical mass.
We now have 6 duplicate functions doing essentially the same thing.
About 60% of the endpoint code is just handling notification emails.

There are also two dangerous gotchas in the code:

1. The code catches bare `Exception`s and logs them at `info` level - a surefire way to leave logical bugs undiscovered forever.
2. The HTML templates used by generator methods don't actually exist.


### Conflicting Optimization Objectives

Better design would require removing the existing email function and replacing it with an abstraction.
But that means making opinionated decisions about the codebase.

This is RLHF-induced sycophancy at work.
Not only does it respond to you with "you're perfectly right!", it considers your code as such.
Well, unless you're terribly off course.
Claude saw one way of doing emails and assumed that was the way.

From a product perspective, this makes sense. I rarely want an LLM coding agent making radical changes to my codebase.

Instead of being a problem, this is an opportunity to delineate responsibilities.

## Reclaim Your Role as an Architect

When LLMs handle the implementation details, our role as reviewers fundamentally changes.
We're no longer line-by-line code inspectors catching syntax errors or style violations.
Instead, we become architectural guardians, focusing on the bigger questions:
Are we solving the right problem?
Is this the right abstraction?
Will this scale?

This shift isn't just about adapting to AI tools; it's about reclaiming the most intellectually rewarding part of software development—the design decisions that shape how our systems grow and evolve.

The first line of defence is a good ol' whiteboard meeting.
I've noticed myself pulling a colleague aside for a design discussion more frequently.
Especially as I know that once I have a solid high-level design, the (LLM-assisted) implementation will be easy.

Sometimes, just implementing a feature is faster than having a long discussion around it.
Normally, I'm hesitant to suggest drastic architectural changes in PRs.
The tradeoff between clean software architecture and time taken to implement it is rarely worth it in a startup.
Especially, if there's an existing solution.
However, as the code can be easily generated, that's no longer a valid concern.
That's why, when reviewing PRs, I now pay more attention to high-level design decisions.

As a reviewer, it is difficult to think about both the big picture design and line-by-line implementation all at once.
That's why I outsource most of the low-level thinking to automated tooling.
Tests and classic standard analyzers are helpful here.
Obviously we use Recurse ML as our safety net.

However, one issue I find myself battling with is that LLMs tend to repeat the same mistakes over and over again.
This is where [custom rules](https://docs.recurse.ml/gh/configs/rules/) are particularly helpful.
Think of them as code review comments that automatically check every PR.
When the tool spots code that violates these patterns, it leaves a comment explaining the issue.

I've learned from [Nate](https://natesesti.com) @ Continue.dev the following rule of thumb:

> Whenever I leave a comment in a PR, I ask myself, should this be an automatic rule?
> Most of the time the answer is "yes" and I create it.

For example, I created [`bare_exceptions.md`](https://github.com/continuedev/awesome-rules/blob/main/rules/recurse-ml/bare_exceptions.md) to prevent the bare `Exception` case from making it into our production codebase.


## Conclusion

LLMs create mediocre architecture because they optimize for consistency with existing patterns, not optimal design.
This isn't a bug—it's how they're trained to be helpful rather than opinionated.

This limitation is an opportunity.
When LLMs handle implementation, we get to focus on the hard architectural decisions that actually matter.

The practical takeaways:
1. Design first, implement second. Pull a colleague aside for that whiteboard session.
2. Review PRs for architecture, not syntax. Let tests and linters catch the small stuff.
3. Turn repeated review comments into automated rules. If you're saying it twice, automate it.
4. Don't hesitate to suggest refactoring. When implementation is cheap, good design becomes affordable.

Embrace the promise of LLMs to free humans from rote work towards creative high-level tasks.
Designing elegant software is hard.
It requires understanding existing code, having a mental model of people who will maintain it and the environment it will run in.
However, it's also fun.
It's creative in the same way that writing or visual art is creative.
The space of viable solutions is vast.
Greatness is highly subjective and context-dependent.
But creating something, that other people appreciate and build upon is one of the most rewarding parts of the human experience.
