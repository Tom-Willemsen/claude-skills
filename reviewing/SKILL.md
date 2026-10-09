---
name: reviewing
description: General code-review guidance. This skill should be used when a code review is requested.
---

When code-reviewing, you should generally assume that you are reviewing code written **by a human**. It will be a human responding to your comments. You should therefore keep your comments brief, focus on the most pressing issues, and avoid creating unnecessary work.

If a changeset is acceptable on balance, **say so**. Always ensure the scope of your review comments is proportionate with the issue being solved. Avoid suggesting large refactors or design changes unless there is a clear justification for why the refactor is necessary.

The resulting code must be functional and maintainable (by humans). Assume that a future human working on the code may be a non-expert, reading the code in a hurry. Simple, readable code is generally preferred over 'clever' solutions that require expert-knowledge.

If a bug is noticed which is outside the scope of the change being reviewed, you may include this in a review but state that it could be tackled in a separate pull-request.

The code review may include questions ("why is this cached?") rather than comments.

If you do not have sufficient context to infer what this change is attempting to achieve or fix, you should ask that **first**. Only start reviewing code once you fully understand what the code is attempting to achieve.

## Items to include

- Architectural/design decisions, and whether they fit with the surrounding code or infrastructure
- Functional correctness issues
- Code which is particularly fragile or misleading for future changes
- Maintainability **by humans**. Assume that a future human may be a non-expert reading the code in a hurry.
- Quality and consistency of documentation
- Performance, if performance is an evidenced concern.
- Automated testing

## Items to exclude

- Comments which would be picked up by a linter, formatter or existing automated tests.
- Minor nit-picks and stylistic comments
