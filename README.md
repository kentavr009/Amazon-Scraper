# Amazon parsing on easy level and all by yourself
<p>I came across a script on the Internet that allows you to parse product cards from Amazon. And I just needed a solution to a problem like that.</p>
<p>I wracked my brain while looking for a way to parse product cards from Amazon. The problem is that Amazon uses different design options for different outputs, in particular – if you need to parse the cards with the search query "bags" – the cards will be arranged vertically, as I need it, but if you take, for example, "t-shirts" – then the cards will be arranged horizontally, and in such way the script falls into an error, it works out opening the page, but does not want to scroll.</p>
<p>Moreover, after reading various articles where users are puzzling over how to bypass the captcha on Amazon, I upgraded the script and now it can bypass the captcha if it occurs (it works with 2captcha). The script checks for the presence of a captcha on the page after each loading of a new page, and if the captcha occurs, it sends a request to the 2capcha server, and after receiving the solution, substitutes it and continues to work.</p>
<p>However, how to bypass the captcha is not the most difficult problem, since this is a trivial task nowadays. The more pressing question is how to make the script work not only with the vertical arrangement of product cards, but also with the horizontal one.</p>
<p>Below I will describe in detail what the script includes, demonstrate its work, and if you can help to solve the problem, if you know what to add (change) in the script so that it works on horizontal setup of cards, I will be grateful.</p>
<p>And for now the script can help someone at least in its limited functionality.</p>
<p>So, let's take the script apart piece by piece!</p>
<H2>Preparation</H2>
<p>Firstly, the script imports the modules needed to complete the task:</p>

```python
from selenium import webdriver
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.common.action_chains import ActionChains
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import csv
import os
from time import sleep
import requests
```

<p>Let's take it apart in parts:</p>

```python
from selenium import webdriver 
```

<p>That imports the <code>webdriver</code> class, which allows you to control the browser (in my case Firefox) through the script</p>

```python
 from selenium.webdriver.common.by import By
```

<p>That imports the <code>By</code> class, with which the script will search for elements to parse by XPath (it can search for other attributes, but in this case <code>Xpath</code> will be used)</p>

```python
 from selenium.webdriver.common.keys import Keys
```

<p>That imports the <code>Keys</code> class, which will be used to simulate keystrokes, in the case of this script, it will scroll the page down <code>Keys.PAGE_DOWN</code></p>

```python
 from selenium.webdriver.common.action_chains import ActionChains
```

<p>That imports the <code>ActionChains</code> class to create complex sequential actions, in our case – clicking on the <code>PAGE_DOWN</code> button and waiting for all elements on the page to load (since on Amazon cards are loaded as they are being scrolled)</p>

```python
 from selenium.webdriver.support.ui import WebDriverWait
```

<p>That imports the <code>WebDriverWait</code> class, which waits until the information we are looking for is loaded, for example, a product description, which we will search by <code>Xpath</code></p>

```python
 from selenium.webdriver.support import expected_conditions as EC
```

<p>That imports the <code>expected_conditions</code> class (abbreviated <code>EC</code>) which works in conjunction with the previous class and tells WebDriverWait which specific condition it needs to wait for. That increases the reliability of the script so that it would not start interacting with the unloaded yet content.</p>

```python
 import csv
```

<p>That imports the <code>csv</code> module to work with csv files.</p>

```python
 import os
```

<p>That imports the <code>os</code> module to work with the operating system (creating directories, checking for the files presence, etc.).</p>

```python
 from time import sleep
```

<p>We import the <code>sleep</code> function – this is the function that will pause the script for a specific time (in my case, 2 seconds, but you can set more) so that the elements would load while scrolling.</p>

```python
 import requests
```

<p>That imports the <code>requests</code> library for sending HTTP requests, to interact with the 2captcha recognition service.</p>

<H2>Configuration</H2>

<p>After everything is imported, the script starts configuring the browser for work, in particular:</p>

<p>Installing the API key to access the 2captcha service</p>

```python
  # API key for 2Captcha
API_KEY = "
```

<p>The script contains a user-agent (it can be changed, of course), which is installed for the browser. After that, the browser starts with the specified settings.</p>

```python
 user_agent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"

options = webdriver.FirefoxOptions()
options.add_argument(f"user-agent={user_agent}")

driver = webdriver.Firefox(options=options) 
```

