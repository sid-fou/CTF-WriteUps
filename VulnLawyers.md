# VulnLawyers Web Challenge Writeup

Welcome to my writeup for the VulnLawyers app challenge! Here’s a walkthrough my process, what worked, what didn’t, and how I managed to dig up all those hidden flags.

---

## Part 1: Scanning & Recon

**Target:** `tantalus.ctfio.com`  
**Tools used:** rustscan, dirsearch, caido

- Started off running a port scan with rustscan. Only found HTTP (80) and HTTPS (443) open, both running nginx. Nothing funky or unexpected here.

  <img width="995" height="309" alt="portscan" src="https://github.com/user-attachments/assets/bb2d720e-11b3-45a6-86b3-926ab370dd63" />


- Ran dirsearch with a given wordlist on HackingHub to enumerate directories. Found some usual suspects: `/css`, `/images`, `/js`, plus interesting endpoints: `/login` and `/denied`.

  <img width="997" height="388" alt="dirsearch_on_scope" src="https://github.com/user-attachments/assets/a6352891-4f2b-42d4-993d-18877cf388b6" />


- Decided to check out `/login`. Got redirected to `/denied`, but before the redirect, the body had a legacy login page and surprise! a hidden flag in a bootstrap-styled div. Also found a note saying the portal moved to `/lawyers-only`.
  
  <img width="2349" height="798" alt="flag1" src="https://github.com/user-attachments/assets/231c88c9-53b5-428b-bc98-badd128ea682" />


**What I learned:**  
Don’t ignore the bodies of redirect responses. Sometimes developers leave secrets behind on old pages.

---

## Part 2: The /denied Endpoint & New Login Flow

- Checked `/denied` directly: a 401 Unauthorized page saying my IP was blocked. Nothing useful or sensitive leaked here, just clean, minimal branding.

- Visited `/lawyers-only`, which redirected me to `/lawyers-only-login`. This new login page used a simple form asking for email and password. No flags, just a standard login.

**Endpoints summary:**

| Endpoint              | Result                                     |
|-----------------------|--------------------------------------------|
| `/denied`             | 401, access denied message                 |
| `/lawyers-only`       | 302 redirect to `/lawyers-only-login`      |
| `/lawyers-only-login` | New login form, legit and ready to probe   |

---

## Part 3: Login Attempts & Virtual Host Fuzzing

- First, I tried to log in using various SQL/NoSQL injection payloads and common credentials (`admin:admin123`, etc). Didn’t get far, site handled them gracefully with generic errors, no verbose feedback or hints.

- Switched gears and tried vhost fuzzing: using ffuf with a wordlist to vary the `Host:` header. This found two new subdomains: `api.tantalus.ctfio.com` and `data.tantalus.ctfio.com`. Both returned JSON with app info and, importantly, another flag.
  
  <img width="995" height="442" alt="vhost_fuzzing" src="https://github.com/user-attachments/assets/857bdee8-5f16-42f6-b870-bbeb03e5998d" />

  ---

  <img width="2366" height="513" alt="flag2" src="https://github.com/user-attachments/assets/d2d00c2e-5d1e-4e3b-9688-25deab7a0a3a" />

  
**Takeaway:**  
If the main app is locked down, always check for forgotten subdomains. The devs often leave APIs wide open(obviously kidding xD)

---

## Part 4: API Enumeration, User Dumps & Brute Force

- Ran dirsearch on `data.tantalus.ctfio.com` and found a `/users` endpoint. It served up a JSON list of users and emails, plus (you guessed it) another flag.

  <img width="1004" height="314" alt="dirsearch_on_new-endpoint" src="https://github.com/user-attachments/assets/e9081b3f-9eb0-4622-a54e-1ca0fcb0168e" />

  ---

  <img width="2360" height="632" alt="flag3" src="https://github.com/user-attachments/assets/6163e3bc-ca00-473d-8691-709df96365f1" />


- Took those emails and tried to brute-force logins via `/lawyers-only-login` using the HackingHub provided `password.txt` wordlist.  
Managed to log in as `jaskaran.lowe@vulnlawyers.ctf` with the password `summer`.

- Dashboard loaded for the account, showing site navigation, “Current Cases,” and yet another flag.

  <img width="2366" height="800" alt="flag4" src="https://github.com/user-attachments/assets/0d1a5bf6-22f1-42b0-953a-3143cc3b0fbb" />


**Big lesson:**  
APIs that leak user data are gold for brute force attacks. Weak passwords are still everywhere.

---

## Part 5: IDOR-Breaking Into Accounts via Profile Details

- Checked out the profile section in the logged-in account. The app made an AJAX request to `/lawyers-only-profile-details/4`, which returned a JSON response with the user’s name, email, and password in plaintext.

- Manually changed the profile ID in the URL (`/1`, `/2`, etc.) and pulled the same info for other accounts, including passwords for everyone, and one response had a flag too.

  <img width="2048" height="444" alt="flag5" src="https://github.com/user-attachments/assets/af89691f-f4d6-473e-a016-7a503fda9f13" />

- Used the credentials for “Shayne Cairns” found in this way to log in as them. With this account, I had a new privilege: I could delete a managed case.

---

## Part 6: Privilege Escalation-Case Deletion

- As Shayne Cairns, I clicked the Delete button on a legal case from the dashboard. The app performed the deletion, showed me an alert about there being “no more cases,” and flashed the final flag.

  <img width="2356" height="796" alt="flag6" src="https://github.com/user-attachments/assets/3124d973-60bd-44cc-b503-48c1f0de0c10" />


**Why it matters:**  
Unrestricted destructive actions, no CSRF or privilege checks, and lots of lost trust.

---

## Summary Table: Findings

| What I Did                              | Result / Weakness                   | Flag?    |
|------------------------------------------|-------------------------------------|----------|
| Looked at legacy login page              | Info leak / hidden flag             | 🏁       |
| Vhost fuzzed for dev APIs                | Unprotected dev APIs & flag         | 🏁       |
| Enumerated users via API                 | Info leak (emails), another flag    | 🏁       |
| Brute-forced staff login                 | Weak password, dashboard flag       | 🏁       |
| Exploited IDOR in profile details        | Exposed user creds, another flag    | 🏁       |
| Privilege escalation + case deletion     | No privilege checks, final flag     | 🏁       |

---

## Lessons Learned

- Always check responses, even if a page redirects away.
- Hidden subdomains and APIs often have big security gaps.
- Exposed email/user lists make brute force attacks trivial.
- If you can guess object IDs in API calls, you’ll likely break privilege boundaries.
- Applications should never display or return passwords in plaintext EVER.
- CSRF, privilege checks, and secure API coding: not optional.

---

Thanks for reading!
  
