# COMP47670 - Assignment 1 - Task 1

## Investigating Earthquakes in Hawaii between 2014 and 2023

**Faith Whyte** - 
*This project is for educational purposes only.*


According to the [USGS](https://www.usgs.gov/observatories/hvo/science/about-earthquakes-hawaii#:~:text=The%20earthquake%20record%20since%201823,one%20M5%20or%20greater%20earthquake.), thousands of earthquakes occur every year in Hawaii, some of which are destructive. The aim of this project is to investigate earthquakes felt by people in Hawaii over the course of a decade. 

This will be acheived by retrieving data from the [USGS Earthquake Catalog API](https://earthquake.usgs.gov/fdsnws/event/1/) under specific conditions, engineering the raw data, performing exploratory analysis, visualising the results, and extracting insights about earthquakes in Hawaii between 2014 and 2023.


The scope of this project limits the collection of data to events with positive 'felt' values, indicating that at least one person has subitted a record through the [*Did You Feel It?*](https://earthquake.usgs.gov/data/dyfi/) system. Geographic coordinates (21.3099°, -157.8581°) denote the midpoint of Hawaii. This midpoint along with a search radius of 1,250km filters the data to earthquakes events across all islands of Hawaii.

This Jupyter Notebook is the first of two for this project and is includes the steps for collecting and storing data from the API.

## Housekeeping

### Importing the necessary libraries 


```python
# File management:

from pathlib import Path
import os

# Data retrieval: 

import requests
import urllib
import urllib.request as ulrq
import urllib.parse as urlparse

# json management:

import json 

# Dataframe management:

import pandas as pd
```

### Setting the working directory


```python
# Create a folder "Earthquake Project" if there is not one already. 

wd = Path("Earthquake Project")
wd.mkdir(parents=True, exist_ok=True)

# Set "Earthquake Project" as the working directory.

os.chdir(wd)

print("Working directory is set. Response and raw data will save to folder: %s." % os.path.basename(os.getcwd()))
```

    Working directory is set. Response and raw data will save to folder: Earthquake Project.
    

## Data Collection

### Retrieving raw data from the API 

The response variable is defined by the endpoint, which is the where the data request will be sent, and params, which define the search queries. 
The data will be fetched as a GeoJSON file, which is similar to a JSON file but includes geospatial information. 


```python
# Define the function that will retrieve the data from the API, setting defaults for the format and limit.
# Since we are querying parameters, we must add the query method to the url, to allow the assignment of parameters. 

def fetch(api_url, params = {'format': 'geojson', 'limit' : '100'}): 
    
    # Add query method to the url if not already present.
    
    if not api_url.endswith("query?"): 
        api_url += "query?" 
     
    response = requests.get(api_url, params=params)
    
    # Add parameters to the url.
    
    if api_url.endswith("query?"):
        endpoint = api_url + urlparse.urlencode(params)
        
    print("Fetching earthquake data from: %s" % (endpoint))

    # If the the status is good, then a positive message is produced and the raw data is written to a JSON file
    
    if response.status_code == 200:
        data = response.json()
        print("Data has been successfully fetched!")
    
        out_path = "earthquakes_raw.json"
        print("Writing the raw data to %s\%s." % (wd, out_path))
        fout = open(out_path, "w")
        json.dump(data, fout, indent=4, sort_keys=True)
        fout.close()
        
    # If the status signifies a fail, a message will be produced to reflect that. 
    
    else:
        print("Failed to fetch data. Response status code: %s" %(response.status_code))

```

Now, the data can be retrieved from the API.


```python
# Define variables that represent the API url and the query parameters. 


url = 'https://earthquake.usgs.gov/fdsnws/event/1/'


queries = { 'format': 'geojson',
            'minfelt' : 1,
            'starttime': '2014-01-01',
            'endtime': '2024-01-01', 
            'latitude': 21.3099,
            'longitude': -157.8581,
            'maxradiuskm': 1250 }


# Run the function with the url and queries variable.

fetch(url, queries)
```

    Fetching earthquake data from: https://earthquake.usgs.gov/fdsnws/event/1/query?format=geojson&minfelt=1&starttime=2014-01-01&endtime=2024-01-01&latitude=21.3099&longitude=-157.8581&maxradiuskm=1250
    Data has been successfully fetched!
    Writing the raw data to Earthquake Project\earthquakes_raw.json.
    

## Data Storage

### Assessing the format of the data

A dataframe must be created to hold the data. Next, an attempt to convert the data into a Pandas DataFrame is conducted. The following block of code signals whether the data is in the correct format for conversion.


```python
try: 
    data = pd.read_json('earthquakes_raw.json')
    print("Data is readable.")
except:
    print("Data is unreadable. JSON file needs to be formatted.")
```

    Data is unreadable. JSON file needs to be formatted.
    

Since the data is not ready for conversion from JSON to a Pandas DataFrame, the keys must be assessed to determine an appropriate method formatting the data.


```python
# Retrieve the keys.

with open('earthquakes_raw.json', 'r') as file:
    data = json.load(file)
    
# Iterate over the keys and produce a list. 

print("The keys is the raw earthquake data are:")
for keys in data.keys():
    print("- %s" % keys)
```

    The keys is the raw earthquake data are:
    - bbox
    - features
    - metadata
    - type
    

### Normalising the JSON, creating a dataframe and assessing the contents

The key of interest is "features" as it contains the details of each earthquake event. This will be used to normalise the data so that it can be converted into the dataframe that will be used for Task 2 and 3.


```python
# Normalise the data by the "features" key. 

df = pd.json_normalize(data["features"])
```

It is now possible to find the number of entries that have been retrieved from the API as well as a list of the variables included.


```python
# Print the number of entries.

print("There are %d entries in the dataframe prior to data engineering." % len(df))
```

    There are 4507 entries in the dataframe prior to data engineering.
    


```python
# Look at what variables are included.

df.columns
```




    Index(['id', 'type', 'geometry.coordinates', 'geometry.type',
           'properties.alert', 'properties.cdi', 'properties.code',
           'properties.detail', 'properties.dmin', 'properties.felt',
           'properties.gap', 'properties.ids', 'properties.mag',
           'properties.magType', 'properties.mmi', 'properties.net',
           'properties.nst', 'properties.place', 'properties.rms',
           'properties.sig', 'properties.sources', 'properties.status',
           'properties.time', 'properties.title', 'properties.tsunami',
           'properties.type', 'properties.types', 'properties.tz',
           'properties.updated', 'properties.url'],
          dtype='object')



### Modifying the variable names

The variable names are lengthy and there multiple "type" variables. In order to make the dataframe more readible, some variables will be renamed, the prefixes "geometry" and "properties" will be removed and the column names will be capitalised.


```python
# Rename the "type" variables.

df = df.rename(columns={"geometry.type": "Geometry Type", "type": "Data Type", "properties.type": "Seismic Type"})

# Create a dataframe with the prefixes dropped.

no_prefix = df.columns.str.removeprefix("properties.").str.removeprefix("geometry.")

# Capitalise the columns of this dataframe.

capitalised_cols = no_prefix.str.capitalize()

# Assign the modified column names to the original dataframe.

df.columns = capitalised_cols

# Assign the "Code" variable as the index.

df.set_index("Code", inplace=True)
```

Then the top 5 entries of the dataframe can be accessed to verify the column names have been successfully updated.


```python
# View the first 5 entries of the dataframe.

df.head(5)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Id</th>
      <th>Data type</th>
      <th>Coordinates</th>
      <th>Geometry type</th>
      <th>Alert</th>
      <th>Cdi</th>
      <th>Detail</th>
      <th>Dmin</th>
      <th>Felt</th>
      <th>Gap</th>
      <th>...</th>
      <th>Sources</th>
      <th>Status</th>
      <th>Time</th>
      <th>Title</th>
      <th>Tsunami</th>
      <th>Seismic type</th>
      <th>Types</th>
      <th>Tz</th>
      <th>Updated</th>
      <th>Url</th>
    </tr>
    <tr>
      <th>Code</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>73704697</th>
      <td>hv73704697</td>
      <td>Feature</td>
      <td>[-155.246666666667, 19.3865, 1.35]</td>
      <td>Point</td>
      <td>None</td>
      <td>2.7</td>
      <td>https://earthquake.usgs.gov/fdsnws/event/1/que...</td>
      <td>NaN</td>
      <td>10</td>
      <td>45.0</td>
      <td>...</td>
      <td>,hv,us,</td>
      <td>reviewed</td>
      <td>1704012869020</td>
      <td>M 2.9 - 6 km SSW of Volcano, Hawaii</td>
      <td>0</td>
      <td>earthquake</td>
      <td>,dyfi,origin,phase-data,</td>
      <td>None</td>
      <td>1709415574040</td>
      <td>https://earthquake.usgs.gov/earthquakes/eventp...</td>
    </tr>
    <tr>
      <th>73704627</th>
      <td>hv73704627</td>
      <td>Feature</td>
      <td>[-155.503666666667, 20.0086666666667, 13.04]</td>
      <td>Point</td>
      <td>None</td>
      <td>2.7</td>
      <td>https://earthquake.usgs.gov/fdsnws/event/1/que...</td>
      <td>NaN</td>
      <td>3</td>
      <td>157.0</td>
      <td>...</td>
      <td>,us,hv,</td>
      <td>reviewed</td>
      <td>1704008857340</td>
      <td>M 2.4 - 8 km SSW of Honoka‘a, Hawaii</td>
      <td>0</td>
      <td>earthquake</td>
      <td>,dyfi,origin,phase-data,</td>
      <td>None</td>
      <td>1709415574040</td>
      <td>https://earthquake.usgs.gov/earthquakes/eventp...</td>
    </tr>
    <tr>
      <th>73700837</th>
      <td>hv73700837</td>
      <td>Feature</td>
      <td>[-155.460833333333, 19.1943333333333, 31.4]</td>
      <td>Point</td>
      <td>None</td>
      <td>2.2</td>
      <td>https://earthquake.usgs.gov/fdsnws/event/1/que...</td>
      <td>NaN</td>
      <td>1</td>
      <td>146.0</td>
      <td>...</td>
      <td>,hv,us,</td>
      <td>reviewed</td>
      <td>1703895268370</td>
      <td>M 2.5 - 2 km ESE of Pāhala, Hawaii</td>
      <td>0</td>
      <td>earthquake</td>
      <td>,dyfi,origin,phase-data,</td>
      <td>None</td>
      <td>1709415573040</td>
      <td>https://earthquake.usgs.gov/earthquakes/eventp...</td>
    </tr>
    <tr>
      <th>73700682</th>
      <td>hv73700682</td>
      <td>Feature</td>
      <td>[-155.747, 20.0081666666667, 6.77]</td>
      <td>Point</td>
      <td>None</td>
      <td>3.1</td>
      <td>https://earthquake.usgs.gov/fdsnws/event/1/que...</td>
      <td>NaN</td>
      <td>3</td>
      <td>105.0</td>
      <td>...</td>
      <td>,hv,us,</td>
      <td>reviewed</td>
      <td>1703885666380</td>
      <td>M 2.4 - 7 km WSW of Waimea, Hawaii</td>
      <td>0</td>
      <td>earthquake</td>
      <td>,dyfi,origin,phase-data,</td>
      <td>None</td>
      <td>1709415570040</td>
      <td>https://earthquake.usgs.gov/earthquakes/eventp...</td>
    </tr>
    <tr>
      <th>73700537</th>
      <td>hv73700537</td>
      <td>Feature</td>
      <td>[-155.285995483398, 19.4120006561279, 1.210000...</td>
      <td>Point</td>
      <td>None</td>
      <td>2.0</td>
      <td>https://earthquake.usgs.gov/fdsnws/event/1/que...</td>
      <td>NaN</td>
      <td>1</td>
      <td>41.0</td>
      <td>...</td>
      <td>,hv,</td>
      <td>automatic</td>
      <td>1703871751150</td>
      <td>M 2.2 - 6 km WSW of Volcano, Hawaii</td>
      <td>0</td>
      <td>earthquake</td>
      <td>,dyfi,origin,phase-data,</td>
      <td>None</td>
      <td>1703889225828</td>
      <td>https://earthquake.usgs.gov/earthquakes/eventp...</td>
    </tr>
  </tbody>
</table>
<p>5 rows × 29 columns</p>
</div>



### Storing the dataset

The dataset is now better prepared for cleaning in the next task. It can be serialised and saved to the working directory for access in the second notebook. 


```python
# Use the to_pickle function to serialise the data to the .pkl file "Earthquake Dataset" 
# This will save it to the working directory.

df.to_pickle("Earthquake Dataset")
```
