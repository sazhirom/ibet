
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
 
Live betting data is received via Redis from three instances and sent to the coordinator.

Goal: Quickly match events and identify odds with the most significant differences.

Challenges:
Transliteration between Cyrillic and foreign bookmaker names.
Each bookmaker formats team and league names differently.

Solution:
Transliteration & fuzzy matching of team/player names, comparing words and letters separately using custom functions.
After processing, data is loaded into ClickHouse via Kafka.

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

# This part is quite complex

redis_client = redis.StrictRedis(
    host='localhost', 
    port=6379, 
    password='*****', 
    decode_responses=True
)

# First, for synchronization, we send timing signals randomly every 4 minutes ±60 seconds.
# The scrapers start synchronous data collection within this time window.
def publish_timing():
    result = []
    big_pause_delta = uniform(140, 250)
    big_pause = (datetime.now() + timedelta(seconds=big_pause_delta)).timestamp()
    delta1 = uniform(15, 21)
    time1 = (datetime.now() + timedelta(seconds=delta1)).timestamp()
    result.extend([big_pause, time1])
    message = json.dumps(result)
    redis_client.publish('timings', message)
    print(f'Sent message {message}')
    return big_pause

# Before starting data collection, we wait for scrapers to be ready and receive confirmation
def gotovo():
    pubsub = redis_client.pubsub()
    pubsub.subscribe('gotovo')
    count = 0
    for message in pubsub.listen():
        if message['type'] == 'message' and message['data'] == 'gotovo':
            count += 1
            print(f'Received responses {count}')
            if count == 3:
                return True
        elif message['type'] == 'message' and message['data'] == 'ne gotovo':
            return False

# Functions split team names into separate words, perform transliteration if necessary, and compare words letter by letter.
# Standard libraries for similarity evaluation do not work in this case!
def are_words_equal_with_tolerance(word1, word2):
    if abs(len(word1) - len(word2)) > 2:
        return False  

    # Count the number of different characters
    diff_count = sum(1 for a, b in zip(word1, word2) if a != b)
    
    # Add the length difference to the mismatch count if words have different lengths
    diff_count += abs(len(word1) - len(word2))
    
    # If words differ by more than 2 characters, they are considered not equal
    return diff_count <= 2

def compare_names(name1, name2):
    # Split names into parts using "-" or "—" as separators
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

# Simple merge - combine two DataFrames into one
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
        df = pd.concat([df, combined], ignore_index=True)
    
    df.drop_duplicates(subset=['event'], keep='last', inplace=True)
    df_olimp = df[['event', 'event_olimp', '1', '1_olimp', 'X', 'X_olimp', '2', '2_olimp', 
                   'F1', 'F1_olimp', 'F2', 'F2_olimp', 'Tb', 'Tb_olimp', 'Tm', 'Tm_olimp', 
                   'F', 'F_olimp', 'T', 'T_olimp', 'timestamp', 'timestamp_olimp', 'category', 'subcategory']]

    return df_olimp

