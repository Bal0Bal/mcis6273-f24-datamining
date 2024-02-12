Home > People > Arpan54321
Arpan54321
import requests
import json
import os
import pandas as pd
import time

def fetch_noaa_data(api_key):
    base_url = "https://www.ncei.noaa.gov/cdo-web/api/v2/data"
    dataset_id = "GHCND"
    location_id = "ZIP:80249"
    date_ranges = [
        ("2008-12-15", "2009-01-21"),
        ("2009-12-15", "2010-01-21"),
        ("2010-12-15", "2011-01-21"),
        ("2011-12-15", "2012-01-21"),
        ("2012-12-15", "2013-01-21"),
        ("2013-12-15", "2014-01-21"),
        ("2014-12-15", "2015-01-21"),
        ("2015-12-15", "2016-01-21"),
        ("2016-12-15", "2017-01-21"),
        ("2017-12-15", "2018-01-21"),
        ("2018-12-15", "2019-01-21"),
        ("2019-12-15", "2020-01-21"),
        ("2020-12-15", "2021-01-21"),
        ("2021-12-15", "2022-01-21")
    ]
    
    # Create 'data' directory if it does not exist
    if not os.path.exists('data'):
        os.makedirs('data')

    for start_date, end_date in date_ranges:
        params = {
            "datasetid": dataset_id,
            "locationid": location_id,
            "startdate": start_date,
            "enddate": end_date,
            "units": "standard",
            "limit": 1000,
        }
        headers = {"token": api_key}  # Include API key in headers
        
        retry_count = 0
        max_retries = 3
        delay = 1  # Initial delay in seconds
        
        while retry_count < max_retries:
            try:
                response = requests.get(base_url, params=params, headers=headers)
                response.raise_for_status()  # Raise an exception for HTTP errors
                data = response.json()
                with open(f"data/winter_{start_date.split('-')[0]}-{end_date.split('-')[0]}.json", "w") as f:
                    json.dump(data, f)
                break  # Exit loop if request succeeds
            except requests.exceptions.HTTPError as err:
                print(f"HTTP error occurred: {err}")
            except json.JSONDecodeError as err:
                print(f"Error decoding JSON response: {err}")
            except Exception as err:
                print(f"An unexpected error occurred: {err}")
            
            # Exponential backoff
            delay *= 2 ** retry_count
            print(f"Retrying in {delay} seconds...")
            time.sleep(delay)
            
            retry_count += 1

def json_to_csv():
    # Combine all JSON files into a single DataFrame
    df_list = []
    for year in range(2008, 2023):
        file_name = f"data/winter_{year}-{year + 1}.json"
        with open(file_name) as f:
            data = json.load(f)
            df = pd.json_normalize(data["results"])
            df_list.append(df)
    final_df = pd.concat(df_list)
    
    # Extract required columns and calculate TAVG
    final_df = final_df[["date", "datatype", "value"]]
    final_df["date"] = pd.to_datetime(final_df["date"])
    final_df = final_df.pivot(index="date", columns="datatype", values="value")
    final_df["TAVG"] = (final_df["TMAX"] + final_df["TMIN"]) / 2
    
    # Export DataFrame to CSV
    final_df.to_csv("data/all_data_max_min_avg.csv")

def compile_data():
    # Combine all data into a single DataFrame
    compiled_data = {}
    for year in range(2008, 2023):
        file_name = f"data/winter_{year}-{year + 1}.json"
        with open(file_name) as f:
            data = json.load(f)
            for result in data["results"]:
                date = result["date"].split("T")[0]
                if date not in compiled_data:
                    compiled_data[date] = {}
                compiled_data[date][f"{year}-{year+1}"] = result["value"]
    
    # Convert compiled_data to DataFrame
    df = pd.DataFrame.from_dict(compiled_data, orient="index")
    df.index = pd.to_datetime(df.index)
    df.index.name = "Date"
    
    # Export DataFrame to CSV
    df.to_csv("data/all_data_min.csv")

def average(df):
    return df.mean()

def average_warmest(df):
    return df["TMIN"].mean()

def average_coldest(df):
    return df["TMAX"].mean()

# Call the functions with your API key
api_key = "aQxLGxOGvBgqVedWxLBqsuxrzQnPRhSb"
fetch_noaa_data(api_key)
json_to_csv()
compile_data()
