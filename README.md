# Entra ID for Beginners: A Help Desk Field Guide

*Infrastructure & Tooling — Phase 2, Project 8*
*Author: Baco Cisse | github.com/baco604*

## Why this guide exists

Entra ID (formerly Azure Active Directory) is the identity layer sitting behind almost
every sign-in in a modern enterprise — and it's the tool a help desk analyst touches
more than almost any other. Password resets, account lockouts, "I'm not getting my MFA
prompt," access requests, new-hire provisioning — nearly all of it runs through here.
We don't have a live Entra ID tenant in the range to review yet, so this guide walks
through spinning up a free tenant, exploring the settings a help desk analyst actually
needs day-to-day, and where to look when a ticket says "I can't sign in." Written for
someone who has never opened the Entra admin center before.

---

## 1. Setting up a free tenant

There are two ways to get hands-on with Entra ID at no cost, and they get you very
different levels of access:

| Path | What you get | Best for |
|---|---|---|
| **Azure free account → Entra ID Free** | Automatically included with any Azure/Microsoft 365 signup. Core directory, basic SSO for up to ~10 pre-integrated apps, user/group management. No credit card required for the Entra portion. | Learning the basic directory model and navigation |
| **Microsoft 365 Developer Program** | A 25-user sandbox tenant with **E5 licensing**, which includes Entra ID P2 — so Conditional Access, Identity Protection, and risk-based sign-in detection are all unlocked. | Actually seeing the security features a SOC would use |

**Recommendation:** start with the Developer Program tenant if you qualify — the free
tier alone won't show you Conditional Access or risky sign-ins, since those require
P1 and P2 respectively, and P2-only capabilities (risk-based access policies,
Identity Protection, Privileged Identity Management) don't exist at all in the free
tier. As a help desk analyst you may never configure these policies yourself, but
you'll be the one fielding the ticket when one of them blocks a user, so it helps to
have actually seen them from the inside.

**Steps (Developer Program path):**
1. Go to `developer.microsoft.com/microsoft-365/dev-program` and join with a Microsoft account.
2. Choose a region, company name, and accept terms.
3. Select **Set up subscription** — this provisions your tenant and creates your first
   Global Admin account.
4. Set up MFA for that admin account (required as part of tenant creation).
5. Optionally add the sample data pack (test users, groups, mailboxes) — handy for
   having realistic-looking accounts to poke around with.

> 📸 *[Insert screenshot: tenant creation confirmation screen]*

---

## 2. Creating test users

From the Entra admin center (`entra.microsoft.com`):

1. **Identity → Users → All users → New user → Create new user**
2. Give them a display name, username, and initial password.
3. Repeat for 2–3 test users — it helps to make one a "regular user" and one an
   "admin" so you can see how role assignment changes what they can access.

> 📸 *[Insert screenshot: user list after creating test accounts]*

---

## 3. Roles and permissions

Entra uses **role-based access control (RBAC)**. Every admin capability — creating
users, managing Conditional Access, resetting passwords — is gated behind a specific
built-in role rather than a single "admin/not admin" switch. A few roles worth knowing:

- **Global Administrator** — full tenant control. Should have as few holders as possible.
- **Conditional Access Administrator** — the least-privileged role that can actually
  create or edit Conditional Access policies.
- **User Administrator** — can manage users and groups, but not security policy. This
  is the role most help desk teams are actually assigned, since it covers password
  resets and account unlocks without granting security-policy control.

Go to **Identity → Roles & administrators** to see the full built-in role list and
who's assigned to each. Knowing this distinction matters on the job: if a ticket
needs something outside your role (like a Conditional Access exception), you'll know
immediately that it needs to be escalated rather than spending time hunting for a
button that isn't there.

> 📸 *[Insert screenshot: Roles & administrators list]*

---

## 4. MFA — and the ticket you'll get about it

MFA is the single biggest lever against credential-based attacks, and "I'm locked out
of MFA" or "I got a new phone and lost my authenticator" is one of the most common
help desk tickets there is. In the admin center:

1. **Protection → Authentication methods → Policies** — this is where you see which
   methods are allowed (Microsoft Authenticator, SMS, FIDO2/passkey).
2. **Identity → Users → [pick a user] → Authentication methods** — this is the page
   you'll actually live in as help desk. It shows what methods that specific user has
   registered, and lets you reset or remove a method (e.g., delete their old phone's
   Authenticator registration so they can re-register on a new device).
3. To *require* MFA org-wide, it's enforced through a Conditional Access policy (see
   below), which needs P1 or above — not a per-user toggle in the free tier.

Knowing where to look here is the difference between a two-minute ticket resolution
and escalating something you could have handled yourself.

