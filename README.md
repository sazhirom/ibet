
---
<a id="ibet-scrape"></a>
## ~~~ 1. Data scraping ~~~
--- 

Data is collected sequentially from three websites and transferred either to Redis or ClickHouse.

Live betting data is sent to Redis for quick calculations and display.
All data is sent to ClickHouse for dashboards and analytics.
Below is an example script for scraping data from Parimatch, which has an infinite scroll feature.

Interesting challenge: Data is only updated when displayed on the screen, and this protection cannot be bypassed.
Solution: A virtual display combined with zoom-out scaling to speed up scanning.

<details>
  <summary><strong>🖼️ Website image</strong></summary>

  ![Внешний вид сайта](https://raw.githubusercontent.com/sazhirom/images/refs/heads/main/pari-live-section.PNG)
</details>

<details>
  <summary><strong>📜 Parimatch</strong></summary>

```python

from bs4 import BeautifulSoup
from selenium.webdriver.firefox.options import Options
from selenium.webdriver.firefox.service import Service
from selenium.webdriver.common.by import By
from selenium import webdriver
import time
import re
import clickhouse_connect
import pandas as pd
import os
from random import uniform
from datetime import datetime, timedelta

os.environ["DISPLAY"] = ":1.0"

def initialize_driver():
    profile_path = '/home/venediktovga/.mozilla/firefox/a6k6fc50.default-release'
    options = Options()
    options.set_preference("profile", profile_path)
    
    options.set_preference("layout.css.devPixelsPerPx", "0.25")  # Zoom out for faster data scanning
    options.set_preference("browser.sessionstore.privacy_level", 2)

    options.add_argument("--disable-infobars")  
    options.add_argument("--start-maximized")
    options.add_argument("--no-sandbox") 
    options.add_argument("--disable-dev-shm-usage")  
    options.set_preference("permissions.default.image", 2)  # Disable images
    options.set_preference("browser.cache.disk.enable", False)
    options.set_preference("browser.cache.memory.enable", False)

    service = Service("/usr/local/bin/geckodriver")  # Using Mozilla for speed + user agent plugins
    service.log_path = os.devnull

    driver = webdriver.Firefox(service=service, options=options)
    driver.set_window_size(1920, 1080) 
    print('Firefox driver started')
    driver.set_page_load_timeout(70)  
    time.sleep(2)  
    driver.maximize_window()  

    return driver

# Get results archive from the previous day for analytics
def source_page(driver):
    date = (datetime.now() - timedelta(days=1))
    url_part = date.strftime('%Y-%m-%d')
    url = f'https://pari.ru/results?date={url_part}'
    driver.get(url)
    time.sleep(14)
    return date

def extract(driver, date):
    data = driver.page_source
    soup = BeautifulSoup(data, 'html.parser')
    matches = soup.find_all('div', class_='results-event--Me6XJ')
    results_list = []
    
    for match in matches:
        row = []
        row.append(date)  # Date
        category = match.find_previous('div', class_=re.compile(r'results-sport__caption-container--e43SF'))
        row.append(category.text)  # Category

        try:
            subcategory = match.find_previous('div', class_=re.compile(r'overflowed-text--JHSWr results-competition__caption--zmv7q'))
            row.append(subcategory.text)
        except Exception:
            row.append('no subcategory')

        teams = match.find_all('div', class_=re.compile(r'results-event-team__name'))
        team_list = [team.get_text(strip=True) for team in teams]
        row.extend(team_list)  # Team 1 and Team 2

        try:
            row.append(' — '.join([team_list[0], team_list[1]]))  # Event
        except Exception:
            row.append('-')

        try:
            scores = match.find_all('div', class_=re.compile(r'results-scoreBlock__score--XvlMM _summary--Jt8Ej _bold--JaGTY'))
            score_list = [str(score.get_text(strip=True)) for score in scores]
            row.extend(score_list if len(score_list) == 2 else ['-', '-'])
        except Exception:
            row.extend(['-', '-'])

        try:
            if int(score_list[0]) > int(score_list[1]):
                row.append('1')  # Winner
            elif int(score_list[1]) > int(score_list[0]):
                row.append('2')
            elif int(score_list[0]) == int(score_list[1]):
                row.append('X')
            else:
                row.append('-')
        except Exception:
            row.append('-')

        try:
            row.append(int(score_list[0]) - int(score_list[1]))  # Goal difference
            row.append(int(score_list[1]) - int(score_list[0]))
        except Exception:
            row.extend([-1000, -1000])

        try:
            row.append(int(score_list[0]) + int(score_list[1]))  # Total goals
        except Exception:
            row.append(-1000)

        results_list.append(row)

    column_names = ["date", "category", "subcategory", "team1", "team2", "event", "score1", "score2", "result", "f1", "f2", "total"]
    df = pd.DataFrame(results_list, columns=column_names)
    df.to_csv('results.csv', index=False, encoding='UTF-8-sig')
    upload(results_list)

def scroll_container(driver, date):
    count_repeat_break = 0
    count = 0
    while True:
        extract(driver, date)
        print(f'Run {count} data uploaded')

        elements = driver.find_elements(By.CLASS_NAME, "results-event--Me6XJ")
        last_element = elements[-1] if elements else None  

        if last_element:
            driver.execute_script("arguments[0].scrollIntoView({block: 'start', inline: 'nearest'});", last_element)
            count += 1
        print(f'Scrolling attempt {count}')
        time.sleep(uniform(9, 11))

        elements = driver.find_elements(By.CLASS_NAME, "results-event--Me6XJ")
        new_last_element = elements[-1]
        if new_last_element == last_element:
            count_repeat_break += 1
        else:
            count_repeat_break = 0
        if count_repeat_break == 3:
            break

    client = clickhouse_connect.get_client(host='10.140.0.7', port=8123, username='default', password='******')
    client.insert('ibet.results_date', [(datetime.now(),)], column_names='date')

# Direct upload to ClickHouse
def upload(extracted_data):
    client = clickhouse_connect.get_client(host='10.140.0.7', port=8123, username='default', password='******')
    client.insert('ibet.results', extracted_data, column_names=["date", "category", "subcategory", "team1", "team2", "event", "score1", "score2", "result", "f1", "f2", "total"])

def main():
    try:
        driver = initialize_driver()
        date = source_page(driver)
        scroll_container(driver, date)
    finally:
        driver.quit()

if __name__ == '__main__':
    main()
```

</details>

<br></br>

---
<a id="ibet-merge"></a>

## ~~~ 2. Data wrangling ~~~
--- 


### Merge
 
<details>
  <summary><strong>📜 Parimatch</strong></summary>

```python



<details>
  <summary><strong>📜 Merge </strong></summary>

```python

import redis
import time
from datetime import datetime, timedelta
from random import uniform
import json
import pandas as pd
from io import StringIO
from kafka import KafkaProducer

# Complex logic ahead

redis_client = redis.StrictRedis(
    host='localhost', 
    port=6379, 
    password='*****', 
    decode_responses=True
)

# To synchronize scraping, we send timing data at random intervals.
# Scrapers start synchronized data collection every 4 minutes ±60 seconds.
def publish_timing():
    result = []
    big_pause_delta = uniform(140, 250)
    big_pause = (datetime.now() + timedelta(seconds=big_pause_delta)).timestamp()
    delta1 = uniform(15, 21)
    time1 = (datetime.now() + timedelta(seconds=delta1)).timestamp()
    result.extend([big_pause, time1])
    message = json.dumps(result)
    redis_client.publish('timings', message)
    print(f'Message sent: {message}')
    return big_pause

# Before starting, wait for scrapers to confirm readiness
def gotovo():
    pubsub = redis_client.pubsub()
    pubsub.subscribe('gotovo')
    count = 0
    for message in pubsub.listen():
        if message['type'] == 'message' and message['data'] == 'gotovo':
            count += 1
            print(f'Received responses: {count}')
            if count == 3:
                return True
        elif message['type'] == 'message' and message['data'] == 'ne gotovo':
            return False

# These functions split team names into separate words, transliterate if needed, and compare words letter by letter.
# Standard similarity comparison libraries do not work!
def are_words_equal_with_tolerance(word1, word2):

    if abs(len(word1) - len(word2)) > 2:
        return False  

    # Count the number of different characters
    diff_count = sum(1 for a, b in zip(word1, word2) if a != b)
    
    # Add the difference in length as additional mismatches
    diff_count += abs(len(word1) - len(word2))
    
    # If they differ by more than 2 characters, they are considered different
    return diff_count <= 2

def compare_names(name1, name2):
    # Split names into parts based on "-" or "—"
    parts1 = [part.strip() for part in name1.replace('—', '-').split('-') if len(part.strip()) >= 3]
    words1_1 = [word.strip() for word in parts1[0].split(' ') if len(word.strip()) > 4]
    words1_2 = [word.strip() for word in parts1[1].split(' ') if len(word.strip()) > 4]
    parts2 = [part.strip() for part in name2.replace('—', '-').split('-') if len(part.strip()) >= 3]
    words2_1 = [word.strip() for word in parts2[0].split(' ') if len(word.strip()) > 4]
    words2_2 = [word.strip() for word in parts2[1].split(' ') if len(word.strip()) > 4]
    check1 = False
    check2 = False

    for words1 in words1_1:
        for words2 in words2_1:
            if are_words_equal_with_tolerance(words1, words2):
                check1 = True
                break
    for words1 in words1_2:
        for words2 in words2_2:
            if are_words_equal_with_tolerance(words1, words2):
                check2 = True
                break
            
    return check1 and check2

# Simple merge - combining two DataFrames
def pari_olimp(df1, df2):
    columns_pari = ['event', '1', 'X', '2', 'F1', 'F2', 'Tb', 'Tm', 'F', 'T', 'timestamp', 'category', 'subcategory']
    columns_olimp = ['event_olimp', '1_olimp', 'X_olimp', '2_olimp', 'F1_olimp', 'F2_olimp', 'Tb_olimp', 'Tm_olimp', 'F_olimp', 'T_olimp', 'timestamp_olimp']
    
    df2.columns = columns_olimp
    match_indexes = []

    try:
        for index1, row1 in df1.iterrows():
            for index2, row2 in df2.iterrows():
                if compare_names(row1['event'], row2['event_olimp']):
                    match_indexes.append([index1, index2])
    except Exception as e:
        print(row1, row2)         

    df = pd.DataFrame(columns=columns_pari + columns_olimp)
    for index1, index2 in match_indexes:
        row1 = df1.iloc[index1]
        row2 = df2.iloc[index2]
        combined = pd.DataFrame({
            **row1.to_dict(),    
            **row2.to_dict()     
        }, index=[0])
        dataframes = [df, combined]
        valid_dataframes = [df for df in dataframes if not df.empty and not df.isna().all().all()]
        df = pd.concat(valid_dataframes, ignore_index=True)

    df.drop_duplicates(subset=['event'], keep='last', inplace=True)
    df_olimp = df[['event', 'event_olimp', '1', '1_olimp', 'X', 'X_olimp', '2', '2_olimp', 'F1', 'F1_olimp', 'F2', 'F2_olimp', 'Tb', 'Tb_olimp', 'Tm', 'Tm_olimp', 'F', 'F_olimp', 'T', 'T_olimp', 'timestamp', 'timestamp_olimp', 'category', 'subcategory']]   

    return df_olimp

# Upload mismatched coefficients to Kafka
def send_dataframe_to_kafka(df, topic='ibet', bootstrap_servers='10.140.0.2:9092'):
    producer = KafkaProducer(
        bootstrap_servers=bootstrap_servers,
        value_serializer=lambda v: json.dumps(v).encode('utf-8')
    )
    
    for _, row in df.iterrows():
        message = row.to_dict()
        producer.send(topic, message)
        
    producer.flush()
    producer.close()

# Main function:  
# 1. Wait for scrapers to be ready.  
# 2. Send synchronized scraping time.  
# 3. Once all data is received, compare values.  
# 4. Load the data into ClickHouse and repeat.

def main(): 
    if gotovo():
        try:
            print('Ready')
            time.sleep(1)
            while True:
                big_pause = publish_timing()
                print(f'Timing sent at {datetime.now()}')
                print('Started listening')
                pinn, olimp = listen()
                send_dataframe_to_kafka(pinn)
                send_dataframe_to_kafka(olimp)
                print(f'Sleeping at {datetime.now()}')
                while time.time() < big_pause + 10:
                    time.sleep(1)
                print(f'Woke up at {datetime.now()}')
        except KeyboardInterrupt:
            print('Stopping...')   
    else:
        raise ValueError('Not ready')

if __name__ == '__main__':
    main()
```

</details>  
<br>

---
<a id="ibet-VNC"></a>
