# Staff are a second principal, and audit does the work a role model would

---

Status: accepted

---

Cherry has two kinds of person: a customer's buyer, and an Aartsen employee. This ADR fixes what an employee principal *is*, how one comes to exist, how it authenticates, and what record it leaves behind.

Decided in [#13](https://github.com/mdegouw/Cherry/issues/13), building on the single staff role from [#3](https://github.com/mdegouw/Cherry/issues/3) and the customer identity model from [#6](https://github.com/mdegouw/Cherry/issues/6).

## One role, and audit instead of a wall

MVP has exactly **one `staff` role**, holding all admin rights across all branches. It covers both job descriptions that have surfaced: the operational one from [#3](https://github.com/mdegouw/Cherry/issues/3) (pickup hours, closures, lead times, band thresholds) and the marketing one from [ADR-0001](./0001-cherry-owned-product-content.md) (photography, copy, categories).

The case for splitting marketing off was blast radius, and it is a real asymmetry — a mistyped closing time moves every order at that branch to the next day, while a wrong photograph is ugly. The reason it does not justify a role is that **a role wall does not address that risk at all**. The person who owns closing times can mistype one, and no permission model stops them. What does address it is a change record with the previous value in it, which is wanted regardless of how many roles exist.

So the split earns its complexity only when someone can be named who must be *unable* to reach commercial config. At Aartsen today nobody can: the same handful of trusted employees do both jobs. A second role would mean a permission check per surface, a role-assignment screen, and a fresh argument about ownership at every new surface — bought against a hypothetical person.

**Revisit when** content editing is done by anyone outside Aartsen's payroll — an agency, a freelancer, an intern. That is a nameable person who should not hold the closing-time editor, and it closes the argument immediately.

## Staff are their own table behind their own guard

A `staff` table and a `staff` auth guard, distinct from `users` and the customer guard. Rejected: one `users` table carrying an `is_staff` flag and a nullable `organisation_id`.

The decisive argument is an invariant. **Every customer-facing query in Cherry is organisation-scoped** — cart, order history, prices, availability at the home branch. If staff live in `users`, `organisation_id` must go nullable, and each of those queries acquires a null case that is *silently wrong* rather than impossible. This repo has twice refused that shape for the same reason: [ADR-0003](./0003-cherry-domain-model.md) dropped address and VAT because a mirrored field with no reader goes silently wrong, and [ADR-0001](./0001-cherry-owned-product-content.md) guards the mid-sync orphaning trap. Separate tables let `users.organisation_id` stay `NOT NULL`.

Two smaller consequences fall out:

- **Foreign keys stay honest.** `order.placed_by_user_id` and `cart_line.touched_by_user_id` become structurally unable to reference an Aartsen employee.
- **The glossary becomes enforceable.** `CONTEXT.md` already defines staff as having no organisation and no debtor number, and tells writers to avoid calling staff a "user". One table would leave that as prose.

The cost normally levelled at a second guard — duplicating an auth scaffold — does not apply. `laravel/blank-vue-starter-kit` ships **no** authentication: no Fortify, no Breeze, only the default `users` / `password_reset_tokens` / `sessions` migration. Customer auth is as greenfield as staff auth, both are hand-built on `Illuminate\Auth`, and a second provider and guard in `config/auth.php` is a few lines.

Staff sign in at **`/beheer`**, unlinked from the customer interface. That is tidiness, not a security measure.

### An Aartsen employee is never a customer

This is a rule at Aartsen, not a rarity — which matters more than it looks. It makes the two principals' email addresses **disjoint**, so the default `password_reset_tokens` table (primary key `email`, one row per address) serves both guards unchanged, with no second table and no re-keying.

Because the rule is load-bearing, it is **enforced rather than assumed**: inviting a staff member rejects an email already present in `users`, and inviting a customer user rejects one already present in `staff`. An unenforced version of this rule fails as a silent password-reset collision, which is precisely the class of bug that is impossible to diagnose from a support call.

## Account lifecycle

**Bootstrap is an artisan command.** `php artisan staff:invite jan@aartsen.nl` creates the row and sends the invite. Not a seeder: a seeded account means a password in git that reaches production. The command is also the recovery door when every staff account is locked out, and it exercises the same code path as the in-app invite rather than a parallel one.

**Thereafter staff invite staff from inside Cherry**, through [#6](https://github.com/mdegouw/Cherry/issues/6)'s invite primitive verbatim — an expiring emailed link on which the person sets their own password. One invite mechanism exists in the codebase; customer-user invitation and staff invitation are two callers of it.

**Any staff member may invite another.** With one all-rights role there is nothing to escalate *to*, so an approval hierarchy would be theatre. The accepted exposure is that one careless manager can mint a second all-rights principal with no-one above them to notice; the audit log records who did it, and the population is roughly ten people.

**Offboarding sets `deactivated_at`; nothing is deleted.** Same pattern as an organisation whose debtor has vanished ([ADR-0003](./0003-cherry-domain-model.md)). A departed employee's name has to survive on their audit entries and on the content they uploaded, or the record stops answering the question it exists for.

## Session, reset and 2FA

[#6](https://github.com/mdegouw/Cherry/issues/6)'s primitives are reused wholesale, with **one deliberate divergence: session length**.

| | Customer user | Staff |
| --- | --- | --- |
| Invite | expiring emailed link, self-set password | identical |
| Password reset | standard Laravel email reset | identical |
| Session | long-lived | **~2h idle timeout** |
| 2FA | none in MVP | none in MVP |

[#6](https://github.com/mdegouw/Cherry/issues/6)'s long session was argued from a specific person: a buyer, on a warehouse floor, at 05:00, who should not be re-authenticating. Staff are not that person. They sit at desks, frequently shared ones, and an abandoned session there can rewrite any branch's closing time or invite a login into any customer's account.

Implementation note: `config/session.php` lifetime is application-wide, so the timeout is a small middleware on the `staff` guard tracking last activity, not a config value. Worth knowing before someone looks for the setting.

**No 2FA in MVP**, consistent with [#6](https://github.com/mdegouw/Cherry/issues/6). The exposure is knowingly asymmetric and should be stated plainly: a compromised customer login exposes one organisation's prices, whereas a compromised staff login exposes **every** customer's prices plus the configuration that moves orders between days. Laravel's built-in authentication is used precisely so this becomes a switch rather than a retrofit, and **staff-only 2FA is the highest-value security addition available post-MVP**.

## The audit log

Section "One role" makes this the load-bearing control rather than a nice-to-have.

**Shape: append-only, with old and new values**, via `spatie/laravel-activitylog`, causer resolved from the `staff` guard. A `updated_by_staff_id` + `updated_at` stamp on each config row was rejected outright — it cannot answer *"what was Venlo's closing time yesterday?"*, and that is the only question anyone ever asks. Answering it needs the value that was replaced.

**Scope: every staff mutation, without exception** — pickup hours, closures, lead times, band thresholds, the `erp_status_code` label lookup, organisation activation, user and staff invitation and deactivation, and product content, images and categories. Logging only the commercially dangerous surfaces reintroduces the distinction that section "One role" declined to enforce, and it invites the argument afresh at every new surface. "Who removed this image" is a real support question too. It is one table.

**A read-only chronological view lives at `/beheer/logboek`** — who, what, when, old → new, newest first. Without it, answering the Venlo question requires a database console, which means in practice it goes unanswered. It is a list; it is not expensive.

Customer actions stay out of the log. Orders are already immutable snapshots ([ADR-0003](./0003-cherry-domain-model.md)) and a cart records `touched_by_user_id` already.

## Impersonation is out of scope

Staff cannot view the shop as a customer sees it, and there is no read-only "view as organisation" either.

The original case for it assumed customer-specific assortment — which [#4](https://github.com/mdegouw/Cherry/issues/4) has since deleted entirely: the assortment is company-wide and identical for everyone. What actually varies per organisation is price, the home branch's availability bands, history, cart, and on-account permission.

Two reasons beyond the shrunken premise:

- **It would decide an open question by accident.** Whether branch staff ever place an order on a customer's behalf is undecided and still fog on the map. A staff session able to act as a customer answers that in the affirmative without anyone choosing it.
- **It breaks order attribution.** `order.placed_by_user_id` references `users`; an order placed by an impersonating employee has no honest value for that column. The same integrity argument that decided the separate table, arriving a second time.

The residual support need — *"the customer says the price is wrong"* — is about price, which is ERP truth the ERP can answer directly, and the identical assortment means a staff member's own catalogue view already shows what the customer sees, modulo branch.

The genuine cost, recorded because it will be felt: a support conversation about a price now means reasoning about two systems and a poll lag rather than looking at what the customer is looking at. Read-only view-as is the fix if that becomes a frequent call — it sidesteps both objections above and can be added without invalidating anything here.

## Consequences

- `users.organisation_id` is `NOT NULL`. No organisation-scoped query needs a null branch, and none should be written defensively as though it does.
- The invite primitive is built once, with three callers: customer-user invitation, staff invitation, and `staff:invite`.
- Cross-table email exclusion is a validation rule on both invite paths, and belongs in the test suite — the failure mode is a silent reset-token collision.
- The staff idle timeout is middleware on the `staff` guard, not `config/session.php`.
- `spatie/laravel-activitylog` joins the dependency list; every staff-editable model is logged, so the trait belongs on a shared base rather than being remembered per model.
- Six staff-facing jobs now exist behind one role: pickup hours and closures and lead times ([#8](https://github.com/mdegouw/Cherry/issues/8)), band thresholds ([#5](https://github.com/mdegouw/Cherry/issues/5)), the `erp_status_code` label lookup ([#10](https://github.com/mdegouw/Cherry/issues/10)), organisation activation and user invitation ([#6](https://github.com/mdegouw/Cherry/issues/6)), staff account management, and product content ([#15](https://github.com/mdegouw/Cherry/issues/15)). They share a shell, a nav and an audit trail; how they are surfaced is not specified here.
- The `/beheer` shell — layout, nav, login page, idle middleware — is specified enough by this ADR to build, and gets no ticket of its own.