# Simple merge - combine two DataFrames into one
def pari_pinn(df1, df2):
    columns_pari = ['event', '1', 'X', '2', 'F1', 'F2', 'Tb', 'Tm', 'F', 'T', 'timestamp', 'category', 'subcategory']
    columns_pinn = ['timestamp_pinn', 'cate_pinn', 'event_pinn', 'event_reverse_pinn', '1_pinn', 'X_pinn', '2_pinn', 
                    'F1_pinn', 'F2_pinn', 'Tb_pinn', 'Tm_pinn', 'F_pinn', 'T_pinn']
    
    df2.columns = columns_pinn

    match_indexes = []
    for index1, row1 in df1.iterrows():
        for index2, row2 in df2.iterrows():
            if compare_names(row1['event'], row2['event_pinn']) or compare_names(row1['event'], row2['event_reverse_pinn']):
                match_indexes.append([index1, index2])
            if not compare_names(row1['event'], row2['event_pinn']) and compare_names(row1['event'], row2['event_reverse_pinn']):
                df2.at[index2, 'event_pinn'] = row2['event_reverse_pinn']
                df2.at[index2, '1_pinn'], df2.at[index2, '2_pinn'] = row2['2_pinn'], row2['1_pinn']
                df2.at[index2, 'F1_pinn'], df2.at[index2, 'F2_pinn'] = row2['F2_pinn'], row2['F1_pinn']

    df = pd.DataFrame(columns=columns_pari + columns_pinn)
    for index1, index2 in match_indexes:
        row1 = df1.iloc[index1]
        row2 = df2.iloc[index2]
        combined = pd.DataFrame({
            **row1.to_dict(),    
            **row2.to_dict()     
        }, index=[0])
        df = pd.concat([df, combined], ignore_index=True)
    
    df.drop_duplicates(subset=['event'], keep='last', inplace=True)   
    df_pinn = df[['event', 'event_pinn', '1', '1_pinn', 'X', 'X_pinn', '2', '2_pinn', 
                  'F1', 'F1_pinn', 'F2', 'F2_pinn', 'Tb', 'Tb_pinn', 'Tm', 'Tm_pinn', 
                  'F', 'F_pinn', 'T', 'T_pinn', 'timestamp', 'timestamp_pinn', 'category', 'subcategory', 'cate_pinn']]

    index_to_drop = df_pinn[df_pinn['category'].str.lower() != df_pinn['cate_pinn'].str.lower()].index
    print('Removing rows:', index_to_drop)
    df_pinn.drop(index=index_to_drop, inplace=True)
    df_pinn.drop(columns='cate_pinn', inplace=True)  
    
    return df_pinn
# Find coefficient differences for matching events from merged DataFrames
def compare(df, event, event_2, _1, _1_2, X, X_2, _2, _2_2, F1, F1_2, F2, F2_2, Tb, Tb_2, Tm, Tm_2, F, F_2, T, T_2, timestamp, timestamp_2, label, subcategory):
    
    numeric_columns = [_1, _1_2, X, X_2, _2, _2_2, F1, F1_2, F2, F2_2, Tb, Tb_2, Tm, Tm_2, F, F_2, T, T_2, timestamp, timestamp_2]
    for col in numeric_columns:
        df[col] = pd.to_numeric(df[col], errors='coerce')

    bets = []

    for index, row in df.iterrows():
        # Compare coefficients for match outcomes (1, X, 2)
        for pair in [[_1, _1_2], [X, X_2], [_2, _2_2]]:
            try:
                coef = max(row[pair[0]] / row[pair[1]], row[pair[1]] / row[pair[0]])
                if coef > 1.15 and abs(row[timestamp] - row[timestamp_2]) < 30 and (1.1 <= row[pair[0]] <= 2 or 1.1 <= row[pair[1]] <= 3):
                    row_bets = [
                        row[event], row[label], str(pair[0]).replace('_', ''), '-', 
                        row[pair[0]], row[pair[1]],
                        'Parimatch' if row[pair[0]] > row[pair[1]] else 'pinnacle_or_olimp',
                        round(coef, 2), row[timestamp], row[timestamp_2]
                    ]
                    bets.append(row_bets)
            except Exception as e:
                print(f'Error encountered: {e}')
                pass
            
        # Compare handicap coefficients
        for pair in [[F1, F1_2], [F2, F2_2]]:
            if row[F] == row[F_2]:
                try:
                    coef = max(row[pair[0]] / row[pair[1]], row[pair[1]] / row[pair[0]])
                    if coef > 1.15 and abs(row[timestamp] - row[timestamp_2]) < 30 and (1.3 <= row[pair[0]] <= 2 or 1.1 <= row[pair[1]] <= 3):
                        row_bets = [
                            row[event], row[label], str(pair[0]).replace('_', ''), row[F], 
                            row[pair[0]], row[pair[1]],
                            'Parimatch' if row[pair[0]] > row[pair[1]] else 'pinnacle_or_olimp',
                            round(coef, 2), row[timestamp], row[timestamp_2]
                        ]
                        bets.append(row_bets)
                except Exception as e:
                    print(f'Error encountered: {e}')
                    pass

        # Compare total over/under coefficients
        for pair in [[Tb, Tb_2], [Tm, Tm_2]]:
            if row[T] == row[T_2]:
                try:
                    coef = max(row[pair[0]] / row[pair[1]], row[pair[1]] / row[pair[0]])
                    if coef > 1.15 and abs(row[timestamp] - row[timestamp_2]) < 30 and (1.1 <= row[pair[0]] <= 2 or 1.1 <= row[pair[1]] <= 3):
                        row_bets = [
                            row[event], row[label], str(pair[0]).replace('_', ''), row[T], 
                            row[pair[0]], row[pair[1]],
                            'Parimatch' if row[pair[0]] > row[pair[1]] else 'pinnacle_or_olimp',
                            round(coef, 2), row[timestamp], row[timestamp_2]
                        ]
                        bets.append(row_bets)
                except Exception as e:
                    print(f'Error encountered: {e}')
                    pass

    df = pd.DataFrame(bets, columns=['Event', 'Category', 'Bet Type', 'F_T', 'Coef1', 'Coef2', 'Platform', 'Ratio', 'Timestamp', 'Timestamp_2'])  
    df.sort_values(by=['Event', 'Ratio'], inplace=True)    
    df.drop_duplicates(subset='Event', keep='last', inplace=True, ignore_index=True) 
    df.sort_values(by=['Ratio'], ascending=False, inplace=True)   
    return df
