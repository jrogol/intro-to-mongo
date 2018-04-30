# Week 3 (Through April 21)

## Indexes in Movies (Parts 1 and 2)
- App makes several queries to database, i.e. clicking on a movie for details
  - `get_movie` method in `db.py` - works off of `_id`
  - Or the search functionality
  - indices in the `movies` collection are the key.
    - Can make queries faster.
- `.index_information()` method has *four* available indexes.
  - `_id` and 3 user-defined indices.
  - lists keys available, the `1` or `-1` indicates ascending or descending order.
  - Can be combinations of values, across multiple fields (a **compound index**)
- More indexes mean faster reads
  - but slower writes
  - Maximum of 64 indices per collection
- Queries occur in stages (the `explain` command reveals the details)
  - i.e. `COLLSCAN`, which searches every column
  - `winningPlan` reveals if an inex was used.
- `title_text_cast_text_directors_text` is another index
  - Text indices don't follow the exact match
  - Rather, how relevant is a key to the entries in an index.
- `$find` looks for *exact* matches
  - but a `$text` search in a filter is more of a fuzzy match
    - case insensitve, or relevance (stemming, stopwords, etc.)
      - but can only specify one per collection in mongoDB
- `.create_index()` method takes an array of tuples. The field to be indexed, and the order inwhich they are ordered (or a text or geometric index.)

``` python
import pymongo
import pprint

# Replace XXXX with your connection URI from the Atlas UI
uri = 'localhost:27017'
mc = pymongo.MongoClient(uri)
mflix = mc.mflix

# List Indexes
# get list of indexes on movies collection
pprint.pprint(mflix.movies.index_information())

pprint.pprint(mflix.movies.find_one())

# Explain a Query

explain = {
    "explain": {
        "find": "movies",
        "filter": {
            "tomatoes.viewer.numReviews": {"$gt": 10}
        },
    },
    "verbosity": "executionStats"
}
mflix.command(explain)

# Text Search vs Exact Match
filters = {"title": "Titanic"}
for m in mflix.movies.find(filters):
    pprint.pprint(m['title'])

filters = { "$text": {
    "$search": "titanic"
}}
for m in mflix.movies.find(filters):
    pprint.pprint(m['title'])
    pprint.pprint(m['cast'])
    pprint.pprint(m.get('directors', ""))
    pprint.pprint("======")

# Create an Index
mflix.movies.create_index([("countries", pymongo.ASCENDING)])

```
## Geospatial queries
- Built in geographic queries!
- In `mflix` there's the `theaters` collection
  - contains geolocation of several theaters, alongside human-readable addresses.
  - geoJSON format (i.e. 'point')