<p>Next comes the captcha solution module. This is exactly the place that users are looking for when they search how to solve a captcha. We will not analyze this piece of code for a long time, since there were no particular problems with it.</p>

<p>In short, the script, after each page load, checks for the presence of a captcha on the page and if it finds it there, solves it by sending it to the 2captcha server. If there is no captcha, it just continues the execution further.</p>

```python
  def solve_captcha(driver):
    # Check for the presence of a captcha on the page
    try:
        captcha_element = driver.find_element(By.CLASS_NAME, 'g-recaptcha')
        if captcha_element:
            print("Captcha detected. Solving...")
            site_key = captcha_element.get_attribute('data-sitekey')
            current_url = driver.current_url
            
            # Send captcha request to 2Captcha
            captcha_id = requests.post(
                'http://2captcha.com/in.php', 
                data={
                    'key': API_KEY, 
                    'method': 'userrecaptcha', 
                    'googlekey': site_key, 
                    'pageurl': current_url
                }
            ).text.split('|')[1]

            # Wait for the captcha to be solved
            recaptcha_answer = ''
            while True:
                sleep(5)
                response = requests.get(f"http://2captcha.com/res.php?key={API_KEY}&action=get&id={captcha_id}")
                if response.text == 'CAPCHA_NOT_READY':
                    continue
                if 'OK|' in response.text:
                    recaptcha_answer = response.text.split('|')[1]
                    break
            
            # Inject the captcha answer into the page
            driver.execute_script(f'document.getElementById("g-recaptcha-response").innerHTML = "{recaptcha_answer}";')
            driver.find_element(By.ID, 'submit').click()
            sleep(5)
            print("Captcha solved.")
    except Exception as e:
        print("No captcha found or error occurred:", e)
</pre>
<H2>Parsing</H2>
<p>Next comes a section of the code that is responsible for sorting pages, loading, and scrolling them</p>
<pre>
  try:
    base_url = "https://www.amazon.in/s?k=bags"

    for page_number in range(1, 10): 
        page_url = f"{base_url}&page={page_number}"

        driver.get(page_url)
        driver.implicitly_wait(10)

        solve_captcha(driver)

        WebDriverWait(driver, 10).until(EC.presence_of_element_located((By.XPATH, '//span[@class="a-size-medium a-color-base a-text-normal"]')))

        for _ in range(5):  
            ActionChains(driver).send_keys(Keys.PAGE_DOWN).perform()
            sleep(2)
```

<p>The next piece is the collection of product data. The most important part. In this part, the script examines the loaded page and takes the data that is specified from there. In our case it is the product name, number of reviews, price, URL, product rating.</p>

```python
  product_name_elements = driver.find_elements(By.XPATH, '//span[@class="a-size-medium a-color-base a-text-normal"]')
        rating_number_elements = driver.find_elements(By.XPATH, '//span[@class="a-size-base s-underline-text"]')
        star_rating_elements = driver.find_elements(By.XPATH, '//span[@class="a-icon-alt"]')
        price_elements = driver.find_elements(By.XPATH, '//span[@class="a-price-whole"]')
        product_urls = driver.find_elements(By.XPATH, '//a[@class="a-link-normal s-underline-text s-underline-link-text s-link-style a-text-normal"]')
        
        product_names = [element.text for element in product_name_elements]
        rating_numbers = [element.text for element in rating_number_elements]
        star_ratings = [element.get_attribute('innerHTML') for element in star_rating_elements]
        prices = [element.text for element in price_elements]
        urls = [element.get_attribute('href') for element in product_urls]
```

<p>Next, the specified data is uploaded to a folder (a csv file is created for each page, which is saved to the output files folder). If the folder is missing, the script creates it.</p>

```python
          output_directory = "output files"
        if not os.path.exists(output_directory):
            os.makedirs(output_directory)
        
        with open(os.path.join(output_directory, f'product_details_page_{page_number}.csv'), 'w', newline='', encoding='utf-8') as csvfile:
            csv_writer = csv.writer(csvfile)
            csv_writer.writerow(['Product Urls', 'Product Name', 'Product Price', 'Rating', 'Number of Reviews'])
            for url, name, price, star_rating, num_ratings in zip(urls, product_names, prices, star_ratings, rating_numbers):
                csv_writer.writerow([url, name, price, star_rating, num_ratings])
```

<p>And the final stage is the completion of work and the release of resources.</p>

```python
  finally:
    driver.quit()
```

<p>The full script</p>

