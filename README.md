# Git Engineering Workflow Lab

A practical Git reference for the workflow expected in professional backend teams.

## Professional workflow

~~~text
Requirement
   |
Feature branch
   |
Development + tests
   |
Commit
   |
Pull Request
   |
CI + Code Review
   |
Merge
   |
Deployment
~~~

## Everyday workflow

~~~bash
git status
git switch -c feature/merchant-search
git diff
git add .
git diff --staged
git commit -m "feat: add merchant search API"
git push -u origin feature/merchant-search
~~~

## Rebase vs merge

Merge preserves branch history and may create a merge commit.

Rebase replays feature commits on top of a newer base.

Use the approach required by the team. Avoid rewriting shared history without agreement.

## Conflict resolution

~~~text
<<<<<<< HEAD
current code
=======
incoming code
>>>>>>> origin/main
~~~

Resolve the file, stage it, continue the rebase/merge and run tests.

## Reset vs revert

Revert creates a new commit that reverses an earlier commit.

Reset moves the branch pointer and can rewrite local history.

Hard reset can discard local work, so it must be used carefully.

## Cherry-pick

Cherry-pick applies a specific commit to the current branch.

Useful for moving an isolated bug fix, but check dependencies before applying it.

## Secret safety

Never commit:
- Passwords
- API keys
- JWT secrets
- Private keys
- Database credentials
- Production configuration containing secrets

If a credential is committed, removing the visible file is not enough. Rotate the credential and clean the history when appropriate.

## Good commit messages

~~~text
feat: add merchant search API
fix: handle duplicate callback
refactor: extract transaction validation
test: cover expired offer scenario
docs: explain Kafka retry strategy
~~~

Avoid vague messages such as "update", "changes" or "final".

## Pull request checklist

- Code compiles
- Tests pass
- No secrets
- Relevant SQL reviewed
- API contract reviewed
- Error cases covered
- Unnecessary files removed
- Commit history understandable

## Interview questions

- fetch vs pull
- merge vs rebase
- reset vs revert
- cherry-pick use cases
- recovering a lost commit with reflog
- resolving conflicts safely
- protecting secrets in Git

## Planned exercises

Conflict resolution -> rebase -> cherry-pick -> reflog recovery -> GitHub Actions CI -> release/tag workflow.