# NREL Wave Hindacast Data Access Example

#### This notebook demonstrates basic usage of the National Renewable Energy Laboratory (NREL) West Coast Wave Hindcast dataset. The data is provided from Amazon Web Services using the HDF Group's Highly Scalable Data Service (HSDS).

#### For this example to work, the packages necessary can be installed via pip:

pip install -r requirements.txt

#### Next you'll need to configure h5pyd to access the data on HSDS:

hsconfigure

#### and enter at the prompt:

hs_endpoint = https://developer.nrel.gov/api/hsds   
hs_username = None   
hs_password = None   
hs_api_key = 3K3JQbjZmWctY0xmIfSYvYgtIcM3CN0cb1Y2w9bf    

#### The example API key here is for demonstation and is rate-limited per IP. To get your own API key, visit https://developer.nrel.gov/signup/

#### You can also add the above contents to a configuration file at ~/.hscfg

#### Finally, you can use Jupyter Notebook or Lab to view the example notebooks depending on your preference



```python
import h5pyd
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
```

### Basic Usage

The Wave Hindcast Dataset is provided in annual .h5 files and currently spans the Western Coastal Region of the Lower 48 from 1979-2010.

Each year can be accessed from /nrel/us-wave/US_Wave_${year}.h5

To open the desired year of Wave Hindcast data server endpoint, username, password is found via a config file


```python
waves = h5pyd.File(f'/nrel/us-wave/US_Wave_1990.h5','r')
```


```python
list(waves) # list of base attributes of the dataset
```

## Datasets

Each dataset quantity:

- directionality_coefficient
- energy_period
- maximum_energy_direction
- mean_absolute_period
- mean_zero-crossing_period
- omni-directional_wave_power
- peak_period
- significant_wave_height
- spectral_width
- water_depth

is structured as ('time_index','coordinate')



```python
# Shapes of datasets
waves['significant_water_height'].shape # (time_index, coordinates)
```


```python
# List the available dataset variable names within the dataset
print(list(waves['coordinates'].attrs))
```


```python
# Locational information is stored in either 'meta' or 'coordinates'

meta = pd.DataFrame(waves['meta'][:])
meta.head()
```

# Basic Usage

The following examples illustrate basic spatial and timeseries based slicing techniques for the West Coast Wave Hindcast dataset, along with simple statistical analysis. 

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

Coordinate pair indices are used to slice the datasets, and are accessed through the 'coordinates' attribute


```python
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

### Timeseries Plots

Extracting a timeseries usually requires us to find a location of interest.      
This can be done easially with KDTree, a function within scipy.spatial.   



```python
from scipy.spatial import cKDTree

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

Here is an example of grouping data to create quick plots of statistics


```python
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
