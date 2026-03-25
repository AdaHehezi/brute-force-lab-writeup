# Brute Force Lab Write-up
**CY 5150: Network Security Practices — Northeastern University**
**Tools:** Kali Linux, Hydra v9.6, crunch, curl

---

## Overview

This lab involved two stages of online password attacks against a live target server. The goal was to identify valid credentials using brute-force techniques, with each stage presenting a different authentication mechanism and constraint set.

---

## Stage 1 — HTTP Basic Authentication + Username Harvesting

### Objective
Gain access to a server protected by HTTP Basic Authentication using harvested credentials.

### Approach

Rather than guessing usernames blindly, I started with reconnaissance. I navigated to `https://nuhuskies.com` and explored the site until I found a staff directory at `/staff-directory` containing over 100 `@northeastern.edu` email addresses. I used a Chrome email extractor extension to export them directly into a `user.txt` file on my Kali VM.

For the password list, I downloaded the 10,000 most common passwords and used Hydra to launch the attack:

```bash
hydra -L user.txt -P 10k_most_common.txt -t 16 http-get://35.209.106.120:80/
```

After several hours, Hydra identified valid credentials:
- **Login:** m.rock@northeastern.edu
- **Password:** fishfish

### Key Takeaway
Effective brute-force attacks start with good reconnaissance. A targeted username list dramatically reduces the search space compared to generic wordlists.

---

## Stage 2 — Constrained Character Set + Custom Wordlist Generation

### Objective
Access a `/secret` endpoint protected by HTTP Basic Authentication, using a known username and a constrained password character set.

### Approach

The lab specified:
- **Username:** my Husky ID (`hezij`)
- **Password constraints:** minimum PCI-DSS compliant length (7 characters), using only the characters `BcEiOuX147`

I generated a custom wordlist using `crunch`:

```bash
crunch 7 7 BcEiOuX147 -o wordlist.txt
```

This produced 10,000,000 combinations (~77MB). I then confirmed the target endpoint using `curl` to verify HTTP Basic Auth was active at `/secret` (an earlier mistake targeting the root path `/` cost time — always verify the exact protected resource first).

I launched Hydra with 32 concurrent threads:

```bash
hydra -l hezij -P wordlist.txt -t 32 http-get://35.209.106.120:80/secret
```

After approximately 1 hour 35 minutes, the correct password was found:
- **Password:** BuuBBEE

### Key Takeaway
When the character set is constrained, custom wordlist generation with `crunch` is far more efficient than generic password lists. Verifying the exact target endpoint before launching the attack saves significant time.

---

## Bonus Lab — HTTP POST Form Attack + Redirect Detection

### Objective
Perform a brute-force attack against an HTTP login form and identify valid credentials despite a non-standard server response.

### Approach

I used Hydra's `http-post-form` module with the following parameters:

```bash
hydra -l luke@skywalker.com -P most_security.txt -t 64 35.209.106.120 \
http-post-form "/secret/thelabbonus1.php:uid=^USER^&passwd=^PASS^:S=CONGRATULATIONS" -v
```

### The Tricky Part

Hydra reported **"0 valid passwords found"** — which initially looked like a failure. However, reviewing the verbose output revealed:

```
[VERBOSE] Page redirected to /congratsb1.php?uid=luke@skywalker.com&passwd=passwordone
```

The server returned an **HTTP 302 redirect** upon successful login rather than displaying the success string in the immediate response body. Hydra's success condition (`S=CONGRATULATIONS`) checked the initial response, not the redirect destination — so it missed the match.

Recognizing that a redirect to a congratulations page *is* the success signal, I confirmed the correct credentials:
- **Login:** luke@skywalker.com
- **Password:** passwordone

### Key Takeaway
Automated tools have blind spots. When a tool reports failure, always check the verbose output — the real answer may be hidden in behavior the tool wasn't configured to detect. Understanding *why* a tool behaves a certain way is more valuable than blindly trusting its output.

---

## Reflections

What I found most interesting about this lab wasn't the mechanics of running Hydra — it was the problem-solving required when things didn't work as expected. The Stage 2 mistake of targeting the wrong endpoint, and the Bonus Lab's misleading "0 valid passwords" output, were the most valuable moments: they forced me to slow down, read the actual output carefully, and think about what was really happening at the protocol level.

That mindset — not accepting surface-level results, digging into the underlying behavior — is what separates effective security practitioners from people who just run tools.

---

*Zijuan (Zia) He | MS Cybersecurity, Northeastern University*
*GitHub: github.com/AdaHehezi*
