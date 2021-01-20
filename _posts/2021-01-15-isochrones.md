---
layout: post
title: Isochrones in Stockholm
hidden: false
---

<p align="center">
    <img width="475" src="/img/isochrones/header_code2.png">
</p>

Suppose that you are about to relocate. An important factor in deciding where to is the
time it takes to travel to work (well, at least in non-pandemic times). One proxy for this
is of course *distance* -- move to a place that is close to your workplace. However, it is
clearly not a perfect one: the time to travel from one location to another depends heavily
on the available means of transport, the location of roads and public transport, etc.

One way to illustrate this is by drawing
[isochrones](https://en.wikipedia.org/wiki/Isochrone_map) on a map: 

> An *isochrone* (iso = equal, chrone = time) is defined as "a line drawn on a
> map connecting points at which something occurs or arrives at the same time".

However, since Stockholm is far from a flat field -- it is situated on [14
islands](https://en.wikipedia.org/wiki/Geography_of_Stockholm) -- the determination of
isochrones (and, in turn, the relocation problem) is somewhat non-trivial.

 A quick Google search shows that there are several services available that can plot
isochrones, but, as far as I can tell, most of these are based on transportation by
bicycle or car (e.g., [RouteWare](https://www.routeware.dk/) and
[Iso4App](http://www.iso4app.net/)). The ones that includes travel by train or public
transport are either specific to larger cities in Europe (e.g., [Tom
Carden](http://www.tom-carden.co.uk/p5/tube_map_travel_times/applet/) and
[Mapumental](https://www.mysociety.org/2007/03/05/more-travel-time-maps-and-their-uses/)
for London, [TravelTime](https://traveltime.com/),
[Mapnificent](https://www.mapnificent.net/) and [Targomo](https://www.targomo.com/)) or
operate on a much coarser scale than the inner city of Stockholm (e.g., [Empty
Pipes](http://emptypipes.org/supp/isochrone_stockholm/) and
[CityMonitor.ai](https://citymonitor.ai/transport/these-isochrone-maps-show-how-well-or-how-badly-europe-s-cities-are-connected-train-1174))
.


As a fun weekend project, let us see if we can put together our own solution
that allows us to *plot a travel-time map (isochrones) of Stockholm*.

<!--more-->
---

## Time to travel from A to B

Public transport (metro, buses, trams) in the inner city of Stockholm is handled
by [SL](https://sl.se/). On their website, you can get the best route
(along with an estimate of the travel time) between any two locations in
Stockholm. Thanks to the people at [Trafiklab](https://www.trafiklab.se/), it is
possible to access this programmatically through an API.

As a first step, we register at Trafiklab and request an API-key to the [SL
Reseplanerare 3.1](https://www.trafiklab.se/api/sl-reseplanerare-31) API. I have
saved the key in a text-file called `sl_api_key.txt`:


```python
with open('sl_api_key.txt') as f:
    api_key = f.read()
```

The API is documented [here](https://www.trafiklab.se/node/25593/documentation).
Our first goal will be to write a function that returns the travel time between
any two coordinates in Stockholm (once we have this, we can query this function
over a grid).

### Querying the API

A query to the
[*Trip*-API](https://www.trafiklab.se/node/25593/documentation#_Ref402159801)
requires us to specify the API-key, an origin and a destination (either as
a station name, or with latitude and longitude coordinates). We can optionally
specify a date and time for the trip (since we are optimizing travel times to
and from work, we probably want to specify a weekday around rush hour) and if we
want to leave or arrive by that time (`searchForArrival`). We can also specify
how many trip suggestions that we want with `numF` and `numB` -- for us, just
one is sufficient. The query returns [JSON](https://en.wikipedia.org/wiki/JSON)
data with each trip suggestion, and the details on each "segment" (`Leg`) of the
trip.

The first segment is usually a walk to the nearest station, the intermediate
segments different modes of transportation (metro, bus, etc.), and the last
segment a walk from a final stop to the destination's coordinates.

For some reason, the `duration`-field of some segments is sometimes not included
in the returned data, which causes an incorrect estimate of the total duration
of the trip. As a workaround for this, we can manually compute the travel time
by subtracting the departure time from the arrival time. The departure time is
specified in the `Date` and `Time` fields of the first segment of the trip, and
the arrival time in the `Date` and `Time` fields of the last segment. These are
given as strings, so let us first make a helper function to convert these
strings to a `datetime` object -- this will make it easier to compute the
time-difference (i.e., travel time).


```python
from datetime import datetime

def strs_to_datetime(date_str, time_str):
    return datetime.strptime(date_str + ' ' + time_str, '%Y-%m-%d %H:%M:%S')

strs_to_datetime('2021-01-15', '13:30:00')
```




    datetime.datetime(2021, 1, 15, 13, 30)



In the function below, we construct a query for the API, parse the returned data
(compute the travel time manually) and return the travel time (in minutes)
between two coordinates. We use Monday 18 January 2021 at 08:00 as our arrival
time and we only ask for one trip suggestion.


```python
import requests

def get_travel_time_between(origin_lat, origin_long, dest_lat, dest_long,
                            verbose = False):

    params = {
        'key': api_key,
        'originCoordLat': origin_lat,
        'originCoordLong': origin_long,
        'destCoordLat': dest_lat,
        'destCoordLong': dest_long,
        'date': '2021-01-18',
        'time': '08:00',
        'searchForArrival': 1, # arrive(1)/depart(0) by specified date and time
        'numF': 0,             # trip suggestions after specified date and time
        'numB': 0              # trip suggestions before specified  -- "" --
    }

    response = requests.get('http://api.sl.se/api2/TravelplannerV3_1/trip.json',
                            params=params)

    # make sure data was retreived successfully
    if not response.status_code == 200:
        if verbose:
            print('ERROR ({}) in retrieving time between ({}, {}) and ({}, '
                  '{})!'.format(response.status_code,
                                origin_lat,
                                origin_long,
                                dest_lat,
                                dest_long)
                  )
        return -1

    json_data = response.json()

    if not 'Trip' in json_data:
        if verbose:
            print('\t\tERROR: \'Trip\' not in returned json-data!')
        return -1

    if len(json_data['Trip']) > 1:
        if verbose:
            print('\t\tWARNING: Received more than 1 trip suggestion!')

    # the returned trip-duration is sometimes incorrectly computed by
    # the API, so let's compute it manually as the time-delta between
    # the first and the last segment of the trip
    first_segment = json_data['Trip'][0]['LegList']['Leg'][0]
    last_segment  = json_data['Trip'][0]['LegList']['Leg'][-1]

    start_time = strs_to_datetime(first_segment['Origin']['date'],
                                  first_segment['Origin']['time'])

    end_time = strs_to_datetime(last_segment['Destination']['date'],
                                last_segment['Destination']['time'])

    return (end_time - start_time).total_seconds() / 60

```

Let us see if it works. Consider, for example, the travel time between KTH and
Liljeholmen (which, I know should be roughly half an hour). The
coordinates can be obtained using [Google Maps](https://maps.google.com).


```python
kth_lat  = 59.3486016
kth_long = 18.0690764

liljeholmen_lat  = 59.3113696
liljeholmen_long = 18.0236446

get_travel_time_between(kth_lat, kth_long, liljeholmen_lat, liljeholmen_long)
```




    31.0



So, 31 minutes -- that sounds about right!

## Travel times within Stockholm

Now that we can compute the travel time between any two coordinates, let us do it
over a grid ontop of Stockholm. This will then allow us to estimate
isochrones (as the [level sets](https://en.wikipedia.org/wiki/Level_set) of
the data).

With the free API, we are
[allowed](https://www.trafiklab.se/api/sl-reseplanerare-31) to do 10,000 queries
per month, and a maximum of 30 per minute. This translates to one query every
other second -- in the code below, we `sleep` a bit longer (2.1 seconds) for
some extra margin (28 queries per minute). We let the grid span over a rectangle
from
[Barkaby](https://www.google.se/maps/place/Barkaby/@59.4085823,17.8542054,12z/)
in the north-west, down to
[Älta](https://www.google.se/maps/place/%C3%84lta/@59.2563638,18.1744499,12z/)
in the south-east. This, roughly, covers central Stockholm.


```python
import pandas as pd
import numpy as np
import time

# Barkaby
sthlm_lat_start  = 59.4085823
sthlm_long_start = 17.8542054

# Älta
sthlm_lat_stop  = 59.2563638
sthlm_long_stop = 18.1744499

def get_travel_times_to(dest_lat, dest_long):

    # number of gridpoints
    n_grid_lat  = 40
    n_grid_long = 40

    print('Will do {} API queries...'.format(n_grid_lat * n_grid_long))
    print('Expected time to finish: {:.1f} m\n'.format(n_grid_lat *
                                                       n_grid_long * 2.1 / 60))
    travel_times = []
    for lat in np.linspace(sthlm_lat_start, sthlm_lat_stop, n_grid_lat):
        print('\tLatitude: {}'.format(lat))
        for long in np.linspace(sthlm_long_start, sthlm_long_stop, n_grid_long):
            travel_times.append(
                {'latitude': lat,
                 'longitude': long,
                 'time': get_travel_time_between(lat, long,
                                                 dest_lat, dest_long)
                 }
            )
            # rate-limit is 30 queries / minute --> 1 query / 2 seconds
            time.sleep(2.1)

    print('\nDone downloading data.')
    return pd.DataFrame(travel_times)
```

Suppose we work at [KTH](https://www.kth.se) (at the [Division of Decision and
Control Systems](https://www.kth.se/is/dcs/) :) with `lat = 59.349725, long
= 18.0660567`. Then we can download all travel times from points on the grid by
calling the function we defined above:


```python
work_lat  = 59.349725
work_long = 18.0660567

travel_df = get_travel_times_to(work_lat, work_long)
```

    Will do 1600 API queries...
    Expected time to finish: 56.0 m
    
    	Latitude: 59.4085823 
    	Latitude: 59.40467926153846 
    	Latitude: 59.400776223076925 
    	Latitude: 59.39687318461539 
    	Latitude: 59.39297014615384 
    	Latitude: 59.389067107692306 
    	Latitude: 59.38516406923077 
    	Latitude: 59.38126103076923 
    	Latitude: 59.377357992307694 
    	Latitude: 59.37345495384616 
    	Latitude: 59.36955191538461 
    	Latitude: 59.365648876923075 
    	Latitude: 59.36174583846154 
    	Latitude: 59.3578428 
    	Latitude: 59.35393976153846 
    	Latitude: 59.350036723076926 
    	Latitude: 59.34613368461539 
    	Latitude: 59.342230646153844 
    	Latitude: 59.33832760769231 
    	Latitude: 59.33442456923077 
    	Latitude: 59.33052153076923 
    	Latitude: 59.326618492307695 
    	Latitude: 59.32271545384616 
    	Latitude: 59.31881241538461 
    	Latitude: 59.314909376923076 
    	Latitude: 59.31100633846154 
    	Latitude: 59.3071033 
    	Latitude: 59.303200261538464 
    	Latitude: 59.29929722307693 
    	Latitude: 59.29539418461539 
    	Latitude: 59.291491146153845 
    	Latitude: 59.28758810769231 
    	Latitude: 59.28368506923077 
    	Latitude: 59.27978203076923 
    	Latitude: 59.275878992307696 
    	Latitude: 59.27197595384616 
    	Latitude: 59.268072915384614 
    	Latitude: 59.26416987692308 
    	Latitude: 59.26026683846154 
    	Latitude: 59.2563638 
    
    Done downloading data.


So what did we just download?


```python
travel_df.describe()
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
      <th>latitude</th>
      <th>longitude</th>
      <th>time</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>count</th>
      <td>1600.000000</td>
      <td>1600.000000</td>
      <td>1600.000000</td>
    </tr>
    <tr>
      <th>mean</th>
      <td>59.332473</td>
      <td>18.014328</td>
      <td>52.476250</td>
    </tr>
    <tr>
      <th>std</th>
      <td>0.045068</td>
      <td>0.094817</td>
      <td>16.056636</td>
    </tr>
    <tr>
      <th>min</th>
      <td>59.256364</td>
      <td>17.854205</td>
      <td>-1.000000</td>
    </tr>
    <tr>
      <th>25%</th>
      <td>59.294418</td>
      <td>17.934267</td>
      <td>43.000000</td>
    </tr>
    <tr>
      <th>50%</th>
      <td>59.332473</td>
      <td>18.014328</td>
      <td>52.000000</td>
    </tr>
    <tr>
      <th>75%</th>
      <td>59.370528</td>
      <td>18.094389</td>
      <td>60.000000</td>
    </tr>
    <tr>
      <th>max</th>
      <td>59.408582</td>
      <td>18.174450</td>
      <td>123.000000</td>
    </tr>
  </tbody>
</table>
</div>



Or, if we visualize the last column (the travel times from different locations in
Stockholm to our workplace):


```python
import matplotlib
travel_df['time'].plot.hist(bins=45)
```

![png](/img/isochrones/histogram.png)


As can be seen from the table and the graph above, there were a few queries for which the
API returned an error (and, hence, `time = -1`). For now, let us just drop these and leave
it to future work to investigate why.


```python
travel_df = travel_df[travel_df['time'] != -1]
travel_df.shape
```




    (1588, 3)



So (1600 - 1588) = 12 entries were removed -- not that many.

## Plotting travel times

Now that we have the data (coordinates and the associated travel times to our workplace),
let us try to visualize it in some useful way. 

We can plot the data directly using, for example,
[Matplotlib](https://matplotlib.org/mpl_toolkits/mplot3d/tutorial.html):


```python
import matplotlib.pyplot as plt

fig = plt.figure(figsize = (7,7), dpi = 90)
ax = fig.add_subplot(111, projection = '3d')

ax.scatter(travel_df['latitude'],
           travel_df['longitude'],
           travel_df['time'],
           c = travel_df['time'],
           s = 8)

ax.set_xlabel('latitude')
ax.set_ylabel('longitude')
ax.set_zlabel('travel time [min]')
```

![png](/img/isochrones/scatter_times2.png)

Interesting -- but, perhaps it would be easier to interpret if we rotated the viewport to
coincide with how a map is normally drawn (north pointing up, east to the right) and
changed the colormap?

```python
fig = plt.figure(figsize = (7,7), dpi = 90)
ax = fig.add_subplot(111, projection = '3d')

ax.scatter(travel_df['latitude'],
           travel_df['longitude'],
           travel_df['time'],
           c = travel_df['time'],
           s = 8,
           cmap = matplotlib.cm.jet)

# rotate viewport
ax.view_init(90, 0)
ax.invert_xaxis()

ax.set_zticklabels([])

ax.set_xlabel('latitude')
ax.set_ylabel('longitude')
```

![png](/img/isochrones/scatter_heatmap_small.png)

Slightly better. But there is still some crucial information missing for it to be
informative -- at least I do not have a very good intuition for what latitude and
longitude correspond to where in Stockholm.


### Plotting on a map

Instead, we would like to use something with specialized map-plotting functionality.
A quick Google search yields a number of Python-libraries that provide such functionality:
[Matplotlib Basemap](https://matplotlib.org/basemap/),
[Folium](https://python-visualization.github.io/folium/),
[Plotly](https://plotly.com/python/maps/), etc. 

Below, I use [Folium](https://python-visualization.github.io/folium/) since it has easy to
use support for drawing nice looking
[heatmaps](https://python-visualization.github.io/folium/plugins.html#folium.plugins.HeatMap)
(which is essentially what we plotted above when we rotated the viewport).


```python
import folium
from folium.plugins import HeatMap

def plot_time_map(additional_markers = [], data = travel_df):
    map = folium.Map(
        location = [(sthlm_lat_start + sthlm_lat_stop) / 2,
                    (sthlm_long_start + sthlm_long_stop) / 2],
        tiles='Stamen Toner',
        zoom_start=11
    )

    marker_str = 'Work'
    folium.Marker([work_lat, work_long],
                  popup=marker_str,
                  tooltip=marker_str).add_to(map)

    for marker in additional_markers:
        folium.Marker([marker['latitude'],
                       marker['longitude']],
                       popup=marker['string'],
                       icon=folium.Icon(color='red'),
                       tooltip=marker['string']).add_to(map)


    max_time = data['time'].max()

    HeatMap(data.to_numpy(),
            min_opacity=0.1,
            max_val=max_time,
            radius=11,
            blur=11,
            max_zoom=10
            ).add_to(map)

    return map
```

Then we can plot the travel-time data on a map of Stockholm with:


```python
plot_time_map()
```


![png](/img/isochrones/work_times_plot.png)

Note that our workplace (i.e., KTH) has been marked with a blue marker.

Cool. So what can we do with this now?

## Distance vs travel time

Consider, for example, Ropsten (north-east of KTH) and Liljeholmen (south-west of KTH):


```python
ropsten_lat  = 59.357298
ropsten_long = 18.1000293

plot_time_map([{'latitude': ropsten_lat,
                'longitude': ropsten_long,
                'string': 'Ropsten'},
               {'latitude': liljeholmen_lat,
                'longitude': liljeholmen_long,
                'string': 'Liljeholmen'}])
```


![png](/img/isochrones/liljeholmen_ropsten.png)


Even though Liljeholmen is more than twice as far away from KTH as Ropsten is
(as measured by the bird path, in meters):


```python
from geopy import distance

print('Distance to')
print('\tRopsten:\t', end='')
print(distance.distance((work_lat, work_long),
                        (ropsten_lat, ropsten_long)))

print('\tLiljeholmen:\t', end='')
print(distance.distance((work_lat, work_long),
                        (liljeholmen_lat, liljeholmen_long)))
```

    Distance to
    	Ropsten:	2.1086527827170345 km
    	Liljeholmen:	4.907707524415635 km


The travel time (using public transport) is almost the same:


```python
print('Travel time from')
print('\tRopsten: ', end='')
print('\t{:.0f} min'.format(get_travel_time_between(ropsten_lat,
                                                    ropsten_long,
                                                    work_lat,
                                                    work_long)))

print('\tLiljeholmen: ', end='')
print('\t{:.0f} min'.format(get_travel_time_between(liljeholmen_lat,
                                                    liljeholmen_long,
                                                    work_lat,
                                                    work_long)))
```

    Travel time from
    	Ropsten: 	28 min
    	Liljeholmen: 	33 min


The reason is that to go [from Ropsten, one has to first take the metro to
Östermalmstorg and there change
line](https://sl.se/?mode=travelPlanner&origName=Ropsten+%28Stockholm%29&origLat=59.35730&origLon=18.10003&destLat=59.34860&destLon=18.06908&destName=KTH&transportTypes=79),
whereas [from Liljeholmen it is straight with no
changes](https://sl.se/?mode=travelPlanner&origName=Liljeholmen+%28Stockholm%29&origLat=59.31137&origLon=18.02364&destLat=59.34860&destLon=18.06908&destName=KTH&transportTypes=79).

In other words, just because something is closer on the map does not mean it is
"closer in time".

## Isochrones

So what about the isochrones? They can be read from the earlier plots. However,
we can make them more explicit by plotting only the locations from which our workplace
can be reached within a certain number of minutes.


```python
def plot_isochrone(time):
    isochrone = travel_df[travel_df['time'] <= time]

    map = folium.Map(
        location = [(sthlm_lat_start + sthlm_lat_stop) / 2,
                    (sthlm_long_start + sthlm_long_stop) / 2],
        tiles='Stamen Toner',
        zoom_start=11
    )

    marker_str = 'Work'
    folium.Marker([work_lat, work_long],
                  popup=marker_str,
                  tooltip=marker_str).add_to(map)

    HeatMap(isochrone[['latitude', 'longitude']].to_numpy(),
            min_opacity=0.5,
            max_val=time,
            radius=11,
            blur=11,
            max_zoom=10,
            gradient={1.0:'#00AA00', 0.0:'#00FF00'}
            ).add_to(map)

    return map
```

### 25-minute isochrone

Hence, all the locations from which KTH can be reached within 25 minutes on
a Monday morning are:


```python
plot_isochrone(25)
```


![png](/img/isochrones/25m_isochrone.png)


### 35-minute isochrone

Similarly, all the locations from which it can be reached within 35 minutes are:


```python
plot_isochrone(35)
```


![png](/img/isochrones/35m_isochrone.png)


Depending on our threshold for travel time to work, these plots give us a good indication
of areas that would be suitable for relocating to.

## More than one point of interest

Suppose now that there is more than one point of interest. You might have some
location-specific hobby which you would also like to minimize travel time to, or your
partner might want to live close to work as well. How can we find the locations from which
both `work_lat, work_long` and `hobby_lat, hobby_long` can be reached within some given
`time`-threshold  (that is, what are the locations that should be considered for
relocation)?

We can solve this by intersecting the isochrones for the two destinations.
First, let us download travel-time data for the location of the hobby (here,
I use
[Eriksdalsbadet](https://motionera.stockholm/trana-gymma-simma/eriksdalsbadet/)
as an example):


```python
hobby_lat  = 59.3048988
hobby_long = 18.0733449

hobby_df = get_travel_times_to(hobby_lat, hobby_long)

hobby_df = hobby_df[hobby_df['time'] != -1]
```

    Will do 1600 API queries...
    Expected time to finish: 56.0 m
    
    	Latitude: 59.4085823 
    	Latitude: 59.40467926153846 
    	Latitude: 59.400776223076925 
    	Latitude: 59.39687318461539 
    	Latitude: 59.39297014615384 
    	Latitude: 59.389067107692306 
    	Latitude: 59.38516406923077 
    	Latitude: 59.38126103076923 
    	Latitude: 59.377357992307694 
    	Latitude: 59.37345495384616 
    	Latitude: 59.36955191538461 
    	Latitude: 59.365648876923075 
    	Latitude: 59.36174583846154 
    	Latitude: 59.3578428 
    	Latitude: 59.35393976153846 
    	Latitude: 59.350036723076926 
    	Latitude: 59.34613368461539 
    	Latitude: 59.342230646153844 
    	Latitude: 59.33832760769231 
    	Latitude: 59.33442456923077 
    	Latitude: 59.33052153076923 
    	Latitude: 59.326618492307695 
    	Latitude: 59.32271545384616 
    	Latitude: 59.31881241538461 
    	Latitude: 59.314909376923076 
    	Latitude: 59.31100633846154 
    	Latitude: 59.3071033 
    	Latitude: 59.303200261538464 
    	Latitude: 59.29929722307693 
    	Latitude: 59.29539418461539 
    	Latitude: 59.291491146153845 
    	Latitude: 59.28758810769231 
    	Latitude: 59.28368506923077 
    	Latitude: 59.27978203076923 
    	Latitude: 59.275878992307696 
    	Latitude: 59.27197595384616 
    	Latitude: 59.268072915384614 
    	Latitude: 59.26416987692308 
    	Latitude: 59.26026683846154 
    	Latitude: 59.2563638 
    
    Done downloading data.


We can plot the travel-time map for the hobby (marked with a red marker) as follows:


```python
plot_time_map(additional_markers=[{'string': 'Hobby',
                                   'latitude': hobby_lat,
                                   'longitude': hobby_long}],
              data=hobby_df)

```


![png](/img/isochrones/hobby_times_plot.png)


In order to combine this data with the previous travel-time data, we join the two
datasets:


```python
merged_df = pd.merge(travel_df,
                     hobby_df,
                     on=['latitude', 'longitude'],
                     suffixes=['_work', '_hobby'])

merged_df.head()
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
      <th>latitude</th>
      <th>longitude</th>
      <th>time_work</th>
      <th>time_hobby</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>59.408582</td>
      <td>17.854205</td>
      <td>63.0</td>
      <td>62.0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>59.408582</td>
      <td>17.862417</td>
      <td>76.0</td>
      <td>53.0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>59.408582</td>
      <td>17.870628</td>
      <td>56.0</td>
      <td>90.0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>59.408582</td>
      <td>17.878840</td>
      <td>55.0</td>
      <td>89.0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>59.408582</td>
      <td>17.887051</td>
      <td>69.0</td>
      <td>59.0</td>
    </tr>
  </tbody>
</table>
</div>



Now we can plot the locations on the map from which we can travel to either work
or the hobby within `time` minutes (these have both `time_work <= time` and
`time_hobby <= time`):


```python
def plot_intersected_isochrone(time):
    intersected_isochrone = merged_df[(merged_df['time_work'] <= time) &
                                      (merged_df['time_hobby'] <= time)]

    map = folium.Map(
        location = [(sthlm_lat_start + sthlm_lat_stop) / 2,
                    (sthlm_long_start + sthlm_long_stop) / 2],
        tiles='Stamen Toner',
        zoom_start=11
    )

    marker_str = 'Work'
    folium.Marker([work_lat, work_long],
                  popup=marker_str,
                  tooltip=marker_str).add_to(map)

    marker_str = 'Hobby'
    folium.Marker([hobby_lat, hobby_long],
                  popup=marker_str,
                  icon=folium.Icon(color='red'),
                  tooltip=marker_str).add_to(map)


    HeatMap(intersected_isochrone[['latitude', 'longitude']].to_numpy(),
            min_opacity=0.5,
            max_val=time,
            radius=12,
            blur=11,
            max_zoom=10,
            gradient={1.0:'#00AA00', 0.0:'#00FF00'}
            ).add_to(map)

    return map
```

### Feasible locations for relocation

The locations from which both the workplace (blue) and the hobby (red) can be reached
within half an hour are:


```python
plot_intersected_isochrone(30)
```

![png](/img/isochrones/inter_isochrone_30m.png)



That essentially limits us to the very center of Stockholm -- expensive!

If we are a bit more flexible and allow for 45 minutes to either location, we
obtain many more feasible locations:


```python
plot_intersected_isochrone(45)
```


![png](/img/isochrones/inter_isochrone_45m.png)

For reference, here is a geographic map of Stockholm
with the railbound lines of public transport shown:

![png](/img/isochrones/transit_stockholm.png)

Note that it is not enough to live *on* a line; we need to live close to
a station so as to have a short walking distance before we enter the public
transport system -- hence the isolated green spots close to Farsta, Kista,
Tensta, etc. in the plot above.

## Conclusion

In this post, we considered the problem of finding suitable locations to relocate to. In
particular, we aimed to find locations from which we can travel to work -- and perhaps
also to a hobby -- within a certain amount of time using the [public transport system of
Stockholm](https://sl.se/).

To do so, we first used the open API provided by [Trafiklab](https://www.trafiklab.se/) to
access travel-time data. We then visualized the data using
[Folium](https://python-visualization.github.io/folium/). From the visualizations, we
noted that distance is *not* always a good proxy for travel time: just because two
locations are close on a map, does not mean that they are close "in time".

Finally, by combining travel-time data for both the workplace and the hobby, we obtained
an answer to the question we set out to answer: what are feasible locations to consider
for relocation (with respect to travel times)? This gives us a basis (and limits our
search) for further investigations that take other important factors into account (price,
neighborhood, etc.).

### Extensions

This post was a weekend experiment and should be read as a fun proof-of-concept --
do not go and buy an apartment just because `plot_intersected_isochrone(30)` puts a green
spot there!

There is obviously a lot that could be improved and many extensions that are possible.
Here are some ideas:

- In order to maximize the amount of information one can obtain, while respecting the
  quota on the number of queries, one could come up with smarter ways of sampling the map
(instead of the naive uniform grid).

- For example, exclude coordinates that lie in the water, since there is obviously no
  house to which you could relocate there. The API seems to return data even for such
locations -- perhaps it computes the time from closest point on land? 

- Increase the `numB` and `numF` fields in the API-query to request more than one trip
  suggestion for the query. Then compute the min/mean/median of the returned travel
times. This would help create a smoother travel-time plot. One could also compute an
average over several days and/or times during a day for a more representative value.

- Use the [Booli API](https://www.booli.se/p/api/) to check if there are any apartments
  for sale in any of the feasible locations. One could also filter for underpriced objects
(with respect to our conditions on travel times).

I hope that you enjoyed reading this post -- if you have any comments, feel free to email
me!
