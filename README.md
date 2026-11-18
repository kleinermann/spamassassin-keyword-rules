# spamassassin-keyword-rules

Manage custom SpamAssassin rules with plain word lists instead of hand-written regular expressions.

Put spam words and phrases into text files, one per line. `sa-keyword-rules` turns them into a valid `local_rules.cf`, checks it with `spamassassin --lint`, installs it and restarts spamd. If the check fails, the previous rules are restored, so a typo can never break your spam filtering.

It also manages trusted and blocked senders (`welcomelist_auth`, `welcomelist_from`, `blocklist_from`) from simple lists.

Requires Python 3 (standard library only) and SpamAssassin. Tested with SpamAssassin 4.x on Debian/Ubuntu (e.g. HestiaCP servers).

> This is an independent project and is not affiliated with or endorsed by the Apache SpamAssassin project.

---

## How it works

1. Reads the word lists in `lists/`.
2. Converts each phrase into a regular expression.
3. Fills them into `local_rules.template`, which holds the rule layout and scores.
4. Writes `/etc/spamassassin/local_rules.cf`.
5. Runs `spamassassin --lint`.
   - **OK:** keeps the new file and restarts spamd.
   - **Error:** restores the previous file, shows the error and changes nothing.

---

## Installation

```bash
cd /root
git clone https://github.com/kleinermann/spamassassin-keyword-rules.git
ln -s /root/spamassassin-keyword-rules/sa-keyword-rules /usr/local/bin/sa-keyword-rules
```

The script finds its lists and template through the symlink, so you can keep the folder anywhere.

If you already have a hand-written `local_rules.cf` (or a separate senders file), move it out of `/etc/spamassassin/` first. SpamAssassin loads every `.cf` file in that folder, so leaving it there would load the rules twice.

```bash
mv /etc/spamassassin/local_rules.cf /root/local_rules.cf.old
```

All lists ship empty, with commented-out examples. Fill in your own words and senders (at least `senders-trusted.txt`), then:

```bash
sa-keyword-rules --dry   # build and lint only, changes nothing
sa-keyword-rules         # build, lint, install, restart spamd
```

> **Don't edit `/etc/spamassassin/local_rules.cf` directly.** It is overwritten on every run.

---

## Usage

Add a phrase and rebuild:

```bash
echo "guaranteed first page" >> /root/spamassassin-keyword-rules/lists/body-business-pitch.txt
sa-keyword-rules
```

| Option | |
|---|---|
| *(none)* | build, lint, install, restart spamd |
| `--dry` | build and lint only, change nothing |
| `-h`, `--help` | show help |

| Environment variable | Default |
|---|---|
| `SPAM_RULES_DIR` | folder of the script, else `/root/spamassassin-keyword-rules` |
| `SPAM_RULES_OUT` | `/etc/spamassassin/local_rules.cf` |

---

## The lists

The list files are grouped by what they check, and the filename prefix tells you which:

| Prefix | Checks |
|---|---|
| `senders-` | who the email is from |
| `subject-` | the subject line |
| `body-` | the message text and links |

### Senders

| File | Purpose |
|---|---|
| `senders-trusted.txt` | Trusted if SPF/DKIM confirms the sender (`welcomelist_auth`) |
| `senders-trusted-from.txt` | Trusted on the From address alone (`welcomelist_from`). Forgeable, so keep it short |
| `senders-blocked.txt` | Always spam (`blocklist_from`) |

### Subject

| File | Purpose | Score (default) |
|---|---|---|
| `subject-spam.txt` | Spam subject lines | 3.5 |

### Body

| File | Purpose | Score (default) |
|---|---|---|
| `body-spam-only.txt` | Spam-only names, tools, brands | 5.0 |
| `body-link-domains.txt` | Domains in links (URLs accepted) | 5.0 |
| `body-classic-spam.txt` | Classic spam phrases | 3.0, 2+ hits 5.0 |
| `body-adult.txt` | Adult content | 3.0, 2+ hits 5.0 |
| `body-business-pitch.txt` | Cold sales pitches to you as a business (SEO, web design, leads), including their "reply No" opt-out lines | 1 hit 1.0, 2+ 3.0, 4+ 5.0 |
| `body-product-promo.txt` | Consumer product ads (coupons, free trials, gadgets). Weak signals | 1.5 for 2+ hits |
| `body-phishing.txt` | Phishing lures | 3.0 |

