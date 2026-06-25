---
name: bruteforce-login
description: >
  Write and run Python scripts to brute-force website login passwords. Use this skill
  whenever the user mentions brute-forcing a login, cracking passwords, testing multiple
  credentials against a login page, running a credential-stuffing attack, batch-testing
  accounts, 爆破登录, 密码爆破, 撞库, 批量登录测试, or similar. This skill handles the full
  workflow: inspecting the login page via chrome-devtools, identifying form fields and
  CAPTCHA type, generating a tailored script with progress tracking and resumability,
  and running it to verify it works.
compatibility: Requires chrome-devtools MCP, Python 3.8+, playwright, DrissionPage, ddddocr
---

# Login Brute-Force Script Generator

Generate a tailored Python brute-force login script based on analysis of the target
website's login page. The script must handle CAPTCHA (image or slider), track progress
so it can resume after interruption, and write a readable report of results.

## Workflow

Follow these steps in order:

### 1. Gather requirements

Ask the user for:

- **Login URL** (required)
- **Username(s)**: a file path (one per line), a single value, or a range (e.g., "0000-9999")
- **Password(s)**: same three options — file, single value, or range
- Any additional context (known error messages, VPN/proxy requirements, etc.)

If the user hasn't specified all of these, ask before proceeding.

### 2. Inspect the login page

Use chrome-devtools to open the URL and analyze the page thoroughly:

**Basic element identification:**
```
- Navigate to the login URL
- Take a snapshot to identify:
  - Username/account input field (note the selector: id, name, class, CSS path)
  - Password input field
  - Submit/login button
  - Checkboxes (e.g., "agree to terms")
  - CAPTCHA element — is it an <img> tag (image CAPTCHA) or a slider component?
  - Any other elements that affect login flow
- Check the Network tab to observe the login request:
  - What endpoint does the form POST to?
  - What's the request/response format?
  - What does a success vs. failure response look like?
```

**JavaScript analysis (important):**
- Open the Network tab and filter for `.js` files. Read the login-related JavaScript
  files to understand the actual login logic — don't rely solely on visible DOM elements.
- Check for:
  - **Hidden or tabbed login forms** — some sites have multiple login modes (phone vs.
    username/password) where one is hidden via `display:none`. The real login logic may
    only be active for one mode, requiring DOM manipulation or function overrides.
  - **Client-side encryption** — look for functions like `getEncrypt()`, `encrypt()`,
    `btoa()`, `RSA.encrypt()`, or custom encoding logic that transforms credentials
    before sending. If found, the script must replicate or call this logic.
  - **The actual submit handler** — sometimes a `<button>` has no `<form>` parent and
    instead triggers a JavaScript function (e.g., `check()`, `doLogin()`). Find that
    function and trace what it does (AJAX call, form submit, validation).
  - **Vue/React component internals** — for SPAs, identify the component tree via
    `__vue__` or React DevTools. CAPTCHA solutions (especially sliders) often require
    calling component methods directly (e.g., `onVerifySuccess(offsetList, tag)`).

**If the site is inaccessible** (ERR_EMPTY_RESPONSE, connection refused, geo-blocked):
- Ask the user if they have a VPN/proxy and need to connect through it.
- If the user has existing reference scripts that worked against this site, read them
  for verified selectors and API endpoints. Use those as the source of truth.
- Still attempt to access the page via alternative means (different user agent,
  different network) before falling back to reference material.

