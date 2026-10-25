# AI code review playbook

Lessons from running an AI pull-request reviewer in production. This is a generic, anonymized write-up. It contains no company code, service names or internal data.

Author: [Ali Nikneshan](https://www.linkedin.com/in/ali-nikneshan/), CTO at BITMAX. Questions: Telegram [@nikneshan](https://t.me/nikneshan).

## 1. Retrieve by key when you know the key

The reviewer first used semantic search over its knowledge base and pulled the nearest chunks into each review. We then sampled 30 review runs:

- 91% of the retrieved chunks were stale or already in the prompt.
- No finding used a retrieved rule.

We removed the search and replaced it with a lookup by repository key. When you know which service a pull request belongs to, you do not need a nearest neighbour, and similarity can surface another service's notes.

**Rule:** measure retrieval before you keep it. Count how often a retrieved chunk changes a finding.

## 2. Keep the knowledge cards fresh

Each service has one short knowledge card (what it does, who calls it, what must not break, past incidents). The reviewer reads the card for the service under review.

- **Stamp every card** with the commit it was generated from.
- **Call a card stale** when it is far behind its repository: 25 or more commits, or 14 or more days old with any drift.
- **Sync on a schedule.** A nightly job pushes the latest cards to wherever the reviewer reads them. A weekly job posts one alert listing stale cards.
- **Regenerate by hand.** Generating cards uses a model, so it stays a human-started step. The scheduled jobs only copy and alert.
- **Never store values that change** (migration heads, versions, counts). Point to the source of truth instead. One card sat 73 commits behind and produced a false major finding about a value it had pinned.

A minimal staleness check, in generic form:

```python
import subprocess
from datetime import datetime, timezone

MAX_COMMITS_BEHIND = 25
MAX_DAYS_WITH_DRIFT = 14

def commits_behind(repo_path: str, card_commit: str) -> int:
    out = subprocess.run(
        ["git", "-C", repo_path, "rev-list", "--count", f"{card_commit}..HEAD"],
        capture_output=True, text=True, check=True,
    )
    return int(out.stdout.strip())

def is_stale(repo_path: str, card_commit: str, generated_at: datetime) -> bool:
    behind = commits_behind(repo_path, card_commit)
    age_days = (datetime.now(timezone.utc) - generated_at).days
    return behind >= MAX_COMMITS_BEHIND or (behind > 0 and age_days >= MAX_DAYS_WITH_DRIFT)
```

Treat a card whose repository or commit you cannot resolve as stale.

## 3. Give the AI a small, reviewable job

- The reviewer only reviews. It posts comments and, at most, narrow safe fixes such as formatting and import order.
- After 5 bot rounds with a score still below the bar, a second pass (a judge) posts a merge or do-not-merge comment. A comment, not a merge.
- A bug fixer opens a pull request, or only a written plan when the evidence is thin. A wrong fix is worse than no fix.
- **By rule, the AI does not merge, deploy or force-push.** Say honestly which of these limits are enforced by tooling and which only by the prompt.
- Compute the score from finding severities in code, not in the model.

## 4. Fail loudly

- Sixteen reviews once failed silently after a login was revoked. Now three failures in a row raise an alert.
- A fixer once closed tickets as "already fixed" on branches nobody had merged. Now a ticket closes only when the fix is on the base branch, checked by a machine-readable line, not by the model's word.
- Match comment commands on whole lines, and de-duplicate per commit. A substring match once triggered a duplicate judge run.

## 5. Short checklist

1. Can you say what each retrieved chunk changed in a finding?
2. Does every knowledge card carry a source commit?
3. Is there an alert for stale cards and one for repeated failures?
4. Which actions can the AI never take, and what enforces that?
5. Who regenerates cards, and how often?

## License

Text: CC BY 4.0. Code snippets: MIT.