Any business pitch or product promotion that also contains an "unsubscribe" line gets +1.0. Unsubscribe on its own scores nothing.

### Foreign-script rules

The template also includes fixed rules (not list-driven) that score text in writing systems commonly used for spam:

| Prefix | Script |
|---|---|
| `CN_` | Chinese |
| `RU_` | Cyrillic (Russian, Ukrainian, Bulgarian, Serbian…) |
| `JP_` | Japanese (kana; kanji also trigger `CN_`) |
| `KO_` | Korean |
| `AR_` | Arabic script (Arabic, Persian, Urdu) |
| `TH_` | Thai |

Each script is detected by its charset declaration, an encoded subject, characters in the subject, and characters in the body. These score 3.0–4.0 each, so a typical message adds up to 7–11 points. If you receive legitimate mail in any of these languages, lower or delete that prefix's lines in `local_rules.template`.

Spam in Latin-script languages (Spanish, German, Vietnamese…) can't be detected by script. For that, the template has an optional, commented-out `ok_languages` block that uses SpamAssassin's TextCat plugin (enable it in `v310.pre`).

Scores are set in `local_rules.template`, not in the lists.

### Writing phrases

| In the list | Matches |
|---|---|
| `free money` | any capitalisation, any spacing or line breaks |
| `if you're not interested` | straight `'` and curly `’` apostrophes |
| `reply "No"` | quotes optional: `reply No`, `reply "No"`, `reply “No”` |
| `white-label` | `white-label`, `white label`, `whitelabel` |
| `no obligation.` | trailing punctuation is ignored |
| `# note` | comment |
| `re:\bbacklinks?\b` | raw Perl regex, for advanced cases |

Word boundaries are added automatically, so `reply no` does not match "reply now". Duplicate lines are skipped.

### Writing senders

| In the list | Becomes |
|---|---|
| `example.com` | `*@example.com` |
| `@example.com` | `*@example.com` |
| `jane@example.com` | that address only |
| `*@*.xyz` | kept as written |

Several entries on one line, separated by spaces or commas, also work. Invalid entries stop the build with an error.

---

## Template placeholders

`local_rules.template` is a normal SpamAssassin config file with placeholders:

| Placeholder | Replaced with |
|---|---|
| `{{list:file.txt}}` | phrases joined with `\|`, for use inside `/(?:...)/i` |
| `{{domains:file.txt}}` | domains joined with `\|` (for `uri` rules) |
| `{{senders:DIRECTIVE:file.txt}}` | one `DIRECTIVE  pattern` line per sender |

Allowed directives: `welcomelist_auth`, `welcomelist_from`, `blocklist_from`, plus the pre-4.0 names `whitelist_auth`, `whitelist_from`, `blacklist_from`. On SpamAssassin older than 4.0, switch the template to the old names.

To add a list of your own, create `lists/mylist.txt` and reference it in the template, for example:

```
body     LOCAL_MYLIST  /(?:{{list:mylist.txt}})/i
score    LOCAL_MYLIST  2.0
describe LOCAL_MYLIST  My custom list
```

An empty list becomes a pattern that never matches, so the rule stays valid.

---

## Notes

- `normalize_charset 1` is set in the template. Matching works on UTF-8 bytes, which is what the curly-quote and non-Latin rules rely on.
- To test which rules a message triggers: `spamassassin -t < message.eml | grep LOCAL_`
- Start with moderate scores and watch your Spam folder before raising them.

## License

MIT © 2026 Stefan Ehrenfeld ([kleinermann.com.au](https://kleinermann.com.au)). See [LICENSE](LICENSE).
