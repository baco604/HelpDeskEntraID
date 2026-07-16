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

Heads up: the Microsoft 365 Developer Program sandbox has gotten harder to qualify for
— Microsoft's automated eligibility check now leans toward accounts with a Visual
Studio Professional/Enterprise subscription or Partner Network membership, and a
fresh personal account will often get a "you don't currently qualify" message even
when everything is filled out correctly. Don't worry if that happens to you — there's
a more reliable path that gets you the same result.

**Primary path — Azure free account + Entra ID trial license (most reliable):**

1. Go to `azure.microsoft.com/free` and sign up (identity verification only, no
   charge). This automatically provisions you a Microsoft Entra ID tenant — this is
   separate from the Developer Program sandbox, and everyone doing an Azure signup
   gets one.
2. Sign in at `entra.microsoft.com` with that new account.
3. Go to **Billing → Purchase services** (or **Identity → Licenses → All products →
   Try/Buy**) and activate a **free 30-day trial of Entra ID P1 or P2**. This is a
   standard self-service trial available on any tenant — it's what unlocks
   Conditional Access (P1) and Identity Protection (P2) without needing a developer
   sandbox at all.
4. Set up MFA for your admin account as part of the signup flow.

This is arguably the more realistic path for a help desk field guide anyway — a
support analyst in a real org won't have a Developer Program account, but plenty of
orgs run on P1/P2 trials or standard licensing, so this mirrors what you'd actually
encounter on the job.

**Secondary path — Microsoft 365 Developer Program (if it works for you):**

1. Try linking [Visual Studio Dev Essentials](https://visualstudio.microsoft.com/dev-essentials/)
   first, using the *exact same Microsoft account* you used for the Developer
   Program — this is free and doesn't require a paid VS subscription. It sometimes
   flips the eligibility check by giving Microsoft a "real developer" signal to key off.
2. Go to `developer.microsoft.com/microsoft-365/dev-program`, join, and check your
   dashboard again for a **Set up E5 subscription** option.
3. If you still see "you don't currently qualify," that's a known, common outcome
   right now and not something you can manually override — fall back to the Azure
   free account path above instead of troubleshooting further.

If you do get the Developer Program sandbox working, it gets you a 25-user tenant
with full E5 (P2 included) rather than a 30-day trial, which is nice if you want more
runway — but it's not required for this guide.

> 📸 <img width="2304" height="1438" alt="blurred_screenshot" src="https://github.com/user-attachments/assets/147a4254-837b-4a06-95b7-76b70ccb92c9" />

---

## 2. Creating test users

From the Entra admin center (`entra.microsoft.com`):

1. **Identity → Users → All users → New user → Create new user**
2. Give them a display name, username, and initial password.
3. Repeat for 2–3 test users — it helps to make one a "regular user" and one an
   "admin" so you can see how role assignment changes what they can access.

> 📸 *<img width="2302" height="980" alt="blurred_screenshot2" src="https://github.com/user-attachments/assets/6f67e68b-d6d5-48f6-859a-ee77b923ae3d" />

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

> 📸 <img width="2328" height="1540" alt="blurred_screenshot3" src="https://github.com/user-attachments/assets/af619339-d18d-4629-a61b-49109c4450f3" />

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

> 📸 <img width="1178" height="778" alt="blurred_screenshot4" src="https://github.com/user-attachments/assets/343eb18b-c6ea-425a-bc9a-cda0e0deed41" />

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

> 📸 <img width="1166" height="771" alt="blurred_screenshot5" src="https://github.com/user-attachments/assets/4dbab6e4-4d44-402e-b3f4-2f5cebb17682" />

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

> 📸 <img width="1171" height="777" alt="blurred_screenshot6" src="https://github.com/user-attachments/assets/960dd571-5a24-419f-a5b8-e6c11cd6939d" />

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
- Any Azure free tenant can activate a **self-service 30-day P1/P2 trial** through
  **Billing → Purchase services** — this is the most reliable free path to see the
  full feature set, and it's the same trial mechanism a lot of real orgs use before
  committing to licensing.
- The Microsoft 365 Developer Program sandbox (when you can get it) also includes P2
  via its E5 license, with a longer 25-user runway — but eligibility for new personal
  accounts has become inconsistent, so don't block on it.

---

## Sources / further reading

- Microsoft Learn — Microsoft Entra Conditional Access overview
- Microsoft Learn — Risk policies, Microsoft Entra ID Protection
- Microsoft Learn — Create a Microsoft Entra developer tenant

---

*Sanitize before publishing: remove any real tenant IDs, domain names, or usernames
from screenshots before pushing to GitHub.*