def listen():
    pubsub = redis_client.pubsub()
    pubsub.subscribe('df')
    df_pari, df_pinn, df_olimp = None, None, None
    count_parimatch, count_pinnacle, count_olimpbet = 0, 0, 0
    
    # Listening for incoming messages in the Redis channel
    for message in pubsub.listen():
        if message['type'] == 'message':
            print('Received a message')
            string_df = json.loads(message['data'])
            
            # Processing data from Pinnacle
            if string_df['source'] == 'pinnacle':
                count_pinnacle += 1
                print(f'Received Pinnacle dataframe {count_pinnacle}')
                df_pinnacle = string_df['data']
                df_pinn = pd.read_csv(StringIO(df_pinnacle))
                print(df_pinn.shape)
                print(df_pinn.head())
                
            # Processing data from Parimatch
            if string_df['source'] == 'parimatch':
                count_parimatch += 1
                print(f'Received Parimatch dataframe {count_parimatch}')
                df_parimatch = string_df['data']
                df_pari = pd.read_csv(StringIO(df_parimatch))
                print(df_pari.shape)
                print(df_pari.head())
                
            # Processing data from Olimpbet
            if string_df['source'] == 'olimpbet':
                count_olimpbet += 1
                print(f'Received Olimpbet dataframe {count_olimpbet}')
                df_olimpbet = string_df['data']
                df_olimp = pd.read_csv(StringIO(df_olimpbet))
                print(df_olimp.shape)
                print(df_olimp.head())
                
            # Once all three dataframes are received, proceed with merging and comparison
            if count_parimatch == 1 and count_olimpbet == 1 and count_pinnacle == 1:
                print('All data received')

                df_pari_olimp = None
                df_pari_pinn = None

                # Merge Parimatch and Olimpbet data
                if df_pari is not None and df_olimp is not None:
                    df_pari_olimp = pari_olimp(df_pari, df_olimp)

                # Merge Parimatch and Pinnacle data
                if df_pari is not None and df_pinn is not None:
                    df_pari_pinn = pari_pinn(df_pari, df_pinn)
                    
                # Compare Parimatch and Pinnacle
                if df_pari_pinn is not None:  
                    df_pinn_final = compare(
                        df_pari_pinn, 
                        event='event', 
                        event_2='event_pinn', 
                        _1='1', 
                        _1_2='1_pinn', 
                        X='X', 
                        X_2='X_pinn', 
                        _2='2', 
                        _2_2='2_pinn', 
                        F1='F1', 
                        F1_2='F1_pinn', 
                        F2='F2', 
                        F2_2='F2_pinn', 
                        Tb='Tb', 
                        Tb_2='Tb_pinn', 
                        Tm='Tm', 
                        Tm_2='Tm_pinn', 
                        F='F', 
                        F_2='F_pinn', 
                        T='T', 
                        T_2='T_pinn', 
                        timestamp='timestamp', 
                        timestamp_2='timestamp_pinn', 
                        label='category',
                        subcategory='subcategory'
                    )
                    print('Comparison of Parimatch and Pinnacle completed')
                    df_pinn_final['Platform'] = df_pinn_final['Platform'].str.replace('pinnacle_or_olimp', 'Pinn_Pari')
                    df_pinn_final['Platform'] = df_pinn_final['Platform'].str.replace('Parimatch', 'Pari_Pinn')
                    print(df_pinn_final.head())
                    print(df_pinn_final.shape)

                # Compare Parimatch and Olimpbet
                if df_pari_olimp is not None:
                    df_olimp_final = compare(
                        df_pari_olimp, 
                        event='event', 
                        event_2='event_olimp', 
                        _1='1', 
                        _1_2='1_olimp', 
                        X='X', 
                        X_2='X_olimp', 
                        _2='2', 
                        _2_2='2_olimp', 
                        F1='F1', 
                        F1_2='F1_olimp', 
                        F2='F2', 
                        F2_2='F2_olimp', 
                        Tb='Tb', 
                        Tb_2='Tb_olimp', 
                        Tm='Tm', 
                        Tm_2='Tm_olimp', 
                        F='F', 
                        F_2='F_olimp', 
                        T='T', 
                        T_2='T_olimp', 
                        timestamp='timestamp', 
                        timestamp_2='timestamp_olimp', 
                        label='category',
                        subcategory='subcategory'
                    )
                    print('Comparison of Parimatch and Olimpbet completed')
                    df_olimp_final['Platform'] = df_olimp_final['Platform'].str.replace('pinnacle_or_olimp', 'Olimp_Pari')
                    df_olimp_final['Platform'] = df_olimp_final['Platform'].str.replace('Parimatch', 'Pari_Olimp')
                    print(df_olimp_final.head())
                    print(df_olimp_final.shape)
                
                # Return final dataframes after merging and comparison
                return df_pinn_final, df_olimp_final