- Enterprise version of Compass can analyze the schema, generating a map. A search area can then be produced, generating a query!
  -`$geoWithin` operator can be used to search a particular field (`location.geo`, as it's embedded)
    - two parameters, longitude, then latitude (backwards from tradition!) in an array
    - Then radius of a circle, in RADIANS
    - `$box`, `$polygon`, `$centerSphere` and `$geometry` work similarly.
- `$nearSphere` is also similar, but returns documents in the order from the center point.
  - `$geometry` is used to identify the center, and `$minDistance` and `$maxDistance` to set the distance parameters
  - the **requires** a geospatial index to work
-  In Compass, clicking Indexes > Create Index, and chosing `2dsphere` on the appropriate field will create this.


``` python
import pymongo
import pprint

# Replace XXXX with your connection URI from the Atlas UI
course_cluster_uri = 'localhost:27017'

course_client = pymongo.MongoClient(course_cluster_uri)
theaters = course_client['mflix']['theaters']

theater = theaters.find_one({})

pprint.pprint(theater)

pprint.pprint(theater['location']['geo'])

EARTH_RADIUS_MILES = 3963.2
EARTH_RADIUS_KILOMETERS = 6378.1

example_radius = 0.1747728893434987

radius_in_miles = example_radius * EARTH_RADIUS_MILES

print(radius_in_miles)

query = {
  "location.geo": {
    "$nearSphere": {
      "$geometry": {
        "type": "Point",
        "coordinates": [-73.9899604, 40.7575067]
      },
      "$minDistance": 0,
      "$maxDistance": 1000
    }
  }
}

for theater in theaters.find(query):
    pprint.pprint(theater)

```
### More information
- [Geospatial Query Operators](https://docs.mongodb.com/manual/reference/operator/query-geospatial?jmp=coursera-intro-mongod)

## Graphing with MongoDB

- `matplotlib` anticipates data in arrays.
- `plt.clf()` clears the plot.
``` python
import matplotlib.pyplot as plt
%matplotlib inline


a = [1, 2, 3, 4, 5]
b = [x ** 2 for x in a]

print(a, b)

plt.clf()
​
fig, ax = plt.subplots()
​
ax.scatter(a, b)
​
plt.scatter(a,b)

import pymongo
import pprint

# Replace XXXX with your connection URI from the Atlas UI
course_cluster_uri = 'localhost:27017'

course_client = pymongo.MongoClient(course_cluster_uri)
movies = course_client['mflix']['movies']

query = {
  "runtime": { "$exists": True },
  "metacritic": { "$exists": True }
  }
​
projection = {
  "_id": 0,
  "runtime": 1,
  "metacritic": 1
  }

rm = list(movies.find(query, projection))

pprint.pprint(rm[0])

runtimes = [movie['runtime'] for movie in rm]

print(runtimes)

metacritic_ratings = [movie['metacritic'] for movie in rm]

plt.clf()
fig, ax = plt.subplots()
ax.scatter(runtimes, metacritic_ratings, alpha=0.5)
plt.title("Metacritic Movie Ratings vs. Movie Runtime")
plt.xlabel('Movie Runtime (minutes)')
plt.ylabel('Movie Rating (metacritic)')
plt.show()



from mpl_toolkits.mplot3d import Axes3D

query = {
  "runtime": { "$exists": True },
  "metacritic": { "$exists": True },
  "year": { "$exists": True }
  }
​
projection = {
  "_id": 0,
  "runtime": 1,
  "metacritic": 1,
  "year": 1
  }

rmy = list(movies.find(query, projection))

runtimes = [movie['runtime'] for movie in rmy]
metacritic_ratings = [movie['metacritic'] for movie in rmy]
years = [movie['year'] for movie in rmy]

plt.clf()

fig = plt.figure()
ax = fig.add_subplot(111, projection='3d')​
ax.scatter(runtimes, metacritic_ratings, years)

plt.title('Movie Ratings vs. Runtime vs. Year')
ax.set_xlabel('Movie Runtime (minutes)')
ax.set_ylabel('Movie Rating (metacritic)')
ax.set_zlabel('Movie Year')

plt.show()

client = pymongo.MongoClient("mongodb://buildapp-student:buildapp-password@cluster0-shard-00-00-jxeqq.mongodb.net:27017,cluster0-shard-00-01-jxeqq.mongodb.net:27017,cluster0-shard-00-02-jxeqq.mongodb.net:27017/?ssl=true&replicaSet=Cluster0-shard-0&authSource=admin")
pings = client['mflix']['watching_pings']

pings.find_one()

cursor = pings.aggregate([
  {
    "$sample": { "size": 50000 }
  },
  {
    "$addFields": {
      "dayOfWeek": { "$dayOfWeek": "$ts" },
      "hourOfDay": { "$hour": "$ts" }
    }
  },
  {
    "$group": { "_id": "$dayOfWeek", "pings": { "$push": "$$ROOT" } }
  },
  {
    "$sort": { "_id": 1 }
  }
  ]);

pings_by_day = [doc['pings'] for doc in cursor]

pings_by_hour_by_day = [[ping['hourOfDay'] for ping in pings] for pings in pings_by_day]

plt.clf()
​
fig, ax = plt.subplots()
ax.boxplot(pings_by_hour_by_day)
ax.set_title('When People Watch Movies')
ax.yaxis.grid(True)
ax.set_xticklabels(['Sun', 'Mon', 'Tues', 'Wed', 'Thur', 'Fri', 'Sat'])
ax.set_xlabel('Day of Week')
ax.set_ylabel('Hour of Day')
plt.show()
```
