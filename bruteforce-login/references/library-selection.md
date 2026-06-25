# Library Selection Guide

## Decision flowchart

```
Login page inspected
│
├─ SPA (Vue/React/Angular) with hash routing?
│  └─ YES → Use Playwright
│
├─ Login uses AJAX/fetch (not form POST)?
│  └─ YES → Use Playwright (need network interception)
│
├─ Slider CAPTCHA requiring JS component access?
│  └─ YES → Use Playwright
│
├─ Need multi-browser or multi-context support?
│  └─ YES → Use Playwright
│
├─ Traditional HTML <form> POST?
│  └─ YES → Use DrissionPage (simpler, faster)
│
├─ Image CAPTCHA only, no complex JS?
│  └─ YES → Use DrissionPage
│
└─ Default → Use Playwright (more capable, handles unexpected complexity)
```

## Playwright

- **Strengths**: Async API, network interception, multi-browser, auto-wait, reliable selectors
- **Best for**: SPAs, sites with client-side routing, slider CAPTCHAs requiring Vue/React internals
- **Key APIs used**: `page.locator()`, `page.on('response')`, `page.evaluate()`, `page.wait_for_url()`
- **Install**: `pip install playwright && playwright install chromium`

## DrissionPage

- **Strengths**: Simple synchronous API, built-in listener for network requests, lighter weight
- **Best for**: Traditional server-rendered pages with form POST login
- **Key APIs used**: `page.ele()`, `page.listen.start()`, `page.listen.wait()`
- **Install**: `pip install DrissionPage`

## Dependency table

Include these imports based on the chosen library:

```python
# Playwright
from playwright.async_api import async_playwright
import asyncio
import json

# DrissionPage
from DrissionPage import ChromiumPage, ChromiumOptions
import time

# Both (always include)
import os
import json
import time

# CAPTCHA (include if image CAPTCHA detected)
import ddddocr

# For checking if ddddocr is available:
# pip install ddddocr
```