# Send the mismatched coefficients to Kafka
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

# Main function that orchestrates the entire process:
# 1. Waits for scrapers to be ready.
# 2. Sends timing for synchronized scraping.
# 3. Listens for incoming data.
# 4. Compares and processes the received data.
# 5. Sends the results to Clickhouse via Kafka.
# 6. Repeats the cycle.

def main(): 
    if gotovo():  # Check if scrapers are ready
        try:
            print('Ready')
            time.sleep(1)
            while True:
                big_pause = publish_timing()  # Send scraping timing
                print(f'Sent timing at {datetime.now()}')
                print('Started listening')
                pinn, olimp = listen()  # Receive and process data
                send_dataframe_to_kafka(pinn)  # Send Pinnacle comparison results to Kafka
                send_dataframe_to_kafka(olimp)  # Send Olimpbet comparison results to Kafka
                print(f'Starting sleep at {datetime.now()}')
                while time.time() < big_pause + 10:  # Wait for the next scraping cycle
                    time.sleep(1)
                print(f'Finished sleeping at {datetime.now()}')
        except KeyboardInterrupt:
            print('Stopping...')   
    else:
        raise ValueError('Not ready')

# Start the script if run directly
if __name__ == '__main__':
    main()
```

</details>  
<br>

---
<a id="ibet-VNC"></a>
## ~~~ 3. VNC ~~~
--- 

Websites use Google Captcha approximately once per day.
Bypassing it is impossible; the only way is to solve it manually once a day.
For this, we use VNC.

We install XFCE4 as the desktop environment, Gnome Icon for icons, and TightVNC for display.
RealVNC is used to connect to the remote desktop.
Additionally, we create a browser profile.

<details>
  <summary><strong>🖼️ VNC </strong></summary>
  
  ![installation](https://raw.githubusercontent.com/sazhirom/images/refs/heads/main/ibet/vnc_server_install.PNG)
  ![start VNC](https://github.com/sazhirom/images/blob/main/ibet/VNCserverstarted.PNG?raw=true)
  ![VNC to localhost](https://github.com/sazhirom/images/blob/main/ibet/localhost_vnc.PNG?raw=true)
  ![Real VNC](https://github.com/sazhirom/images/blob/main/ibet/connectin_real_vnc.PNG?raw=true)
  ![remote desktop](https://github.com/sazhirom/images/blob/main/ibet/remote_desktop_vnc.PNG?raw=true)
</details>  

<br>

---
<a id="ibet-redis"></a>
## ~~~ 4. Redis ~~~
--- 
Setting up Redis is a straightforward process. In our case, Redis acts as a fast message broker.
We do not require delivery guarantees, but speed and low resource consumption are essential.

Steps:  
Install Redis via apt, then configure redis.conf  
Disable protection mode.  
Set a password.  
Choose a memory management policy.  
Bind Redis to a specific IP address.  
Define the maximum memory usage for messages.  
Done!  

Since our workload is low, any eviction policy would work, but logically, we choose FIFO using allkeys-lru.

<details>
  <summary><strong>🖼️ Redis </strong></summary>
  
  ![settings](https://github.com/sazhirom/images/blob/main/ibet/redis%20setting%20right%20screen.PNG?raw=true)
  ![redis bind](https://github.com/sazhirom/images/blob/main/ibet/redis_bind.PNG?raw=true)

</details>  
<br>
<br>


## ~~~ 5. Kafka ~~~
--- 
Kafka is much more complex. The key here is to follow the documentation step by step.

Steps:
Install and extract Kafka using tar.
Create a cluster and record its ID.
Format the log directory with a special command.
Start the server.
The most interesting part:
Configuring two servers to work together using KRaft.

There aren’t many examples of this setup (almost none), so here’s my configuration:

```properties
process.roles=broker,controller
node.id=1
listener.security.protocol=PLAINTEXT
listener.security.protocol.map=BROKER:PLAINTEXT, CONTROLLER:PLAINTEXT
listeners=BROKER://10.140.0.7:9092,CONTROLLER://10.140.0.7:9093
advertised.listeners=BROKER://10.140.0.7:9092
log.retention.hours=24
num.partitions=1
default.replication.factor=1
log.segment.bytes=1073741824
log.retention.bytes=1073741824
kafka.cluster.id=L-bJf_UEQ6ioPQQhc-IZPg
log.dirs=/opt/kafka/logs
controller.quorum.voters=1@10.140.0.7:9093,2@10.140.0.2:9093
controller.listener.names=CONTROLLER
inter.broker.listener.name=BROKER
```

Each server, thanks to KRaft, can act as both a message broker and a controller for metadata management.

Key points:
We create an alias for both the Broker and Controller.  

In this setup, they run on the same server but use different ports.  

This is a minimal configuration, meaning there is no partitioning or replication.  

The log directory is also set here, and it must have write permissions for Kafka (or you need to change the owner to kafka).  

Security & Networking:
listener.security.protocol.map=BROKER:PLAINTEXT, CONTROLLER:PLAINTEXT sets the encryption protocol. Since we don’t use encryption, it's just PLAINTEXT.  

advertised.listeners defines the address where the server can be reached.  

The quorum includes both nodes.
For the second node, the settings are almost the same, but you need to change the IP and node number.

<br>
<br>

---
<a id="ibet-clickhouse"></a>
## ~~~ 6. ClickHouse ~~~
---   

Setting up and configuring ClickHouse for stable operation on a server with 2 cores and 4GB of RAM is quite a challenge.  
There are no ready-made examples, and most guides do not regulate CPU threads or core usage properly.  

Key Configuration Strategy:  
The main idea is to strictly limit CPU core and memory usage while ensuring that all settings remain consistent:  

Reduce the number of active threads  
Limit memory consumption  
Adjust resource allocation to match server capabilities  
This approach helps prevent overloading and ensures smooth performance even on a low-resource machine.  

```xml
<!-- Concurrent quries -->
<max_concurrent_queries>16</max_concurrent_queries>