**If the page structure differs from the user's description or reference scripts:**
- Trust what you observe in chrome-devtools over the description. Pages change over time.
- Report any discrepancies to the user (e.g., "the #submitBtn from the reference script
  no longer exists; the site now uses #loginId instead").

Keep detailed notes on selectors, URLs, encryption logic, and response patterns — these
will be embedded in the generated script.

### 3. Classify CAPTCHA type

| Type | Indicator | Strategy |
|------|-----------|----------|
| None | No CAPTCHA element found | Skip CAPTCHA logic entirely |
| Image CAPTCHA | `<img>` with src containing "captcha"/"code"/"check" | Use ddddocr OCR; periodically re-fetch if wrong |
| Slider CAPTCHA | `<div>` with "slider"/"verify"/"block" class, draggable element | Generate mouse trajectory, trigger onVerifySuccess or simulate drag |
| reCAPTCHA/hCaptcha | iframe from google.com/hcaptcha.com | Warn user: cannot be reliably automated; suggest manual mode or token service |
| Puzzle/Click CAPTCHA | Requires clicking specific images | Warn user: difficult to automate; suggest semi-automated approach |

### 4. Choose a library

Use these criteria to pick the right tool. Read `references/library-selection.md` for
detailed comparison.

**Choose Playwright when:**
- The site is a SPA (Vue, React, Angular) with client-side routing
- You need network response interception to detect login result
- The CAPTCHA involves complex JS interaction (e.g., slider needs Vue component access)
- The site uses AJAX/fetch for login with async response handling

**Choose DrissionPage when:**
- The site is a traditional server-rendered page
- Login is triggered by a simple HTML form or by clicking a button that fires an AJAX
  call (use `page.listen.start()` to capture the POST response)
- Image CAPTCHA only (no complex JS interaction needed)
- Simplicity and fast iteration are priorities

**Hybrid cases:**
- A traditional page that submits via AJAX with encryption is still a good fit for
  DrissionPage — use `page.listen` to capture the response and `page.run_js()` to
  call or replicate the encryption logic.
- If unsure, default to Playwright — it handles both simple and complex sites, and
  its async API is more resilient under network variability.

**DrissionPage note:** Do NOT pass `--disable-blink-features=AutomationControlled`
to ChromeOptions. DrissionPage has its own anti-detection mechanism and this flag
triggers a Chrome warning banner. The reference script pattern (bare `ChromiumOptions()`)
is correct.

### 5. Plan credential iteration strategy

Determine the Cartesian product of usernames × passwords the script must iterate over:

- **Single user × password list**: standard brute-force
- **User range × single password**: iterate user IDs
- **File × file**: full credential-stuffing
- **File × range**: e.g., testing default passwords across accounts

Decide the loop nesting order (outer = username, inner = password is usually best
for caching and efficiency) and set reasonable rate-limiting (delay between attempts).

### 6. Generate the script

Write a complete Python script to the project directory. The script must include these
components (every one is mandatory unless the target site genuinely doesn't need it):

1. **Config block** at the top: URL, file paths, credentials, timing
2. **Progress persistence**: save/load JSON with `tested`, `last_index`, `success_list`, `fail_list`
3. **CAPTCHA handling**: image OCR or slider solving, specific to the detected type
4. **Client-side encryption replication** (if discovered during inspection): either call
   the page's existing encryption function via JS injection, or port the logic to Python
5. **Login attempt function**: fill fields, solve CAPTCHA, submit, detect result
6. **Result detection with error classification**: parse API response, check page URL,
   or scan DOM messages. Map known error strings to human-readable categories so the
   report groups failures by reason rather than showing raw codes
7. **Report generation**: write `result.txt` with summary grouped by failure reason,
   and `successfully_login.txt` appended immediately on each success
8. **Main loop**: iterate credentials, skip already-tested ones, handle Ctrl+C gracefully

Read the relevant reference file for code patterns and a full template:

- `references/playwright-patterns.md` for Playwright-based scripts
- `references/drissionpage-patterns.md` for DrissionPage-based scripts

These references contain production-tested patterns for progress tracking, CAPTCHA
solving, and login detection. Do not reinvent these from scratch — adapt the patterns
to the specific site.

#### Script generation rules

- Use exact selectors found during inspection, not guesses
- Handle edge cases: network timeouts, element not found, unexpected response format
- Write UTF-8 encoding declaration at the top
- Include a `MAX_RETRIES` config for each attempt
- Include a `MAX_CAPTCHA_RETRIES` config specifically for CAPTCHA failures — the script
  must distinguish between "CAPTCHA was wrong, re-fetch and retry" and "credentials were
  rejected, move to next credential"
- Save progress every N attempts (default: every 10) AND on KeyboardInterrupt
- The success file (`successfully_login.txt`) should be appended to immediately on each success
- Use `print()` for real-time progress so the user can monitor

### 7. Run and verify

After generating the script:

1. Run it with a small test (e.g., 2-3 known credentials) to verify:
   - Page navigation works
   - Field filling works
   - CAPTCHA solving works
   - Login detection works (both success and failure cases)
   - Progress saving works
2. If errors occur, fix the script and re-test
3. Once verified, tell the user the script is ready and explain:
   - Where the script is saved
   - What files it will generate (results, progress, success log)
   - How to resume if interrupted
   - Any known limitations (e.g., CAPTCHA accuracy, rate-limiting concerns)

## Troubleshooting

Common issues encountered during script generation and how to handle them:

**Chrome warning banner ("不受支持的命令行标记")**
- Cause: `--disable-blink-features=AutomationControlled` was passed to Chrome.
- Fix: Remove this flag from DrissionPage scripts entirely (DrissionPage doesn't need it).
  In Playwright scripts, it's passed via `launch(args=[...])` and usually harmless.

**Page not accessible (ERR_EMPTY_RESPONSE, geo-block)**
- Cause: The site blocks your IP range or geographic region.
- Fix: Ask the user about VPN/proxy access. If the user has a working reference script,
  use its selectors and API patterns as verified references.

**Login form not visible / wrong tab active**
- Cause: The page has multiple login modes (phone, username/password, QR code) and the
  one you need is hidden.
- Fix: Read the page's JS to find the tab-switching logic. Inject JS to show the correct
  form and override the submit handler if needed.

**Credentials are encrypted before submission**
- Cause: The site's JavaScript encrypts or encodes the username/password before sending.
- Fix: Find the encryption function in the JS files. Either call it directly via
  `page.evaluate()`/`page.run_js()`, or port its logic to Python.

**CAPTCHA accuracy is poor**
- Cause: ddddocr has difficulty with certain CAPTCHA styles (distorted text, overlapping
  characters, unusual fonts).
- Fix: Add character correction mapping (`o→0`, `l→1`, etc.). Increase `MAX_CAPTCHA_RETRIES`
  so wrong guesses don't mark the credential as failed. Consider using a third-party CAPTCHA
  solving service for very difficult CAPTCHAs.

**Rate limiting / IP blocking**
- Cause: Too many requests too quickly.
- Fix: Increase `DELAY_BETWEEN` (0.5s → 2-5s). Add exponential backoff on errors.
  Rotate user agents. Suggest the user use a proxy pool for large-scale testing.

**Element not found during script execution**
- Cause: Timing issues — the script tries to interact with an element before it's rendered,
  or the selector changed.
- Fix: Add explicit waits (`page.wait.ele_displayed()`, `page.wait_for_selector()`).
  Use selectors found during inspection, not guesses. If the page is an SPA, wait for
  `networkidle` after navigation.

## Important principles

- **Inspect before you write.** Never guess selectors. Always navigate to the page first
  with chrome-devtools and find the real element IDs, classes, or CSS paths.
- **Read the JavaScript.** The visible DOM tells you what elements exist, but the JS
  files tell you what they actually do. Trace the submit handler, check for encryption,
  and look for hidden login modes.
- **Trust observation over documentation.** If the user's description or an old reference
  script says `#submitBtn` but the live page only has `#loginId`, use `#loginId`.
  Report discrepancies to the user.
- **Prefer specific selectors.** `#loginBtn` > `button:has-text("登录")` > `button:first`.
  Generic selectors break when the page changes.
- **Distinguish CAPTCHA errors from credential errors.** If the server says "wrong CAPTCHA",
  re-fetch the image and retry. If it says "wrong password", move on. Don't mix the two.
- **Handle both success and failure explicitly.** A login attempt can succeed, fail with
  a known error, fail with an unknown error, or timeout. Handle all four cases.
- **Classify failure reasons.** Map server error codes to human-readable categories so
  the report groups similar failures together (e.g., "Account locked (userClock) — 15
  attempts" rather than 15 separate raw entries).
- **Progress is critical.** These scripts may run for hours testing thousands of credentials.
  If the script crashes or the user interrupts it, they must be able to resume from the
  last tested credential without redoing work.
- **CAPTCHA accuracy is never 100%.** Build in retry logic — if the CAPTCHA was wrong,
  re-fetch and retry rather than marking the credential as failed.
