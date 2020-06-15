# NREL Wave Hindacast Data Access Example

#### This notebook demonstrates basic usage of the National Renewable Energy Laboratory (NREL) West Coast Wave Hindcast dataset. The data is provided from Amazon Web Services using the HDF Group's Highly Scalable Data Service (HSDS).

#### For this to work you must first install h5pyd:

pip install --user h5pyd

#### Next you'll need to configure HSDS:

hsconfigure

#### and enter at the prompt:

hs_endpoint = https://developer.nrel.gov/api/hsds

hs_username = None

hs_password = None

hs_api_key = 3K3JQbjZmWctY0xmIfSYvYgtIcM3CN0cb1Y2w9bf

#### The example API key here is for demonstation and is rate-limited per IP. To get your own API key, visit https://developer.nrel.gov/signup/

#### You can also add the above contents to a configuration file at ~/.hscfg



```python
import h5pyd
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

from pyproj import Proj
from scipy.spatial import cKDTree
```

### Basic Usage

The Wave Hindcast Dataset is provided in annual .h5 files and currently spans the Western Coastal Region of the Lower 48 from 1979-2010.

Each year can be accessed from /nrel/us-wave/US_Wave_${year}.h5


```python
# Open the desired year of Wave Hindcast data
# server endpoint, username, password is found via a config file

waves = h5pyd.File(f'/nrel/us-wave/US_Wave_1990.h5','r')
```


```python
list(waves.attrs) # list of base attributes of the dataset
```




    ['ref_IEC62600-101',
     'ref_SWAN-Manual',
     'ref_Wu-Wang-Yang-Garcia-Medina-2020',
     'source',
     'version']



## Datasets

What follows are basic examples of navigating the dataset and discovering variables



```python
# List the available dataset variable names within the dataset

print(list(waves['coordinates'].attrs))
```

    ['description', 'dimensions', 'src_name', 'units']



```python
# Locational information is stored in either 'meta' or 'coordinates'

meta = pd.DataFrame(waves['meta'][:])
meta.head()
```


```python
# Shapes of datasets
waves['significant_water_height'].shape # (time, lon_lat)
```

## Time Slicing

Basic description of timestamp extraction


```python
# Extract datetime index for datasets (Saved as binary)

time_index = pd.to_datetime(waves['time_index'][:].astype(str))
time_index, time_index.inferred_freq # Temporal resolution is 3 hours
```


```python
Aug_7th_midnight = time_index.get_loc('1990-08-07 00:00:00') #precise data returns single index
Aug = time_index.get_loc('1990-08') #Inprecise will return a range a values
```


```python
print(f'August 7th midnight: {time_index[Aug_7th_midnight]}')
print(f'Month of August: {time_index[Aug]}')
```

## Mapping Data

Basic methods to quickly plot data from the Hindcast spatially and temporally

### Spatial Plots


```python
# the view the available coordinates which are provided as lat,lon pairs

coord_slice = 100

print(f'Available attributes: {dict(waves["coordinates"].attrs)}')
coords = waves['coordinates'][::coord_slice]
print(f'Coordinate pairs: {coords}')
```


```python
# Building a scatter plot of significant wave height values

df = pd.DataFrame() # Combine data with coordinates in a DataFrame
df['longitude'] = coords[:, 1]
df['latitude'] = coords[:, 0]
df['Hsig'] = waves['significant_water_height'][Aug_7th_midnight, ::coord_slice] 
```


```python
%matplotlib widget

df['Hsig'][df['Hsig']<0] = np.nan # There are negative values here Levi
df.plot.scatter(x='longitude', y='latitude', c='Hsig',
                colormap='viridis',
                title=str(time_index[Aug_7th_midnight]))
plt.show()
```

### Timeseries


```python
# Extracting a timeseries usually requires us to find a location of interest.
# This can be done easially with KDTree

tree = cKDTree(coords)
def nearest_site(tree, lat_coord, lon_coord):
    lat_lon = np.array([lat_coord, lon_coord])
    dist, pos = tree.query(lat_lon)
    return pos

lat_coord, lon_coord = 39.6122, -125.318
site_idx = nearest_site(tree, lat_coord, lon_coord)

print(f'Site index: {site_idx}')
print(f'Coordinates Searched: {lat_coord, lon_coord}')
print(f'Coordinates of Nearest Point: {coords[site_idx]}')
```


```python
# Now we can extract data at that location over a specified time range

var = 'peak_period'

dt = pd.DataFrame(waves[var][Aug,site_idx],index=time_index[Aug],columns=[var])
dt.plot(y=var,title=f'Time sliced {var} Timeseries')
```

## Basic Statistics and Grouping


```python
# Here is an example of grouping data to create quick plots of stats

Vars =['significant_water_height','peak_period'] 
variables = np.array([waves[Vars[0]][Aug,site_idx], waves[Vars[1]][Aug,site_idx]])

ds = pd.DataFrame(variables.T,
                    index=time_index[Aug],
                    columns=Vars
                )
group = ds.groupby('peak_period')
```


```python
# Plot the number of occurances of sea-states within peak period bins

group.count().plot(kind='bar')
```


```python
# plot the mean Hsig of the sea-states within peak period bins along with
# the variance of those waveheights

group.mean().plot(kind='bar',yerr=group.var())
```


```python

```