<max_server_memory_usage>3221225472</max_server_memory_usage>
<max_thread_pool_size>10000</max_thread_pool_size>

<!-- THREADS -->
<background_buffer_flush_schedule_pool_size>1</background_buffer_flush_schedule_pool_size>
<background_pool_size>2</background_pool_size>
<background_merges_mutations_concurrency_ratio>1</background_merges_mutations_concurrency_ratio>
<background_merges_mutations_scheduling_policy>round_robin</background_merges_mutations_scheduling_policy>
<background_move_pool_size>1</background_move_pool_size>
<background_fetches_pool_size>1</background_fetches_pool_size>
<background_common_pool_size>2</background_common_pool_size>
<background_schedule_pool_size>1</background_schedule_pool_size>
<background_message_broker_schedule_pool_size>1</background_message_broker_schedule_pool_size>
<background_distributed_schedule_pool_size>1</background_distributed_schedule_pool_size>

<!-- Tables loading-->
<tables_loader_foreground_pool_size>0</tables_loader_foreground_pool_size>
<tables_loader_background_pool_size>0</tables_loader_background_pool_size>
<async_load_databases>false</async_load_databases>

<max_server_memory_usage_to_ram_ratio>1</max_server_memory_usage_to_ram_ratio>

<!-- Data storage-->
<storage_configuration>
    <disks>
        <default>
            <keep_free_space_bytes>1073741824</keep_free_space_bytes>
        </default>
    </disks>