> 📸 *[Insert screenshot: Authentication methods policy page]*

---

## 5. Conditional Access policies

You likely won't be the one writing these, but you need to recognize them — because
"why can't I sign in from home" or "why does it keep asking me for MFA" is almost
always a Conditional Access policy doing exactly what it was configured to do.
Conditional Access policies are if-then statements: *if* a user tries to sign in under
these conditions, *then* they must do this before being granted access.

Requires **Entra ID P1** at minimum (included in the Developer Program E5 tenant).

**A simple starter policy — require MFA for all users:**
1. **Protection → Conditional Access → Create new policy**
2. **Users** → Include: All users. Exclude: your break-glass/emergency admin account
   (always keep at least one account outside every blanket policy, so you can't lock
   yourself out).
3. **Target resources** → Include: All resources.
4. **Grant** → Require multifactor authentication.
5. Save in **Report-only** mode first, confirm it behaves as expected, then switch it On.

**A risk-based policy — require MFA on medium/high sign-in risk (needs P2):**
Same flow, but under **Conditions → Sign-in risk**, set it to Yes and choose
Medium and High. This is powered by Identity Protection, which scores each sign-in
using signals like impossible travel, anonymous IP, and leaked credentials.

> 📸 *[Insert screenshot: Conditional Access policy creation screen]*

---

## 6. Where risky sign-in activity shows up

If you're on a P2-enabled tenant: **Protection → Identity Protection → Sign-ins** (or
**Risky sign-ins**) shows every sign-in Entra scored as risky, along with the specific
detection that fired (atypical travel, anonymous IP address, malware-linked IP,
leaked credentials, etc.) and the risk level assigned.

This matters for a very practical help desk reason: if a user calls in saying they're
suddenly blocked or being asked to change their password out of nowhere, this is the
first place to check *why* — did Entra flag something real, or is it a false positive
(a legitimate VPN, a new device) that needs to be cleared? Knowing which one it is
determines whether you reset the user and move on, or escalate to security.

> 📸 *[Insert screenshot: Risky sign-ins report]*

---

## 7. What a help desk analyst should know to look for

You're not the one setting security policy, but recognizing these patterns tells you
when a ticket is routine versus when it needs to go up the chain:

- **Legacy authentication still enabled** — if you notice a user's account allows
  sign-in methods that skip MFA entirely, that's worth flagging up, not just resolving
  the immediate ticket.
- **Repeated lockouts on the same account** — one lockout is a forgotten password;
  three in a day, especially from different locations, is worth mentioning to security
  before you just keep resetting it.
- **A user is blocked by Conditional Access they've never hit before** — check
  Sign-in logs first (see below) before assuming it's a simple password issue.
- **Risky sign-ins left sitting unresolved** — if Identity Protection shows a flagged
  sign-in that nobody's acted on, that's a real escalation, not a "reset and move on."
- **Sign-in logs for unfamiliar locations or apps** — **Identity → Monitoring →
  Sign-in logs** is where you go first when a user says "I didn't try to log in from
  [wherever]."

---

## 8. How this fits into the bigger security picture

Entra ID doesn't sit in isolation — in a mature environment, its sign-in logs, audit
logs, and Identity Protection risk events feed into a SIEM like Microsoft Sentinel via
the **Microsoft Entra ID data connector**, where the security team can correlate a
flagged sign-in against other activity (like something suspicious happening on that
same user's device). You won't be building those correlations as help desk, but
understanding that your ticket queue and the SOC's alert queue are pulling from the
same underlying data is useful context — it's part of why accurate, complete ticket
notes matter: what you write down about a "resolved" lockout might be exactly what a
security analyst needs later if it turns out to be part of something bigger.

---

## Notes on licensing (for your own reference)

- **Entra ID Free** — directory, basic SSO, no Conditional Access, no Identity Protection.
- **Entra ID P1 (~$6/user/mo)** — adds Conditional Access, self-service password reset
  with on-prem writeback, group-based licensing.
- **Entra ID P2 (~$9/user/mo)** — adds Identity Protection (risk-based sign-in/user
  detection) and Privileged Identity Management (PIM).
- The Microsoft 365 Developer Program sandbox gives you E5, which includes P2 — so
  it's the only free path that shows the full security feature set.

---

## Sources / further reading

- Microsoft Learn — Microsoft Entra Conditional Access overview
- Microsoft Learn — Risk policies, Microsoft Entra ID Protection
- Microsoft Learn — Create a Microsoft Entra developer tenant

---

*Sanitize before publishing: remove any real tenant IDs, domain names, or usernames
from screenshots before pushing to GitHub.*
