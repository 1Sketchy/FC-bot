import time
import requests
import re
import random
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

# ==========================================
# 🎛️ THE CONTROL PANEL 🎛️
# ==========================================

TOKEN = "8716815911:AAF4VoJRkkInYKl_WVVrCAHnxrv1GWpWfVU"
CHAT_ID = "8173016143" 

# The base URL without any cache limits
BASE_URL = "https://www.firstcry.com/hot-wheels/5/0/113?ref2=menu_dd_toys_hotwheels_V"

TARGET_KEYWORDS = [
    "batman",
    "lamborghini",
    "ferrari",
    "formula 1",
    "redbull",
    "nissan",
    "toyota",
    "aston martin",
    "ford mustang",
    "porsche",
    "honda"
]

# ==========================================
# ⚙️ THE BOT LOGIC ⚙️
# ==========================================

# DYNAMIC MEMORY: We now only remember what was in stock during the LAST scan.
previously_in_stock = set()

def send_telegram_message(message):
    api_url = f"https://api.telegram.org/bot{TOKEN}/sendMessage?chat_id={CHAT_ID}&text={message}"
    try:
        requests.get(api_url)
    except Exception as e:
        print(f"Failed to send Telegram message: {e}")

def check_firstcry_stock():
    global previously_in_stock 
    current_in_stock = set() # What we find right now
    
    print("\n[🤖] Booting up stealth browser...")
    options = webdriver.ChromeOptions()
    options.add_argument("--headless=new") 
    options.add_argument("--window-size=1920,1080") 
    options.add_argument("--disable-gpu")
    
    # Anti-Bot Stealth Headers
    options.add_argument("--disable-blink-features=AutomationControlled")
    options.add_experimental_option("excludeSwitches", ["enable-automation"])
    options.add_experimental_option('useAutomationExtension', False)
    options.add_argument("user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36")
    
    driver = webdriver.Chrome(options=options)
    
    try:
        # THE CACHE BUSTER: Attach the current time to the URL so FirstCry MUST give us a live page
        live_url = f"{BASE_URL}&nocache={int(time.time())}"
        
        print(f"[🔍] Scanning live database for targeted models...")
        driver.get(live_url)
        
        WebDriverWait(driver, 15).until(
            EC.presence_of_element_located((By.TAG_NAME, "body"))
        )
        
        # Safety Check: Did FirstCry block us with a blank page?
        body_text = driver.find_element(By.TAG_NAME, "body").text.lower()
        if "firstcry" not in body_text and len(body_text) < 50:
            print("[⚠️] Page did not load correctly. FirstCry might be verifying connection. Retrying next cycle.")
            return

        driver.execute_script("window.scrollTo(0, document.body.scrollHeight / 2);")
        time.sleep(2)
        driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
        time.sleep(3) 
        
        # The Ultimate Card-Level Extractor
        extracted_data = driver.execute_script("""
            let validItems = [];
            let seenUrls = new Set();
            let links = document.querySelectorAll('a[href*="/product-detail"]');
            
            links.forEach(link => {
                let href = link.href || "";
                if (href.includes('firstcry.com') && !seenUrls.has(href)) {
                    let card = link.closest('.list_block, .bg-white, .product-card, .item') || link.parentElement.parentElement.parentElement;
                    if (card) {
                        let cardText = card.innerText.toLowerCase();
                        if (!cardText.includes('out of stock') && !cardText.includes('sold out') && !cardText.includes('currently unavailable')) {
                            seenUrls.add(href);
                            validItems.push({
                                "href": href,
                                "card_text": cardText
                            });
                        }
                    }
                }
            });
            return validItems;
        """)

        found_new_items = False
        
        for item in extracted_data:
            href = item["href"]
            card_text = item["card_text"]
            href_lower = href.lower()
            
            for keyword in TARGET_KEYWORDS:
                url_dashed = keyword.replace(" ", "-")
                url_joined = keyword.replace(" ", "")
                
                if re.search(rf"\b{keyword}\b", card_text) or url_dashed in href_lower or url_joined in href_lower:
                    
                    # It's in stock right now, so we add it to the current memory
                    current_in_stock.add(href)
                    
                    # If it WAS NOT in stock 2 minutes ago, it's a new drop!
                    if href not in previously_in_stock:
                        found_new_items = True
                        
                        try:
                            raw_name = href.split("firstcry.com/")[1].split("/")[1].replace("-", " ").title()
                        except:
                            raw_name = f"{keyword.title()} Model"
                            
                        print(f"[🚨] BRAND NEW DROP DETECTED! Target: {raw_name}")
                        message = f"🚨 FIRSTCRY LIVE DROP!\n\nTarget: {keyword.upper()}\nModel: {raw_name}\n\nBuy it instantly here:\n{href}"
                        send_telegram_message(message)
                        time.sleep(1) 
        
        if not found_new_items:
            print("[💤] Scan complete. No NEW drops detected this cycle.")
            
        # VERY IMPORTANT: Update the old memory with the new live data for the next check
        previously_in_stock = current_in_stock
            
    except Exception as e:
        print(f"[⚠️] An error occurred while scanning: {e}")
        
    finally:
        driver.quit()
        print("[💤] Browser closed.")

# --- THE INFINITE LOOP ---
print("=== FIRSTCRY LIVE TRACKER ONLINE ===")
print("Press Ctrl+C in this terminal to stop the bot.")

send_telegram_message("🟢 FirstCry Live-Tracker Booted Up and Hunting!")

while True:
    check_firstcry_stock()
    
    # HUMAN STEALTH WAIT: Waits a random amount of time between 1 min 50s and 2 min 20s
    sleep_time = random.randint(110, 140)
    print(f"[⏳] Going to sleep for {sleep_time} seconds to avoid bot detection. Will check again automatically...")
    time.sleep(sleep_time)