```python
  from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.common.action_chains import ActionChains
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import csv
import os
from time import sleep
import requests

# API key for 2Captcha
API_KEY = "Your API Key"

# Set a custom user agent to mimic a real browser
user_agent = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"

options = webdriver.FirefoxOptions()
options.add_argument(f"user-agent={user_agent}")

driver = webdriver.Firefox(options=options)

def solve_captcha(driver):
    # Check for the presence of a captcha on the page
    try:
        captcha_element = driver.find_element(By.CLASS_NAME, 'g-recaptcha')
        if captcha_element:
            print("Captcha detected. Solving...")
            site_key = captcha_element.get_attribute('data-sitekey')
            current_url = driver.current_url
            
            # Send captcha request to 2Captcha
            captcha_id = requests.post(
                'http://2captcha.com/in.php', 
                data={
                    'key': API_KEY, 
                    'method': 'userrecaptcha', 
                    'googlekey': site_key, 
                    'pageurl': current_url
                }
            ).text.split('|')[1]

            # Wait for the captcha to be solved
            recaptcha_answer = ''
            while True:
                sleep(5)
                response = requests.get(f"http://2captcha.com/res.php?key={API_KEY}&action=get&id={captcha_id}")
                if response.text == 'CAPCHA_NOT_READY':
                    continue
                if 'OK|' in response.text:
                    recaptcha_answer = response.text.split('|')[1]
                    break
            
            # Inject the captcha answer into the page
            driver.execute_script(f'document.getElementById("g-recaptcha-response").innerHTML = "{recaptcha_answer}";')
            driver.find_element(By.ID, 'submit').click()
            sleep(5)
            print("Captcha solved.")
    except Exception as e:
        print("No captcha found or error occurred:", e)

try:
    # Starting page URL
    base_url = "https://www.amazon.in/s?k=bags"

    for page_number in range(1, 2): 
        page_url = f"{base_url}&page={page_number}"

        driver.get(page_url)
        driver.implicitly_wait(10)

        # Attempt to solve captcha if detected
        solve_captcha(driver)

        # Explicit Wait
        WebDriverWait(driver, 10).until(EC.presence_of_element_located((By.XPATH, '//span[@class="a-size-medium a-color-base a-text-normal"]')))

        for _ in range(5):  
            ActionChains(driver).send_keys(Keys.PAGE_DOWN).perform()
            sleep(2)

        product_name_elements = driver.find_elements(By.XPATH, '//span[@class="a-size-medium a-color-base a-text-normal"]')
        rating_number_elements = driver.find_elements(By.XPATH, '//span[@class="a-size-base s-underline-text"]')
        star_rating_elements = driver.find_elements(By.XPATH, '//span[@class="a-icon-alt"]')
        price_elements = driver.find_elements(By.XPATH, '//span[@class="a-price-whole"]')
        product_urls = driver.find_elements(By.XPATH, '//a[@class="a-link-normal s-underline-text s-underline-link-text s-link-style a-text-normal"]')
        
        # Extract and print the text content of each product name, number of ratings, and star rating, urls
        product_names = [element.text for element in product_name_elements]
        rating_numbers = [element.text for element in rating_number_elements]
        star_ratings = [element.get_attribute('innerHTML') for element in star_rating_elements]
        prices = [element.text for element in price_elements]
        urls = [element.get_attribute('href') for element in product_urls]
        
        sleep(5)        
        output_directory = "output files"
        if not os.path.exists(output_directory):
            os.makedirs(output_directory)
        
        with open(os.path.join(output_directory, f'product_details_page_{page_number}.csv'), 'w', newline='', encoding='utf-8') as csvfile:
            csv_writer = csv.writer(csvfile)
            csv_writer.writerow(['Product Urls', 'Product Name', 'Product Price', 'Rating', 'Number of Reviews'])
            for url, name, price, star_rating, num_ratings in zip(urls, product_names, prices, star_ratings, rating_numbers):
                csv_writer.writerow([url, name, price, star_rating, num_ratings])

finally:
    driver.quit()
```

<p>This way the script works without errors, but only for vertical product cards. Here is an example of how the script works.</p>

[<img src="https://img.youtube.com/vi/PQRah0oQwIo/hqdefault.jpg" width="600" height="300"
/>](https://www.youtube.com/embed/PQRah0oQwIo)

<p>I will be glad to discuss it in the comments if you have something to say about it.</p>