</storage_configuration>
```

⚠️ WARNING: These ClickHouse Settings Will Cause Errors!  

  ![Конфликт с настройками MergeTree](https://github.com/sazhiromru/images/blob/main/ibet/clickhouse_error.PNG?raw=true)  
  
Issue:  
There is a conflict between reducing CPU cores and the default MergeTree settings.  

Solution:  
We need to analyze the logs, understand the exact cause, and check the documentation for MergeTree. Some default MergeTree settings depend on core availability, and they must be explicitly adjusted when limiting CPU usage.  

```xml
    <merge_tree>
        <max_suspicious_broken_parts>5</max_suspicious_broken_parts>
        <number_of_free_entries_in_pool_to_execute_mutation>1</number_of_free_entries_in_pool_to_execute_mutation>
        <number_of_free_entries_in_pool_to_execute_optimize_entire_partition>1</number_of_free_entries_in_pool_to_execute_optimize_entire_partition>
        <number_of_free_entries_in_pool_to_lower_max_size_of_merge>1</number_of_free_entries_in_pool_to_lower_max_size_of_merge>
    </merge_tree>
```
Kafka Integration in ClickHouse  
For storing data via Kafka, ClickHouse requires three tables:  

Table for receiving data via KafkaEngine  
Buffer table (intermediate storage)  
Main storage table with MergeTree or another engine  

```sql
CREATE TABLE kafka
(
    Event String,
    Category String,
    Subcategory String,
    Stavka String,
    F_T String,
    Coef1 Float64,
    Coef2 Float64,
    Platform String,
    Ratio Float64,
    Timestamp Float64,
    Timestamp_2 Float64
) ENGINE = Kafka
(
    '10.140.0.7:9092',
    'ibet',
    'group1',
    'JSONEachRow'
);
```

- Material view for data transfer
```sql

CREATE MATERIALIZED VIEW saver TO stavki AS
SELECT
    Event AS event,
    Category AS category,
    Subcategory AS subcategory,
    Stavka AS stavka,
    F_T AS f_t,
    Coef1 AS coef1,
    Coef2 AS coef2,
    Platform AS platform,
    Ratio AS ratio,
    toDateTime(CAST(Timestamp AS UInt32)) AS timestamp,
    toDateTime(CAST(Timestamp_2 AS UInt32)) AS timestamp_2,
    toInt32(toDateTime(CAST(Timestamp AS UInt32)) - toDateTime(CAST(Timestamp_2 AS UInt32))) AS time_delta
FROM kafka;
```
- Final table
```sql
CREATE TABLE stavki (
    event String,
    category String,
    subcategory String,
    stavka String,
    f_t String,
    coef1 Float64,
    coef2 Float64,
    platform String,
    ratio Float64,
    timestamp DateTime,
    timestamp_2 DateTime,
    time_delta Int32 
 ) ENGINE = MergeTree()
