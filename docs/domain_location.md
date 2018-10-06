# Mapping Physical Locations of Servers
// Reported on **September 20th, 2018**<br/>
// Status: **Complete w/ plans to Convert to Service**

This simple lab displays all foreign domains on a website and creates [`geojson`](https://geojson.io) that shows the physical locations of servers. This yields some pretty fascinating results.

## Example: [UW CSE website](https://courses.cs.washington.edu/courses/cse143/18sp/)
![](https://imgur.com/6WQmeoE.png)
Source: [gist](https://gist.github.com/qwergram/175f285a88c381397f213d060097ca65)

## Example: [hackernews](https://news.ycombinator.com/)
![](https://imgur.com/2MKXJHA.png)
Source: [gist](https://gist.github.com/qwergram/e3abbfa3fd382733166a039f965db243)

## Example: [uw.edu](https://uw.edu)
See below

## GeoJSON format:
```json
{
  "type": "Feature",
  "properties": {},
  "geometry": {
    "type": "Point",
    "coordinates": [
      longitude,
      latitude
    ]
  }
}
```


```python
import socket
import requests
import json
from urllib.parse import urlparse
from bs4 import BeautifulSoup
```


```python
def get_foreign_domains(target):
    response = requests.get(target)
    soup = BeautifulSoup(response.text, "lxml")
    links = [item['href'] for item in soup.find_all("a", href=True)]
    foreign_links = filter(lambda x: x.startswith("http"), links)
    return [urlparse(link).netloc for link in foreign_links]

def get_geojson(domain, source_coord):
    ip = socket.gethostbyname(domain)
    response = requests.get("https://data.pengra.io/geoip/{}/".format(ip))
    data = response.json()
    if source_coord['longitude'] != data['longitude'] and source_coord['latitude'] != data['latitude']:
        return [{
            "type": "Feature",
            "properties": {
                "Name": data['IP']
            },
            "geometry": {
                "type": "LineString",
                "coordinates": [
                    [
                        source_coord['longitude'],
                        source_coord['latitude']
                    ],
                    [
                        data['longitude'],
                        data['latitude']
                    ]
                ]
            }
        },
        {
            "type": "Feature",
            "properties": {
                "Name": data['IP']
            },
            "geometry": {
                "type": "Point",
                "coordinates": [
                    data['longitude'],
                    data['latitude']
                ]
            }
        }
        ]
    
    return []
```


```python
TARGET = "https://uw.edu"
domains = get_foreign_domains(TARGET)
source_coord = requests.get("https://data.pengra.io/geoip/{}/".format(socket.gethostbyname(urlparse(TARGET).netloc))).json()

geo_json = []
for domain in domains:
    for data_set in get_geojson(domain, source_coord):
        geo_json.append(data_set)
        
geo_json.append({
    "type": "Feature",
    "properties": {
        "Name": source_coord['IP']
    },
    "geometry": {
        "type": "Point",
        "coordinates": [
            source_coord['longitude'],
            source_coord['latitude']
        ]
    }
})
        
print(json.dumps({"type": "FeatureCollection", "features": geo_json[:50]}, indent=2))
```

    {
      "type": "FeatureCollection",
      "features": [
        {
          "type": "Feature",
          "properties": {
            "Name": "69.91.245.67"
          },
          "geometry": {
            "type": "LineString",
            "coordinates": [
              [
                -121.8034,
                47.4323
              ],
              [
                -122.2919,
                47.6606
              ]
            ]
          }
        },
        {
          "type": "Feature",
          "properties": {
            "Name": "69.91.245.67"
          },
          "geometry": {
            "type": "Point",
            "coordinates": [
              -122.2919,
              47.6606
            ]
          }
        },
        ... 48 more items
      ]
    }


Results are limited to 50 because the gist won't render any more features above that.
![](https://imgur.com/RXNuCPS.png)
Source: [gist](https://gist.github.com/qwergram/9b8655059613f51459b176998e035121)
