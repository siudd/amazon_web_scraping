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

Try to use Google Colab for this so the web scraping has to run headless.  The function <code>amazon_scrape</code> accepts a string variable <code>sURL</code> for the page want to scape.  It first checks the returned HTML title and see if it is a page for bot detection rather than the page we want.  It will sleep for 1-3 seconds and retry until it is not the bot detection page.  Finally return a <code>BeautifulSoup</code> object. 

Another function <code>amazon_search</code> is to scrap the items from the return search result and store them in a dataframe for future use.

```python
def amazon_search(sKeyWord):
  #get all items attributes from the search result
  page_no = 1
  item_count = 0
  search_result = []
  soup = BeautifulSoup("", 'lxml')
  while page_no <= 7:
    url = "https://www.amazon.ca/s?k=" + urllib.parse.quote(sKeyWord) + "&page=" + str(page_no)
    soup = amazon_scrap(url)
    product_listings = soup.find_all("div", {'class': 'a-section a-spacing-base'})
    for product in product_listings:
      #title
      title = product.find("span", {'class': 'a-size-base-plus a-color-base a-text-normal'}).text.strip()
      href = urllib.parse.unquote(product.find('a', class_='a-link-normal s-underline-text s-underline-link-text s-link-style a-text-normal')['href'])
      #ASIN & Href
      match = re.search(r'/[dg]p/([a-zA-Z0-9]{10})/', href)
      if match:
        asin = match.group(1)
        href = match.group(0)
      else:
        asin = 'video'
      #Prices
      try:
        price = float(product.find('span', class_='a-price').find('span', class_='a-offscreen').text.strip().split('$')[1].replace(',',''))
      except AttributeError:
        price = np.nan
      try:
        original_price = float(product.find('span', class_='a-price a-text-price').find('span', class_='a-offscreen').text.strip().split('$')[1].replace(',',''))
      except AttributeError:
        original_price = np.nan
      try:
        coupon = product.find('span', class_='s-coupon-unclipped').find('span', class_='a-size-base s-highlighted-text-padding aok-inline-block s-coupon-highlight-color').text.strip()
        if "Save" in coupon:
          coupon = float(coupon.split('$')[1].replace(',',''))
        else:
          coupon = float(coupon.split()[0].replace('$',''))
      except AttributeError:
        coupon = np.nan
      rating = product.find('span', class_='a-icon-alt')
      #Rating
      if rating:
          rating = float(rating.text.strip().split()[0])
      else:
          rating = np.nan
      #Bestseller / Amazon's Choice
      if product.find('div',class_='a-row a-badge-region'):
        badge_text = (product.find('div',class_='a-row a-badge-region').find('span',class_='a-badge-text')).text.strip()
        if badge_text == "Amazon's":
          bestseller = False
          amazon_choice = True
        elif badge_text == "Bestseller":
          bestseller = True
          amazon_choice = False
        else:
          bestseller = False
          amazon_choice = False
      else:
        bestseller = False
        amazon_choice = False
      search_result.append([item_count, asin, title, href, price, original_price, coupon, rating, bestseller, amazon_choice])
      item_count = item_count + 1
    page_no = page_no + 1

  dfResult = pd.DataFrame(search_result)
  dfResult = dfResult.rename(columns={0: 'ID'})
  dfResult = dfResult.rename(columns={1: 'ASIN'})
  dfResult = dfResult.rename(columns={2: 'Title'})
  dfResult = dfResult.rename(columns={3: 'Href'})
  dfResult = dfResult.rename(columns={4: 'Price'})
  dfResult = dfResult.rename(columns={5: 'Original_Price'})
  dfResult = dfResult.rename(columns={6: 'Coupon'})
  dfResult = dfResult.rename(columns={7: 'Rating'})
  dfResult = dfResult.rename(columns={8: 'Bestseller'})
  dfResult = dfResult.rename(columns={9: 'Amazon_Choice'})
  dfResult['Original_Price'] = dfResult.apply(lambda row: row['Price'] if pd.isna(row['Original_Price']) else row['Original_Price'], axis=1)
  dfResult['Price_After_Coupon'] = dfResult.apply(lambda row: row['Price'] - row['Coupon'] if not pd.isna(row['Coupon']) else row['Price'], axis=1)
  dfResult['Discount_Percentage'] = ((dfResult['Original_Price'] - dfResult['Price_After_Coupon']) / dfResult['Original_Price']) * 100
  return dfResult
```

This function accepts a string <code>sKeyWord</code> and build the URL needed for web scraping with a loop. Imposed a 7 pages limited of the search result as most search return no more than that.  Item details are extracted from their corresponding elements, first stored in a list and eventually converted into a dataframe with more readable headers for further analysis.

To web scape the search result you want, simply call the functions with below.

```python
df = amazon_search('wireless earbuds') # or any other keywords you like
```

First we want to see the percentage of Bestseller / Amazon's Choice item within the search result.  A pie chart was plot.
![image](https://github.com/siudd/amazon_web_scraping/assets/144144392/d30023c1-a2b2-44ed-bc85-7812b7f2fdc3)

Then, we want to check out the top 10 discounted items in the result.  This also incorporated the coupon as well.
![image](https://github.com/siudd/amazon_web_scraping/assets/144144392/fbc2048a-d0cd-428e-a33b-3c48cffc58e6)

Lastly, we want to know the rating distribution of the search result.
![image](https://github.com/siudd/amazon_web_scraping/assets/144144392/1274895f-1b34-41d3-8228-8f51d8d69697)

With the powerful libraries and proper tweaks & tuning, web scraping on Amazon could be a simple task!