ORDER BY timestamp;
```
<br>
<br>

---
<a id="ibet-postgres"></a>
## ~~~ 7. Postgres ~~~
---   
To ensure stable operation, Airflow should use PostgreSQL instead of the default database.  

⚠️ Even with minimal load, the default Airflow database fails!  
With the default setup, even two DAGs can cause performance issues, and Airflow Scheduler constantly loses connection.  

Steps to configure PostgreSQL for Airflow:  
Install PostgreSQL  
Create a database and table for Airflow  
Add an airflow user  
Grant permissions to the airflow user to insert records  
Modify the Airflow config (airflow.cfg) to use PostgreSQL instead of SQLite  

<details>
  <summary><strong>🖼️ Postgres </strong></summary>
  
  ![settings](https://github.com/sazhirom/images/blob/main/ibet/postgre%20airflow-setting.PNG?raw=true)
  ![airflow config](https://github.com/sazhirom/images/blob/main/ibet/postgre_airflowsetup.PNG?raw=true)

</details>  

To allow local users (including Airflow) to connect without a password, you need to adjust the pg_hba.conf file  

<details>
  <summary><strong>🖼️ Postgres conf </strong></summary>
  
  ![conf](https://github.com/sazhirom/images/blob/main/ibet/postgres-airflow.PNG?raw=true)

</details>
<br>
<br>


---
<a id="ibet-airflow"></a>
## ~~~ 8. Airflow ~~~
---  
Airflow requires careful attention to documentation, much like Kafka. It can be unpredictable, with a dedicated section for bugs and workarounds.  

For example, Bash commands may not work over SSH, and after hours of debugging, the solution turns out to be adding a space after the command—because... just because. And it works.  

Step-by-Step Setup:  
✅ Connect PostgreSQL as the backend (with LocalExecutor)  
✅ Create an Airflow user & disable example DAGs  
✅ Set up an Airflow service  
✅ Reduce scheduler_heartbeat_sec in config (lowers CPU load, actually helps!)  
✅ Install SSH connection support & configure all connections  
✅ Ensure correct permissions for log & temp directories  
✅ Create all necessary DAG files & configure schedules  


<details>
  <summary><strong>🖼️ Airflow </strong></summary>
  
  ![installation](https://github.com/sazhirom/images/blob/main/ibet/airflow_installation.PNG?raw=true)
  ![postgres](https://github.com/sazhirom/images/blob/main/ibet/postgre_airflowsetup.PNG?raw=true)
  ![user](https://github.com/sazhirom/images/blob/main/ibet/airflow_disable%20examples.PNG?raw=true)
  ![service](https://github.com/sazhirom/images/blob/main/ibet/airflow_temp_solution.PNG?raw=true)
  ![conf](https://github.com/sazhirom/images/blob/main/ibet/airflow_heartbeat.PNG?raw=true)
  ![log problem](https://github.com/sazhirom/images/blob/main/ibet/airflow-private%20temp.PNG?raw=true)
  ![dags](https://github.com/sazhirom/images/blob/main/ibet/airflow-dags.PNG?raw=true)

</details>  

<br>
<br>

---

<a id="ibet-bash"></a>
## ~~~ 9. GCP and BASH ~~~
---  
Configuring SSH keys and Google CLI  
Disabling inheritance in Windows  
Using chown to allow programs to write logs  
Setting up Firewall for proper SSH login and enabling Grafana & Airflow Webserver  
Editing visudo so Airflow can run Python without sudo  
Adding environment variables in .bashrc to ensure Kafka and Airflow work properly  

<details>
  <summary><strong>🖼️ Bash </strong></summary>
  
  ![ssh](https://github.com/sazhirom/images/blob/main/ibet/bash_key_add.PNG?raw=true)
  ![google cli](https://github.com/sazhirom/images/blob/main/ibet/google1.PNG?raw=true)
  ![редактирование ключа](https://github.com/sazhirom/images/blob/main/heritage_disable.PNG?raw=true)
  ![logs](https://github.com/sazhirom/images/blob/main/ibet/logs_dir.PNG?raw=true)
  ![service](https://github.com/sazhirom/images/blob/main/ibet/service.PNG?raw=true)
  ![firewal](https://github.com/sazhirom/images/blob/main/ibet/firewall.PNG?raw=true)
  ![visudo](https://github.com/sazhirom/images/blob/main/ibet/airflow_visudo.PNG?raw=true)
  ![bashrc](https://github.com/sazhirom/images/blob/main/ibet/bashrc.PNG?raw=true)

</details>  

<br>
<br>

---

<a id="ibet-grafana"></a>

## ~~~ 10. Grafana ~~~
---  

There's not much to showcase about Grafana. It’s a very unique system, quite different from Metabase and even more so from Power BI.  

Configure all visualizations  
Enable dynamic color changes for multiple parameters using override  
Publish dashboards via share  
<details>
  <summary><strong>🖼️ Grafana </strong></summary>
  
  ![grafana](https://github.com/sazhirom/images/blob/main/ibet/grafana%20overrides.PNG?raw=true)


</details>  
Example Above  
In the created table:  

Instead of a numerical column, create a text-based one using CASE  
Add color changes using override  
Apply mapping for better visualization  

