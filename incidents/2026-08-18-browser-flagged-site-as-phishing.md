# 2026-08-18 — Chrome flagged the n8n site as phishing

## Symptom

After setting up Caddy as a reverse proxy to serve
`n8n.automatewithfemi.com` over HTTPS — with a valid Let's Encrypt
certificate issued, and n8n's published port removed so it was reachable only
over Docker's internal network — I opened the site in Chrome.

Chrome showed a full red interstitial warning: the site was flagged as
phishing, with a recommendation not to proceed because it might steal my
information.

## What I checked, in order

Each step was chosen to eliminate a specific possibility.

**1. The certificate.**
`curl -vI https://n8n.automatewithfemi.com` returned a clean TLS 1.3
handshake, `subject: CN=n8n.automatewithfemi.com`, issued by Let's Encrypt,
valid to 16 November 2026.

I checked this first because a bad certificate is the most common cause of a
browser refusing a site. But the reasoning cut both ways: **had it been a
certificate problem, Chrome would have said so specifically.** A phishing
warning is a different category of message entirely, so this test was as much
about confirming the category as confirming the certificate.

→ **Certificate ruled out.**

**2. Google's own Safe Browsing site status tool.**
Rather than infer from the browser warning, I queried Google's transparency
report directly for both names.

- `n8n.automatewithfemi.com` → **"No available data"** — not flagged
- `automatewithfemi.com` (root) → **flagged**, category: *tries to trick
  visitors into sharing personal info*

→ **My subdomain is not flagged by Google.** The warning was not a
Google-wide block on the thing I had built.

**3. The domain's history.**
Checked the Wayback Machine for archived snapshots of
`automatewithfemi.com` — **no history recorded.** So I could not date the
flag or see what content produced it.

**4. A second browser.**
Safari on my phone loaded the n8n login page immediately, with no warning.

→ **Not universal.**

**5. Incognito, same browser, same machine.**
Loaded the login page with no warning.

→ **Something in my normal Chrome profile**, since Safe Browsing behaves
identically in incognito.

**6. A different Chrome profile, on a different device.**
On my phone's Chrome app: my usual profile showed the red warning; a separate
profile with different account data loaded the site fine. Same result with a
second profile on the laptop.

→ **Decisive.** A phone and a laptop are separate machines. The only thing
they share is the Google account. **The warning follows the account, not the
device, the network, or the site.**

**7. Confirmed the port change had actually taken effect.**
`dig` and the earlier `curl` confirmed the domain resolved to the server and
that traffic was being served by Caddy on 443. n8n's own port was no longer
published on the host, so nothing was reachable directly.

→ **The reverse proxy was working exactly as intended.**

## Cause

The warning came from my personal Chrome profile, not from Google's Safe
Browsing database and not from anything on the server.

**Most likely an extension** — the profile has several wallet and browser
extensions installed — but I have not confirmed which. That is a separate,
minor issue on my own machine.

## Fix

Nothing on the server needed changing. The infrastructure was correct
throughout.

Practical resolution: **use a dedicated Chrome profile for this project**,
separate from personal browsing. That removes extension interference, keeps
project credentials separate from personal ones, and means what I see is what
a client sees — which matters, because I cannot demo a system I cannot load.

## ⚠️ Unresolved

**The root domain `automatewithfemi.com` is genuinely flagged by Google, and I
do not know why.**

What is established:

- It **does not resolve** to an IP (`dig` returns nothing) — the only records
  on it are a URL redirect and a `www` CNAME to the registrar's parking page
- It has **never pointed at my server**
- **No Wayback history exists**, so the flagged content cannot be dated or
  inspected

The most plausible explanation is content from a previous registration, but I
cannot demonstrate that.

**Deferred deliberately.** A Search Console review request needs something at
the root for Google to look at, and there is currently nothing there. This gets
resolved when the public site is built — and **before any outreach email is
sent from this domain**, since email reputation systems use similar data and a
flagged domain would quietly land in spam folders.

## Lessons

**Check a domain's Safe Browsing status and history before buying it.** I did
not, and inherited a flag on the root. This now belongs in my pre-purchase
checklist — and it is something I would do on a client's behalf before
starting a project from scratch.

**Query the authoritative source rather than inferring from a symptom.** The
browser warning suggested a global block. Google's own tool said the opposite.
One lookup separated "the world thinks this is dangerous" from "my browser
thinks this is dangerous" — two very different problems.

**Vary one thing at a time to localise a fault.** Different browser, then
incognito, then different profile, then different device. Each step ruled out a
layer. **The step that settled it was using the same profile on a different
machine** — that isolated the account as the only shared variable.

**Match the test to the category of error.** A certificate error and a phishing
warning are different messages with different causes. Reading which one you
actually got narrows the search before you run a single command.

**Say what is unresolved rather than inventing a cause.** The root domain flag
has no established explanation. Recording what was ruled out is more useful
than a guess presented as a finding.
