# Amazon Web Scraping with Python (Selenium &amp; BeautifulSoup)

Playing around with web scraping with Python and want to extract product data from the search result page.  Initially went for BS4 & Request but the successful rate of evading Amazon bot detection wasn't good.  So end up switch to Selenium and thing almost work flawlessly.  Here is the core function to perform the web scraping.

```python
def amazon_scrap(sURL):
  #selenium config
  options = webdriver.ChromeOptions()
  options.add_argument('--ignore-certificate-errors')
  options.add_argument('--incognito')
  options.add_argument('--headless')
  options.add_argument("--window-size=1920,1200")
  options.add_argument("start-maximized")
  options.add_argument("disable-infobars")
  options.add_argument("--disable-extensions")
  options.add_argument("--disable-gpu")
  options.add_argument("--disable-dev-shm-usage")
  options.add_argument("--no-sandbox")
  driver = webdriver.Chrome(options=options)

  driver.get(sURL)
  webpage = driver.page_source
  bs4soup = BeautifulSoup(webpage, "lxml")
  while "<title>Amazon.ca Something Went Wrong / Quelque chose s'est mal passé</title>" in webpage:
    time.sleep(random.uniform(1,3))
    driver.get(sURL)
    webpage = driver.page_source
    bs4soup = BeautifulSoup(webpage, "lxml")
  driver.quit()
  return bs4soup
```
