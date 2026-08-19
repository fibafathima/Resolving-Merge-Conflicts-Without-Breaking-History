# Conflict Recovery Submission

## What Was Broken

The repository contains a recovery history that is more complicated than the current branch names suggest. The original feature work diverged from the shared base at `677ef5b`: `fea73ef` implemented a 10% discount on `feature/discounts`, while `3f1514d` implemented a 5% tax change on the other side of the checkout function. The setup script records that these branches were merged without a deliberate integration plan, and that the conflict was committed with unresolved markers in the intermediate history.

The rounding fix was introduced in `fb7f6b5`, then carried into the production-preparation commit `0fbf8d4`. The integration commits `6f3eb9b` and `1c47c61` combine overlapping checkout changes, which makes the history difficult to audit and creates a release risk if one of the business rules is silently dropped. The current `main` tip, `ba3ac47`, points at the supplied recovery documentation rather than exposing separate feature branch names, so the original feature refs are no longer available locally. That missing branch metadata is itself a traceability problem.

## Recovery Actions Taken

I inspected `git log --oneline --all --graph`, `git branch -a`, and `git status` before changing files. I created the required `recovery/main-fix` branch from the current `main`, leaving shared `main` untouched. The repository already contains the stable reconstruction through `0d42bc3`, so I did not reset or force-push `main`; rewriting a shared branch would make collaborators with the old `ba3ac47` history diverge.

The stable sequence preserves the rounding fix, applies the 10% discount, and then applies the 5% tax. I validated it with `node test.js`, which returns `$9.45` for a `$10.00` item. A `git revert` was not needed because the current `main` tip does not contain an additional bad application commit to undo, and a `git cherry-pick` was not needed because the required logic is already present in the reachable stable ancestry. If a bad commit were later identified on shared `main`, `git revert` would be the appropriate non-destructive tool; `git reset --hard` would rewrite history and is therefore inappropriate here.

## Conflict Resolution Decisions

The conflict described by `setup_mess.sh` is in `checkout.js`, inside `checkout(items)`. The discount side calculates the item total and applies a 10% reduction, while the tax side calculates the same base total and applies a 5% increase. The rounding side also needs to remain in the final function because it prevents floating-point output from escaping with excess precision.

The correct resolution is a combination: calculate the item sum once, apply the discount first, apply tax to the discounted amount, and round the final result to two decimal places. This preserves both valid business requirements and keeps the rounding fix. The resolved file contains no conflict markers, and the validation case confirms that `$10.00` becomes `$9.00` after discount and `$9.45` after tax.

## Repository State After Recovery

`recovery/main-fix` is the required working branch, and `main` was not rewritten. The checkout implementation is deployable for the supplied validation case, the conflict markers are absent from `checkout.js`, and `node test.js` passes. The remote `recovery/stable-release` branch remains open because it contains the stable reconstruction and its investigation artifacts; it is safe to leave open as historical reference while the pull request is reviewed. The original feature branch refs are not present in this clone, so their names and exact original tips cannot be used for new merges without reconstructing them from the setup script.

## Video Talking Points

In the recording, I will show the commit graph and point out the diverged discount and tax commits, the rounding fix, and the integration commits. I will then show the creation of `recovery/main-fix`, the resolved `checkout.js`, and the passing `node test.js` command. I will explain that short-lived branches synced with `main` regularly, together with protected `main` and required validation before merging, would have prevented this failure.