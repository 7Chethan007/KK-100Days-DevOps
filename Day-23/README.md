# Day 23 — Fork a Git Repository

A KodeKloud "100 DevOps" lab. Read **§2 Reasoning model** and **§3
Concepts** first — the runbook is just mechanical execution once you
understand *why* each action exists and *how you'd derive it yourself*.

Days 21-22 worked entirely at the Git CLI/filesystem layer — a bare
repo, a clone. This lab moves to a Git *hosting platform* (Gitea) and
a third repository relationship neither prior lab introduced: a fork,
which is Git-platform-level, not something `git clone`/`git init`
alone can produce.

---

## 1. Scenario

Fork `sarah/story-blog` into Jon's Gitea account, producing
`jon/story-blog`.

| Step | Action |
|---|---|
| Account | Log into Gitea as `jon` |
| Locate | Find `sarah/story-blog` |
| Fork | Click Fork, select `jon` as destination owner |
| Verify | `jon/story-blog` exists, references `sarah/story-blog` as its source |

```text
sarah/story-blog  (original)
        │  Gitea's Fork action
        ▼
jon/story-blog    (fork — full history copied, source relationship recorded by Gitea)
```

---

## 2. Reasoning model — how to *derive* the steps, not memorize them

### 2.1 A fork is a platform-level relationship, not a `git` CLI operation

```text
git clone <url>   → produces an independent copy; Git itself tracks no "forked from" relationship
Gitea Fork button  → produces a new repository AND records the source relationship in Gitea's own data
```

Nothing in Git's own object model has a concept of "this repository is
a fork of that one" — forking is a feature the *hosting platform*
(Gitea, GitHub, GitLab) layers on top of Git, by copying the
repository's data into a new location under a different owner and
separately recording the provenance link in its own database. This is
why the lab requires using Gitea's Fork button specifically, rather
than manually cloning and re-creating a new repository by hand — a
manual recreation would copy the *content* but not establish the
*relationship* the platform (and this lab's verification) actually
checks for.

### 2.2 Fork vs. clone — same underlying copy, different scope and ownership

```text
Clone (Day 22):  copies a repo into a NEW LOCAL WORKING DIRECTORY, same ownership/account context
Fork (Day 23):   copies a repo into a NEW REPOSITORY under a DIFFERENT ACCOUNT, on the SAME platform
```

Both operations copy repository history, but they solve different
problems: cloning gets you a local working copy to edit; forking gets
you your *own remote, platform-hosted* copy you can push to
independently, without needing write access to the original. This is
why forking matters specifically for collaboration across accounts —
`jon` gets a repository he fully owns and controls, derived from
`sarah`'s, without `sarah` needing to grant him any access to her own
repository.

### 2.3 Why verify identity and location before searching, not after

```text
Log in as jon FIRST → every subsequent action happens in Jon's account context
Search for story-blog → multiple repos could share a similar name across different owners
```

Confirming you're logged in as the *correct* account before doing
anything else matters because the fork destination defaults to
whichever account initiated it — forking while accidentally logged in
as the wrong user would produce a fork under the wrong owner entirely.
Searching by name alone (`story-blog`) could also surface
similarly-named repositories under other accounts; confirming the
exact `sarah/story-blog` (owner *and* name together) before forking is
the same "exact identifier, not an approximate match" discipline as
every CLI `--filters`/exact-path check in this series — just performed
by reading the UI carefully instead of passing a flag.

### 2.4 Selecting the destination owner is the one place user intent actually matters

```text
Gitea may prompt: "Fork to which account/organization?"
```

On platforms where a user belongs to multiple organizations, the fork
destination isn't automatically "whichever account you're logged in
as" — it can offer a choice. Explicitly selecting `jon` (rather than
accepting whatever default appears) is the step that actually
determines the lab's required outcome (`jon/story-blog` specifically),
the same way Day 20's AWS console lab required selecting the *exact*
policy name rather than trusting a default-looking option.

### 2.5 Verification means confirming both ends of the relationship, not just that *a* repository exists

```text
jon/story-blog exists           → necessary, but doesn't prove it's a FORK
jon/story-blog's page shows      → the decisive confirmation: source relationship recorded
"forked from sarah/story-blog"
```

A repository literally named `story-blog` existing under `jon`'s
account doesn't by itself prove it originated as a fork of `sarah`'s
— someone could also have manually created a same-named, unrelated
repository. Checking that Gitea's UI explicitly displays the fork
relationship (source repo named, history shared) is the decisive
check, the same "existence isn't the same as correctness" pattern as
Day 22's `remote -v` check (a working tree existing doesn't prove
`origin` points at the right source).

### 2.6 The compressed reasoning chain

```text
Requirement: fork sarah/story-blog → jon/story-blog
   → log in as jon FIRST — fork destination follows the logged-in account (§2.3)
   → locate the EXACT repo: owner AND name together, not name alone (§2.3)
   → use Gitea's Fork button — a manual clone+recreate would copy content but NOT the
     platform-level "forked from" relationship Gitea tracks (§2.1)
   → if prompted, explicitly select jon as the destination owner (§2.4)
   → verify BOTH: jon/story-blog exists, AND its page shows the fork relationship to
     sarah/story-blog (§2.5) — existence alone isn't sufficient
```

