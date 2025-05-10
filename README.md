# olx_scrape

import time
import csv
import logging
from datetime import datetime, timedelta
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from webdriver_manager.chrome import ChromeDriverManager
from bs4 import BeautifulSoup

# Disable logging noise from webdriver-manager
logging.getLogger('WDM').setLevel(logging.NOTSET)

# --------------- Setup Headless Browser ------------------
chrome_options = Options()
chrome_options.headless = True
chrome_options.add_argument("--no-sandbox")
chrome_options.add_argument("--disable-dev-shm-usage")
chrome_options.add_argument("--disable-gpu")
chrome_options.add_argument("--disable-extensions")
chrome_options.add_argument("--window-size=1920,1080")
chrome_options.add_argument("--log-level=3")  # suppress logs

driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=chrome_options)

# ------------------- Input -------------------
search_query = input("Enter your OLX search term: ").strip()
url = f"https://www.olx.in/items/q-{search_query.replace(' ', '-')}"

driver.get(url)
time.sleep(4)

# ------------------- Load More Loop -------------------
while True:
    try:
        driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
        time.sleep(2)

        load_more_btn = WebDriverWait(driver, 5).until(
            EC.element_to_be_clickable((By.CSS_SELECTOR, 'button[data-aut-id="btnLoadMore"]'))
        )

        # Use JavaScript click to bypass interception
        driver.execute_script("arguments[0].click();", load_more_btn)
        time.sleep(3)

    except Exception:
        # Break silently if button is not found or click fails
        break

# ------------------- Parse Page -------------------
soup = BeautifulSoup(driver.page_source, 'html.parser')
driver.quit()

results = []
items = soup.find_all('li', {'data-aut-id': 'itemBox3'})

def convert_date(text):
    text = text.lower()
    if "today" in text:
        return datetime.now().strftime("%d-%m-%Y")
    elif "yesterday" in text:
        return (datetime.now() - timedelta(days=1)).strftime("%d-%m-%Y")
    return text

# ------------------- Extract Listings -------------------
for item in items:
    try:
        title = item.select_one("span[data-aut-id=itemTitle]").get_text(strip=True)
        price = item.select_one("span[data-aut-id=itemPrice]").get_text(strip=True)
        location = item.select_one("span[data-aut-id=item-location]").get_text(strip=True)
        date_text = item.select_one("span._2jcGx span").get_text(strip=True)
        date = convert_date(date_text)
        link_tag = item.select_one("a")
        link = "https://www.olx.in" + link_tag['href'] if link_tag else "N/A"
        img_tag = item.select_one("img")
        image = img_tag['src'] if img_tag and 'src' in img_tag.attrs else "N/A"

        results.append({
            "Title": title,
            "Price": price,
            "Location": location,
            "Date": date,
            "Image": image,
            "Link": link
        })
    except Exception:
        continue  # skip silently if something fails

# ------------------- Save to CSV -------------------
csv_file = "olx_results.csv"
with open(csv_file, "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["Title", "Price", "Location", "Date", "Image", "Link"])
    writer.writeheader()
    writer.writerows(results)

print(f"\n✅ Scraping completed silently. Data saved to: {csv_file}")
