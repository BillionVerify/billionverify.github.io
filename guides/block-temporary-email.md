# How to Block Temporary Email on a Signup Form

Canonical page: https://billionverify.github.io/guides/block-temporary-email/

To block temporary email, check each address when the form is submitted and compare its domain with a list of known disposable providers. No single signal catches every address, so most checkers combine several. Figures here were read in October 2026 from BillionVerify's public repository, public DNS lookups and vendor documentation.

**Key facts (checked October 2026):**

- BillionVerify's open list held 220,601 domains on 27 October 2026.
- The list merges nine community sources and updates hourly.
- The list includes privacy relays, so rejecting every entry also rejects real people.
- Domain checks run in memory with no network call when the list is embedded.

## What signals can a checker use?

The domain list is the main method. The checker takes everything after the @ sign and looks it up. If the list is embedded in the program, the lookup needs no network call.

Mail server records add a second signal. A temporary service often points many domains at one mail host. A public DNS lookup in October 2026 showed guerrillamail.com and sharklasers.com sharing the host mail.guerrillamail.com. A checker that knows the host can treat other domains on it as suspects.

| Signal | What it looks at | Weakness |
| --- | --- | --- |
| Domain list | The domain after the @ sign | Misses domains no list has recorded |
| Mail host | Which server receives the mail | Needs a known disposable host |
| Domain age | How recently the domain was registered, from records such as RDAP | A new business looks the same |
| Alias or relay | Whether mail only forwards elsewhere | Many people use relays for privacy |
| Signup behavior | IP address, timing and volume | Only the site itself sees it |

## Why does detection keep changing?

A page that promises undetectable temp mail describes one result, from one checker, on one day. BillionVerify's open list pulls from nine community sources every hour, so a domain that passes today can fail tomorrow. One service can also run many domains. Guerrilla Mail listed 11 in its menu, and all 11 were already in the list.

### Block or flag

A hard block is not always right. Many teams explain the rule and ask for a lasting address. Others accept the signup and tag it as low trust.

### Avoid false positives

Watch for false positives. The BillionVerify list includes privacy relays such as relay.firefox.com, simplelogin.com and duck.com. Rejecting every entry also rejects people who use relays for privacy, and a new company domain can trip an age check. Do not block Gmail or Outlook because they are free. Keep your own exceptions list. To see how a privacy relay is flagged before you set a rule, you can [verify an email address for free](https://billionverify.com/email-verifier) and read the flags it returns.

## Which tools can run the check?

BillionVerify publishes an [open-source library on GitHub](https://github.com/BillionVerify/disposable) with Go and Rust packages and a Go command-line tool. According to its README, it embeds domains merged from nine community lists, and the list held 220,601 domains on 27 October 2026. Rule order is exceptions first, then an exact match, then wildcard suffixes. The README also documents a [hosted check](/docs/api/disposable-check/) that does not use credits and performs only the domain lookup.

For one address, the free disposable email checker answers a single question: is the domain disposable? To find throwaway addresses already in your database, run a bulk verification and segment them before the next campaign. Before you write any code, you can paste a suspect address into [BillionVerify](https://billionverify.com/email-checker) and see the disposable flag next to the live SMTP result.

BillionVerify does not provide temporary inboxes. A BillionVerify account gets 20 free credits on each day you sign in, up to 600 a month.

## Original article

- [Temp Mail Not Detected? How Sites Detect Temporary Email](https://billionverify.com/blog/disposable-email-detection)
