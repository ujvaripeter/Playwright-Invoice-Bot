#!/usr/bin/env python3
import random
import time
import traceback
from playwright.sync_api import sync_playwright, TimeoutError as PWTimeout

# DEMO URL - replace with your system URL
LEDGER_URL  = "https://demo-ledger-url.example.com/login.pl"
LEDGER_USER = "YOUR_USERNAME"
LEDGER_PASS = "YOUR_PASSWORD"
STORE_NAME  = "YOUR_STORE_NAME"
SELECT_NAME = "store_selector"
HEADLESS    = False

INV_NUMBER = input("Enter invoice number: ").strip()

def log(msg: str) -> None:
    print(f"[{time.strftime('%H:%M:%S')}] {msg}", flush=True)

def human_move(page, selector: str, steps=(15, 30)) -> None:
    box = page.locator(selector).bounding_box()
    if not box:
        return
    cx = box["x"] + box["width"] / 2
    cy = box["y"] + box["height"] / 2
    page.mouse.move(cx, cy, steps=random.randint(*steps))

def find_menu_frame(page):
    for fr in page.frames:
        if fr == page.main_frame:
            continue
        if fr.name.lower().startswith("menu") or "menu" in fr.url.lower():
            return fr
    return None

def open_submenu(frame, root_id: str, submenu_id: str, timeout=8.0) -> None:
    root_loc = frame.locator(f"#{root_id}")
    root_loc.scroll_into_view_if_needed()
    root_loc.click(force=True)
    deadline = time.time() + timeout
    while time.time() < deadline:
        visible = frame.evaluate(
            "(sid) => { const el = document.getElementById(sid); return el && el.style.display != 'none'; }",
            submenu_id
        )
        if visible:
            return
        frame.evaluate("([sub,root]) => SwitchMenu(sub, root)", submenu_id, root_id)
        time.sleep(0.2)
    raise RuntimeError(f"Failed to open submenu: #{submenu_id} ( {root_id} )")

def main():
    p = sync_playwright().start()
    browser = p.firefox.launch(headless=HEADLESS, args=["--no-sandbox"])
    ctx = browser.new_context(ignore_https_errors=True)
    page = ctx.new_page()
    log("🦊 Firefox started…")

    wait   = lambda sel, **kw: page.wait_for_selector(sel, **kw)
    fill   = lambda sel, txt: (wait(sel, timeout=30_000), human_move(page, sel), page.fill(sel, txt))
    click  = lambda sel:      (wait(sel, timeout=40_000), human_move(page, sel), page.click(sel), page.wait_for_load_state("networkidle"))
    select = lambda sel, val: (wait(sel, timeout=30_000), human_move(page, sel), page.select_option(sel, label=val))

    try:
        # Login
        page.goto(LEDGER_URL, wait_until="networkidle")
        fill('input[name="login"]',    LEDGER_USER)
        fill('input[name="password"]', LEDGER_PASS)
        click('input[type="submit"][value="Login"]')

        # Store selection
        select(f'select[name="{SELECT_NAME}"]', STORE_NAME)
        click('input[type="submit"][value="Continue"]')

        # Menu navigation: Revenues -> Lists -> Invoice List
        menu_fr = find_menu_frame(page)
        if not menu_fr:
            raise RuntimeError("Menu frame not found!")
        open_submenu(menu_fr, "menu1", "sub1")
        open_submenu(menu_fr, "menu3", "sub3")
        link = menu_fr.locator('#sub3 >> text=Invoice List')
        link.scroll_into_view_if_needed()
        link.click(force=True)

        # Invoice search
        time.sleep(1.0)
        main_fr = page.frame(name="main_window")
        inv = main_fr.locator('input[name="invnumber"]')
        inv.wait_for(timeout=8000)
        inv.fill(INV_NUMBER)
        cont = main_fr.locator('input[type="submit"][value="Continue"]')
        cont.wait_for(timeout=8000)
        cont.click()

        # Invoice details
        time.sleep(1.0)
        invoice_link = main_fr.locator(f'a:has-text("{INV_NUMBER}")')
        invoice_link.wait_for(timeout=8000)
        invoice_link.click()

        time.sleep(1.0)
        sz_nr = main_fr.locator(
            '//th[normalize-space(.)="Invoice Number"]/following-sibling::td'
        ).first.text_content().strip()
        customer = main_fr.locator('input[name="customer"]').input_value().strip()
        pay_mode = main_fr.locator('input[name="shipvia"]').input_value().strip()
        total = main_fr.locator(
            'xpath=//th[normalize-space(.)="Total Amount"]/following-sibling::td[1]'
        ).text_content().strip()

        log("📋 Retrieved data:")
        print(f"Invoice Number   : {sz_nr}")
        print(f"Customer Name    : {customer}")
        print(f"Payment Method   : {pay_mode}")
        print(f"Total Amount (HUF): {total}")

    except PWTimeout as te:
        log(f"⏱️ Playwright timeout: {te}")
        traceback.print_exc()
    except Exception as e:
        log(f"💥 Error: {e}")
        traceback.print_exc()
    finally:
        log("🚦 Browser remains open. Press Ctrl+C to close.")
        try:
            while True:
                time.sleep(3600)
        except KeyboardInterrupt:
            pass
        browser.close()
        p.stop()

if __name__ == "__main__":
    main()