---

## 3. Concepts (reference)

### 3.1 Fork (platform-level)
A hosting platform's feature that creates a new, independently-owned
repository copy, while recording a provenance link back to the
original — a relationship Git itself has no native concept of.

### 3.2 Fork vs. clone
A clone produces a local working copy under the same account/access
context as the source. A fork produces a new, independently-owned
remote repository under a *different* account, enabling collaboration
without requiring write access to the original.

### 3.3 Destination owner selection
On platforms supporting multiple accounts/organizations per user, the
fork destination is an explicit choice, not always implicitly "the
account you're logged in as" — confirming it matters for ensuring the
fork lands where intended.

### 3.4 Verifying a relationship, not just an artifact
Confirming a derived resource exists is necessary but not sufficient
when the requirement is about a *relationship* (fork-of, remote-of,
attached-to) — the relationship itself needs its own explicit check.

---

## 4. Runbook

### 4.1 Log into Gitea as `jon`
```text
Click the Gitea UI button in the lab's top bar
  → Username: jon
  → Password: Jon_pass123
```

### 4.2 Locate the exact source repository
```text
Search: story-blog
  → confirm owner is sarah → sarah/story-blog
```

### 4.3 Fork the repository
```text
Open sarah/story-blog → click Fork
  → if prompted, select jon as the destination owner
  → confirm
```

### 4.4 Verify the fork
```text
Confirm the new repository is: jon/story-blog
  → confirm the repository page identifies the original as sarah/story-blog,
    wherever Gitea displays the fork relationship
```

### 4.5 Submit the lab
```text
Capture a screenshot showing the fork under Jon's account
  → return to the lab page → click Check / Submit
```

---

## 5. Final state

| Requirement | Result |
|---|---|
| Logged into Gitea as `jon` | ✅ |
| Located the exact source `sarah/story-blog` | ✅ |
| Forked via Gitea's Fork button (not a manual clone+recreate) | ✅ |
| Fork destination explicitly `jon` | ✅ |
| `jon/story-blog` exists | ✅ |
| Fork relationship to `sarah/story-blog` confirmed on the repo page | ✅ |

```text
Gitea
 ├── sarah/story-blog   (original)
 └── jon/story-blog     (fork — history copied, source relationship recorded)
```

---

## 6. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Fork lands under the wrong account | Logged in as the wrong user, or wrong destination selected | Log out, log back in as `jon`; re-select the destination explicitly during fork |
| Can't find `sarah/story-blog` | Search matched a similarly-named repo, or wrong spelling | Confirm exact owner AND name together, not name alone |
| New repo exists but isn't actually a fork | Repository was manually created/cloned instead of using Gitea's Fork button | Delete the manually-created repo; use Fork specifically — it's the only action that records the platform-level relationship |
| Fork button missing or disabled | Already forked, or permissions issue on the source repo | Check if `jon/story-blog` already exists; check source repo's fork settings |
| Lab check fails despite the fork existing | Fork relationship not actually recorded, or wrong repo name | Confirm the repo page explicitly shows "forked from sarah/story-blog" |

---

## 7. Do it yourself (checklist)

- [ ] Without looking above, walk the reasoning chain (§2.6) out loud
      from "log in as `jon`" to "verified the fork relationship."
- [ ] Explain, in one sentence, why a fork is not something `git
      clone` or `git init` alone can produce.
- [ ] Explain the difference between what cloning (Day 22) and
      forking (this lab) each actually accomplish.
- [ ] Explain why confirming `jon/story-blog` exists isn't, by itself,
      proof that it's actually a fork of `sarah/story-blog`.
- [ ] Explain why logging in as the correct account matters *before*
      clicking Fork, not after.
- [ ] From memory, fork a second repository into a different
      destination (if your Gitea account belongs to an organization),
      and confirm the fork relationship is recorded correctly for that
      destination too — without copying any step from this file.

---

## 8. Document template for future lab days

Use this skeleton for `day-N/README.md`:

```markdown
# Day N — <task title>

## 1. Scenario
<what was asked, which account/region/resources, constraints, credentials
if relevant>

## 2. Reasoning model — how to derive the commands
<walk the requirement down to a subsystem/resource hierarchy, step by step,
in the order you'd actually discover it: "what resource, is it global or
region-scoped, what existing state constrains my choice, what CLI
service+operation performs the read (exact lookup vs. filtered search),
what performs the write, how do I verify the SPECIFIC field the
requirement names." End with a compressed step-chain.>

## 3. Concepts (reference)
<one subsection per concept the task exercises — explain WHY, not just WHAT>

## 4. Runbook
<copy-pasteable commands in the order run, WITH the actual intermediate
output/results captured inline as code blocks, not just the commands>

## 5. Final state
<table + diagram of what the infrastructure looks like after completion>

## 6. Troubleshooting
<symptom / cause / fix table>

## 7. Do it yourself (checklist)
<"walk the reasoning chain out loud" + comprehension questions +
"redo without copy-pasting, recompute derived values" prompt>
```
