# datadog_extraction_python
To extract data from datadog requires api keys for authentications and the queries which you want to run in Datadog for extraction. Below steps cover this in step by step format:


### Step 1: Initiate the required libraries and set up your credentials for making the API calls:

```python
from datadog import initialize, api
from datetime import datetime
import time
import pandas as pd
import dateutil.relativedelta

options = {
    'api_key': 'API key from Datadog',
    'app_key': 'APP key from Datadog',
    'api_host': 'https://api.datadoghq.com'
}

initialize(**options)
```

### Step 2: Specify the parameters for time-series data extraction. 
##### Below we’re setting the time period from Thursday, May 20, 2021 12:00:00 AM to Wednesday, June 2, 2021 11:59:00 PM:

```python
# Specify the time period in epoch format over which you want to fetch the data. You can use https://www.epochconverter.com/ to get the timestamps
start = 1669133509  # Tuesday, November 22, 2022 16:11:49 GMT
end = 1669392709  # Friday, 25 Nov 2022 16:11:49 GMT

# run datadog metric collection for every 600 sec (10 min) window. 
# This will remove the granularity issue discussed in the blog. 

time_delta = 600 

# query we want to run in datadog for data extraction
query = 'avg:system.cpu.user{*}' 
```

One thing to keep in mind is that **Datadog has a rate limit** based on [organization policies](https://docs.datadoghq.com/api/latest/rate-limits/). In case you face rate issues, try increasing the “time_delta” in the query above to reduce the number of requests you make to the Datadog API.

### Step 3: Run the extraction logic. 
##### Take the start and the stop timestamp and split them into buckets of width = time_delta.

```python
#Extraction Logic :
# 1. take the datadog query 
# 2. pass the query to datadog api with a time span of time_delta milliseconds -> This would pull data in spans of T to T + time_delta
# 3. append this data to a pandas dataframe

def extract(query, start_at, end_at, time_delta):
    
    print('Running extraction for cpu utilization')
    
    '''
    datadog return data as a json blob with a field called 'series' that stores a list called 'pointlist' which has metrics like :
    * timestamp of the point for which the metric was extracted for.
    * value of the field at this timestamp.
    '''
    
    #initialise data lists : 
    
    datetime_list = []
    value_list = []
    metric_list = []
    
    # start reading till this timeframe and then increment timedelta window in the loop
    iteration_end = start_at + time_delta 
    
    while (iteration_end <= end_at):

        results = api.Metric.query(start=start_at, end=iteration_end, query=query)

        for datadog_result in results['series']:
            for time_value_pair_list in datadog_result['pointlist']:
                
                converted_datetime = datetime.fromtimestamp(time_value_pair_list[0]/1000)
                
                datetime_list.append(converted_datetime)
                value_list.append(time_value_pair_list[1])
                metric_list.append(datadog_result['metric']) # store the query that was executed in datadog.
         
        # increment the time spans 
        start_at = iteration_end # change start time as end of last iteration
        iteration_end = iteration_end + time_delta # increment reading frame
        
        #logging logic to see extraction progress
        #print('finished extracting data from ', datetime.fromtimestamp(start_at).strftime('%c') , ' to ', datetime.fromtimestamp(iteration_end).strftime('%c'))


    all_data = {
        'datetime': datetime_list,
        'value': value_list,
        'metric': metric_list
    }

    print('Finished extraction for cpu utilization')
    return pd.DataFrame.from_dict(all_data)
    
#extract data into a dataframe    
data_cpu_metrics = extract(query, start, end, time_delta)
```

### Step 4: Data is available in data_cpu_metrics dataframe for analysis

```python 
print(data_cpu_metrics.head(10))


import seaborn as sns
import numpy as np
import matplotlib.pyplot as plt

data_cpu_metrics['cpu_usage_metric'] = data_cpu_metrics['value'].astype(float)

fig, ax = plt.subplots(figsize=(15, 10))

sns.kdeplot(data_cpu_metrics['cpu_usage_metric'], ax = ax, bw= 1)
plt.xlabel('CPU utilization')
plt.ylabel('Density')
plt.title('Visualizing distribution of CPU utilization')
ax.get_legend().remove()
```

<img width="822" alt="image" src="https://user-images.githubusercontent.com/32186649/204024863-4caa080c-3f3f-4eea-9b75-c7063a9ced52.png">