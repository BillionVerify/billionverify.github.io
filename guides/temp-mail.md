# Temp Mail in Plain Words: A Short Guide

Canonical page: https://billionverify.github.io/guides/temp-mail/

Temp mail is a free inbox that lives for a short time. You open a website and get a working address without an account. New messages appear on the same page. When the timer ends, or you press delete, the inbox is gone.

**Key facts (checked October 2026):**

- Guerrilla Mail says it keeps incoming mail for 1 hour.
- YOPmail says it keeps messages for 8 days and its inboxes have no password.
- 10minutemail.com runs a ten-minute countdown with a button that adds time.
- BillionVerify does not run a temp mail service.

## How does temp mail work?

The service runs mail servers on its own domains, and they accept mail for almost any name. So a [temp email address](https://billionverify.com/blog/what-is-temp-mail) is only a name, and nothing is created until the first message arrives. YOPmail says all of its inboxes already exist.

### Who can read a temp mail inbox?

Most free inboxes have no password. Anyone who types the same name sees the same mail. [Guerrilla Mail](https://www.guerrillamail.com/about) warns that "anyone who knows your Inbox ID may access your inbox." [Mailinator](https://www.mailinator.com/docs/) puts it more plainly: "All emails are public."

### How long does temp mail last?

Each service sets its own retention. The table shows what each one says on its own pages, read in October 2026. Our [10 minute mail comparison](https://billionverify.com/blog/10-minute-mail-alternatives) covers nine services.

| Service | Mail is kept for | Sending |
| --- | --- | --- |
| 10minutemail.com | Ten minutes, extendable | Not stated |
| Guerrilla Mail | 1 hour | Compose tab |
| YOPmail | 8 days | Replies only, no outside mail |
| Mailinator (public) | Several hours | Receive-only |

## What is temp mail used for?

The oldest use is keeping spam out of a real inbox. Developers also use it to test sign-ups and password resets without writing to real people. It works for one message and is a poor fit for anything you need to reach again.

- Other people can read a public inbox if the name is easy to guess.
- Mail disappears when the limit ends, so a later reset link may never arrive.
- Account recovery fails when the signup inbox no longer exists.
- It hides your real mailbox only. Your IP address, browser and payment details stay visible.

For a long-term account, a forwarding alias is better than a [burner email](https://billionverify.com/glossary/burner-email). It sends mail to your real mailbox, and you can switch it off later.

## Why do websites reject temp mail?

A site that asks for an email address wants to reach you later. A temporary address breaks that plan within hours. Sites also see fake signups, repeat free trials and receipts that go nowhere.

So many forms compare the domain after the @ sign with a [list of known temporary services](https://billionverify.com/blog/disposable-email-detection). If the domain is listed, the form asks for another address. When the account matters, use a mailbox you own.

### Tools for site owners

BillionVerify builds email verification tools for people who run forms.

The free [disposable email checker](https://billionverify.com/disposable-email-detection) takes one address and answers one question: is the domain disposable? For code, BillionVerify publishes an [open-source library on GitHub](https://github.com/BillionVerify/disposable) with Go and Rust packages. Its README says the domains are merged from nine community lists and updated hourly. The list held 220,601 domains on 27 October 2026.

Do not block Gmail or Outlook only because they are free. They are real mailboxes.

A [BillionVerify account](https://billionverify.com/auth/sign-up) gets 20 free credits on each day you use your account, up to 600 a month.

## Read the full guides on BillionVerify

- [What Is Temp Mail? How Temporary Email Works](https://billionverify.com/blog/what-is-temp-mail)
- [10 Minute Mail Alternatives: 9 Temp Mail Services Compared](https://billionverify.com/blog/10-minute-mail-alternatives)
- [Temp Mail Not Detected? How Sites Detect Temporary Email](https://billionverify.com/blog/disposable-email-detection)
- [Free and Open-Source Disposable Email Detection](https://billionverify.com/blog/free-and-open-source-disposable-email-detection)
