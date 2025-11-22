
#!/usr/bin/env python3
"""
Simple scraper to find "Springbank" products on the Craft Cellars Scotch collection
(with availability filter) and report a best-effort inventory status.

Usage:
    pip install requests beautifulsoup4 lxml
    python3 scripts/scrape_springbank.py

Notes:
 - Be polite: don't hammer the site. This script includes a small delay.
 - Respect robots.txt and site terms of service. If the site provides an API,
   prefer using it (Shopify stores often expose /products/<handle>.json).
"""
import re
import time
import urllib.parse
from typing import List, Tuple

import requests
from bs4 import BeautifulSoup

BASE = "https://craftcellars.ca"
COLLECTION_URL = "https://craftcellars.ca/collections/scotch?filter.v.availability=1"
HEADERS = {
    "User-Agent": "Mozilla/5.0 (compatible; simple-scraper/1.0; +https://example.com/bot)"
}
REQUEST_DELAY = 1.0  # seconds between requests


def fetch(url: str) -> str:
    resp = requests.get(url, headers=HEADERS, timeout=15)
    resp.raise_for_status()
    return resp.text


def find_product_links_from_collection(html: str) -> List[Tuple[str, str]]:
    """
    Return list of (product_handle_or_href, product_title_guess)
    Searches for anchors linking to /products/<handle>
    """
    soup = BeautifulSoup(html, "lxml")
    anchors = soup.find_all("a", href=re.compile(r"^/products/"), limit=500)
    seen = {}
    for a in anchors:
        href = a.get("href").split("?")[0]  # strip query
        # try to get a readable title from the anchor text or title attr
        title = (a.get("title") or a.get_text(separator=" ", strip=True) or "").strip()
        if not title:
            # fallback: look for product tile children
            child = a.find(["h2", "h3", "span"], string=True)
            if child:
                title = child.get_text(strip=True)
        if href and href not in seen:
            seen[href] = title
    return list(seen.items())


def find_springbank_products(links: List[Tuple[str, str]]) -> List[Tuple[str, str]]:
    matches = []
    for href, title in links:
        combined = f"{href} {title}".lower()
        if "springbank" in combined or "spring bank" in combined or "spring-bank" in combined:
            matches.append((href, title or "(no title found)"))
    return matches


def detect_availability_from_product_page(html: str) -> str:
    """
    Heuristic attempts:
    1) Look for Add to cart / Sold out / Out of stock button text
    2) Look for link[itemprop=availability] (Shopify often uses link with itemprop)
    3) Look for JSON snippet with "available":true/false
    4) Fallback: check for 'sold out' keywords anywhere
    """
    soup = BeautifulSoup(html, "lxml")
    # 1) Buttons
    btn = soup.find(lambda tag: tag.name in ("button", "input") and tag.get_text(strip=True) and
                    re.search(r"(add to cart|sold out|out of stock|notify me|pre-order|unavailable|add to basket|in cart)", tag.get_text(strip=True), re.I))
    if btn:
        return btn.get_text(strip=True)

    # 2) itemprop availability (link href often contains InStock/OutOfStock)
    link_avail = soup.find("link", itemprop="availability")
    if link_avail and link_avail.get("href"):
        if "InStock" in link_avail["href"]:
            return "In stock"
        if "OutOfStock" in link_avail["href"]:
            return "Out of stock"

    # 3) look for product JSON style availability flags
    text = soup.get_text(" ", strip=True)
    if re.search(r'"available"\s*:\s*true', html, re.I):
        return "Available (product JSON says available:true)"
    if re.search(r'"available"\s*:\s*false', html, re.I):
        return "Unavailable (product JSON says available:false)"

    # 4) generic keyword search
    if re.search(r"(sold out|out of stock|unavailable)", text, re.I):
        return "Sold out / unavailable (detected on page)"
    if re.search(r"(add to cart|add to basket|in stock|buy now)", text, re.I):
        return "Likely available (page contains buy/add-to-cart text)"

    return "Unknown (could not determine from static HTML)"


def full_url(href: str) -> str:
    return urllib.parse.urljoin(BASE, href)


def main():
    print("Fetching collection page:", COLLECTION_URL)
    try:
        col_html = fetch(COLLECTION_URL)
    except Exception as e:
        print("Failed to fetch collection page:", e)
        return

    product_links = find_product_links_from_collection(col_html)
    print(f"Found {len(product_links)} product links on the collection page (raw).")

    spring_products = find_springbank_products(product_links)
    if not spring_products:
        print("No obvious Springbank products found in the collection HTML. Possible reasons:")
        print("- The products are loaded via JavaScript after page load (JS rendering).")
        print("- The product name variant doesn't include the word 'Springbank' exactly.")
        print("If the page is JavaScript-driven, consider using Playwright/Selenium or try the product JSON endpoint.")
        return

    print(f"Found {len(spring_products)} Springbank candidates:")
    for href, title in spring_products:
        url = full_url(href)
        print("\n---")
        print("Product:", title)
        print("URL:", url)
        try:
            time.sleep(REQUEST_DELAY)
            prod_html = fetch(url)
        except Exception as e:
            print("  Failed to fetch product page:", e)
            continue
        status = detect_availability_from_product_page(prod_html)
        print("  Inventory status (heuristic):", status)

    print("\nDone.")


if __name__ == "__main__":
    main()
