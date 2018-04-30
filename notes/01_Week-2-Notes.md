# Week 2 (Through April 15)

## Projections & Aggregation framework

- Aggregation framework can do anything the query language Can
  - Has more power with regard to Projections/reshaping data
- `"movies_scratch"` collection will be used to show examples when walking through the following code.
- Pipeline limited to first 100 items (first item in pipeline)
- `$project` is used for projections on all documents passing through the stage.
  - There's talk of splitting fields into array valued fields using an add fields stage.
    - looks like `$split` is run on a specific field, and a separator is indicated, along with the name of the new fields.
  - RESHAPING
  - Include fields with a `1`, exclude them with a `0`
    - can further modify them as need be (i.e. `fullplot` to `fullPlot`)
      - Find the value of `$fullplot` key, and add it to the new kek, `$fullPlot`
    - IMDb is a new document. Simplify to a singular key, and add a dictionary of the 3 IMDb fields.
      - Not necessary, but an example of what's necessary


``` python
import pymongo
from pymongo import MongoClient
import pprint
from IPython.display import clear_output
from data.keys import mongoURI

# Replace XXXX with your connection URI from the Atlas UI
client = MongoClient(mongoURI)


pipeline = [
    {
        '$limit': 100
    },
    {
        '$project': {
            'title': 1,
            'year': 1,
            'directors': {'$split': ["$director", ", "]},
            'cast': {'$split': ["$cast", ", "]},
            'writers': {'$split': ["$writer", ", "]},
            'genres': {'$split': ["$genre", ", "]},
            'languages': {'$split': ["$language", ", "]},
            'countries': {'$split': ["$country", ", "]},
            'plot': 1,
            'fullPlot': "$fullplot",
            'rated': "$rating",
            'released': 1,
            'runtime': 1,
            'poster': 1,
            'imdb': {
                'id': "$imdbID",
                'rating': "$imdbRating",
                'votes': "$imdbVotes"
                },
            'metacritic': 1,
            'awards': 1,
            'type': 1,
            'lastUpdated': "$lastupdated"
        }
    },
    {
        '$out': "movies_scratch"
    }]

clear_output()
pprint.pprint(list(client.mflix.movies_initial.aggregate(pipeline)))
```  
- Nothing should show in the output, switch to compass to see the changes in the `movies_scratch` collection.

## Projecting Queries, Part II

- "released" is just a string, and can be converted to date.
  - But some are empty, which is why there's a `$cond` if/else operator.
    - `$cond` is a document, with three keys: if/then/else
    - 'if' uses `$ne`, or not equal to.
      - here: if $released (field) is NOT equal to "" (empty string)
    - 'then' here, parses the date using `$dateFromString` operator
    -


``` python
# This pipeline will not work on Atlas until MongoDB 3.6 has been released
# If you're testing this before 3.6 is released you can download and install MongoDB 3.5.X locally
# In that case you should use "mongodb://localhost:27017" as your connection URI
pipeline = [
    {
        '$limit': 100
    },
    {
        '$project': {
            'title': 1,
            'year': 1,
            'directors': {'$split': ["$director", ", "]},
            'actors': {'$split': ["$cast", ", "]},
            'writers': {'$split': ["$writer", ", "]},
            'genres': {'$split': ["$genre", ", "]},
            'languages': {'$split': ["$language", ", "]},
            'countries': {'$split': ["$country", ", "]},
            'plot': 1,
            'fullPlot': "$fullplot",
            'rated': "$rating",
            'released': {
                '$cond': {
                    'if': {'$ne': ["$released", ""]},
                    'then': {
                        '$dateFromString': {
                            'dateString': "$released"
                        }
                    },
                    'else': ""}},
            'runtime': 1,
            'poster': 1,
            'imdb': {
                'id': "$imdbID",
                'rating': "$imdbRating",
                'votes': "$imdbVotes"
                },
            'metacritic': 1,
            'awards': 1,
            'type': 1,
            'lastUpdated': "$lastupdated"
        }
    },
    {
        '$out': "movies_scratch"
    }]

clear_output()
pprint.pprint(list(client.mflix.movies_initial.aggregate(pipeline)))
```
- Irritatingly, `$dateFromString` is only available in Mongo 3.6+, which isn't offered in the free tier of Atlas. That's annoying.
- [Aggregation Pipeline Quick Reference](https://docs.mongodb.com/manual/meta/aggregation-quick-reference/index.html?jmp=coursera-intro-mongodb)

## Projecting Queries, Part III

- Running 3.6 locally, we can start the database by running `mongod` in the terminal.

``` python
# Import the required libraries, and set for the local instance.
import pymongo
from pymongo import MongoClient
import pprint
from IPython.display import clear_output

mongoURI = "localhost:27017"

# Replace XXXX with your connection URI from the Atlas UI
client = MongoClient(mongoURI)
print(client.mflix)
```

- The following will parse a timestamp for the "lastUpdated"
  - Includes a timezone
  - There's extraneous milliseconds there, `$dateFromString` won't work on that.
  - New stage will split on the decimal in the lastupdated field
  - `arrayElemAt` takes the array generated by the above, and selects the 0th element (seconf value, the index of the element)
  - Adds the above as a new value to every document passed through the pipeline.
  - `$addFields` will overwrite a new field if it already exists.

``` python

pipeline = [
    {
        '$limit': 100
    },
    {
        '$addFields': {
            'lastupdated': {
                '$arrayElemAt': [
                    {'$split': ["$lastupdated", "."]},
                    0
                ]}
        }
    },
    {
        '$project': {
            'title': 1,
            'year': 1,
            'directors': {'$split': ["$director", ", "]},
            'actors': {'$split': ["$cast", ", "]},
            'writers': {'$split': ["$writer", ", "]},
            'genres': {'$split': ["$genre", ", "]},
            'languages': {'$split': ["$language", ", "]},
            'countries': {'$split': ["$country", ", "]},
            'plot': 1,
            'fullPlot': "$fullplot",
            'rated': "$rating",
            'released': {
                '$cond': {
                    'if': {'$ne': ["$released", ""]},
                    'then': {
                        '$dateFromString': {
                            'dateString': "$released"
                        }
                    },
                    'else': ""}},
            'runtime': 1,
            'poster': 1,
            'imdb': {
                'id': "$imdbID",
                'rating': "$imdbRating",
                'votes': "$imdbVotes"
                },
            'metacritic': 1,
            'awards': 1,
            'type': 1,
            'lastUpdated': {
                '$cond': {
                    'if': {'$ne': ["$lastupdated", ""]},
                    'then': {
                        '$dateFromString': {
                            'dateString': "$lastupdated",
                            'timezone': "America/New_York"
                        }
                    },
                    'else': ""}}
        }
    },
    {
        '$out': "movies_scratch"
    }]

clear_output()
pprint.pprint(list(client.mflix.movies_initial.aggregate(pipeline)))

```

- [Aggregation Pipeline Quick Reference](https://docs.mongodb.com/manual/meta/aggregation-quick-reference/index.html?jmp=coursera-intro-mongodb)

## Updating Documents, Part 1

- Data cleaning can also be performed using the query language, as well as the aggregation pipeline above.
- Aggregation framework is **declarative**
  - Specifies what the end result looks like
  - Query Language is more akin to traditional scripting.
- Within the `for` loop, iterating through documents to update.
  - uses the `.update_one()` method at the collection level.
    - two parameters: a filter to select the document to update_doc
      - `_id` is a unique ID, and what's iterated over in the `for` loop.
      - `update_doc` is a dictionary that defines the updates to make.

``` python
import pymongo
from pymongo import MongoClient
import pprint
from datetime import datetime
import re
from IPython.display import clear_output

# Replace XXXX with your connection URI from the Atlas UI
client = MongoClient(mongoURI)

runtime_pat = re.compile(r'([0-9]+) min')

for movie in client.mflix.movies.find({}):

    fields_to_set = {}
    fields_to_unset = {}

    for k,v in movie.copy().items():
        if v == "" or v == [""]:
            del movie[k]
            fields_to_unset[k] = ""

    if 'director' in movie:
        fields_to_unset['director'] = ""
        fields_to_set['directors'] = movie['director'].split(", ")
    if 'cast' in movie:
        fields_to_set['cast'] = movie['cast'].split(", ")
    if 'writer' in movie:
        fields_to_unset['writer'] = ""
        fields_to_set['writers'] = movie['writer'].split(", ")
    if 'genre' in movie:
        fields_to_unset['genre'] = ""
        fields_to_set['genres'] = movie['genre'].split(", ")
    if 'language' in movie:
        fields_to_unset['language'] = ""
        fields_to_set['languages'] = movie['language'].split(", ")
    if 'country' in movie:
        fields_to_unset['country'] = ""
        fields_to_set['countries'] = movie['country'].split(", ")

    if 'fullplot' in movie:
        fields_to_unset['fullplot'] = ""
        fields_to_set['fullPlot'] = movie['fullplot']
    if 'rating' in movie:
        fields_to_unset['rating'] = ""
        fields_to_set['rated'] = movie['rating']

    imdb = {}
    if 'imdbID' in movie:
        fields_to_unset['imdbID'] = ""
        imdb['id'] = movie['imdbID']
    if 'imdbRating' in movie:
        fields_to_unset['imdbRating'] = ""
        imdb['rating'] = movie['imdbRating']
    if 'imdbVotes' in movie:
        fields_to_unset['imdbVotes'] = ""
        imdb['votes'] = movie['imdbVotes']
    if imdb:
        fields_to_set['imdb'] = imdb

    if 'released' in movie:
        fields_to_set['released'] = datetime.strptime(movie['released'],
                                                      "%Y-%m-%d")
    if 'lastUpdated' in movie:
        fields_to_set['lastUpdated'] = datetime.strptime(movie['lastUpdated'][0:19],
                                                         "%Y-%m-%d %H:%M:%S")

    if 'runtime' in movie:
        m = runtime_pat.match(movie['runtime'])
        if m:
            fields_to_set['runtime'] = int(m.group(1))

    update_doc = {}
    if fields_to_set:
        update_doc['$set'] = fields_to_set
    if fields_to_unset:
        update_doc['$unset'] = fields_to_unset
    pprint.pprint(update_doc)

    db.movies.update_one({'_id': movie['_id']}, update_doc)

```

- `$set` updates or adds fields to a document
- `$unset` removes keys (and the values) from a document
  - here: remove singular values, or the IMDB key/value pairs that were combined into a nested document.

## Updating Documents, Part 2

- `.find({})` looks for an empty dictionary, i.e. every docuument in the collection
  - returns a cursor, which lets you iterate over the retreived Documents
  - `.limit(x)` operates on a cursor and return a cursor.
    - this lets python iterate over the cursor.
    - pymongo pulls documents in batches to be more efficient.
- `if` statements eliminate fields, adding them to `fields_to_unset` dictionary, and splits them if they exist to the `fields_to_set` dictionary
  - also use dateTime classes to parse dates
- This technique also lets us parse the runtime (i.e. "1 min") using regular expressions, and convert it to an integer.
- As long as a value exists, it is retained, unless it is updated or removed using `$set/$unset`
- `.update_one` isn't useful when trying to operate over an an entire collection, inefficient.

## Bulk updates

- Bulk writes: multiple updates with a single request
  - batch together and make a single request to the database.
- instead of `.update_one`, we use the UpdateOne class to create an Operation object.
  - adds an update to a list. Once the list hits the user-defined batch size, the `.bulk_write` method is called.

  ``` python
  import pymongo
  from pymongo import MongoClient, UpdateOne
  from datetime import datetime
  import pprint
  import re
  from IPython.display import clear_output

  # Replace XXXX with your connection URI from the Atlas UI
  client = MongoClient(mongoURI)

  runtime_pat = re.compile(r'([0-9]+) min')

  batch_size = 1000
  updates = []
  count = 0
  for movie in client.mflix.movies_initial.find({}):

      fields_to_set = {}
      fields_to_unset = {}

      for k,v in movie.copy().items():
          if v == "" or v == [""]:
              del movie[k]
              fields_to_unset[k] = ""

      if 'director' in movie:
          fields_to_unset['director'] = ""
          fields_to_set['directors'] = movie['director'].split(", ")
      if 'cast' in movie:
          fields_to_set['cast'] = movie['cast'].split(", ")
      if 'writer' in movie:
          fields_to_unset['writer'] = ""
          fields_to_set['writers'] = movie['writer'].split(", ")
      if 'genre' in movie:
          fields_to_unset['genre'] = ""
          fields_to_set['genres'] = movie['genre'].split(", ")
      if 'language' in movie:
          fields_to_unset['language'] = ""
          fields_to_set['languages'] = movie['language'].split(", ")
      if 'country' in movie:
          fields_to_unset['country'] = ""
          fields_to_set['countries'] = movie['country'].split(", ")

      if 'fullplot' in movie:
          fields_to_unset['fullplot'] = ""
          fields_to_set['fullPlot'] = movie['fullplot']
      if 'rating' in movie:
          fields_to_unset['rating'] = ""
          fields_to_set['rated'] = movie['rating']

      imdb = {}
      if 'imdbID' in movie:
          fields_to_unset['imdbID'] = ""
          imdb['id'] = movie['imdbID']
      if 'imdbRating' in movie:
          fields_to_unset['imdbRating'] = ""
          imdb['rating'] = movie['imdbRating']
      if 'imdbVotes' in movie:
          fields_to_unset['imdbVotes'] = ""
          imdb['votes'] = movie['imdbVotes']
      if imdb:
          fields_to_set['imdb'] = imdb

      if 'released' in movie:
          fields_to_set['released'] = datetime.strptime(movie['released'],
                                                        "%Y-%m-%d")
      if 'lastUpdated' in movie:
          fields_to_set['lastUpdated'] = datetime.strptime(movie['lastUpdated'][0:19],
                                                           "%Y-%m-%d %H:%M:%S")

      if 'runtime' in movie:
          m = runtime_pat.match(movie['runtime'])
          if m:
              fields_to_set['runtime'] = int(m.group(1))

      update_doc = {}
      if fields_to_set:
          update_doc['$set'] = fields_to_set
      if fields_to_unset:
          update_doc['$unset'] = fields_to_unset

      updates.append(UpdateOne({'_id': movie['_id']}, update_doc))

      count += 1
      if count == batch_size:
          client.mflix.movies_initial.bulk_write(updates)
          updates = []
          count = 0

  if updates:         
      client.mflix.movies_initial.bulk_write(updates)
  ```

- A duplicate `movies` collection is created for testing purposes, to ensure it works before overwriting `movies_initial`

``` bash
mongoimport --drop --type csv --headerline --db mflix --collection movies --host "<CLUSTER>/<SEED_LIST>" --authenticationDatabase admin --ssl --username analytics --password analytics-password --file movies_initial.csv
```

## Data Types in MongoDB

``` python
import pymongo
import pprint

# Replace XXXX with your connection URI from the Atlas UI
course_cluster_uri = "localhost:27017"

course_client = pymongo.MongoClient(course_cluster_uri)
movies = course_client['mflix']['movies_initial']

movie = movies.find_one()

pprint.pprint(movie)


movie['_id']

from datetime import datetime

dates = course_client['test']['dates']

dates.insert_one({ "dt": datetime.utcnow() })

dates.find_one()


from bson.decimal128 import Decimal128
decimals = course_client['test']['decimals']

decimals.insert_one({ "money": Decimal128("99.99") })

decimals.find_one()

movies.find_one({ "year": { "$type": "int" } })
```

- `ObjectId` is a special data type.
  - allows mongo to uniquely identify a documents
  - current time in UNIX, PID, machine ID, and a random Number

- `date` type stores the individual pieces of the time (hour, minute, etc)
  - Lets the aggregation framework determine which day of the year a date is, etc.
- `decimal128` is used where precision is necessary (i.e. money)
  - By default, all numbers are stored as double-precision.
- data types can be queried using `$type` as in the last example above

## Filtering on Array fields

- In Compass, we can query for a particular language using the filter `{languages: "Korean"}`
  - Right now, it's pulling in any string the array
  - can filter with multiples, too
    - `{languages: {$all: ["Korean","English"]}}`
    - order and position don't matter.

``` python
import pymongo
from pymongo import MongoClient
import pprint
from IPython.display import clear_output

# Replace XXXX with your connection URI from the Atlas UI
client = MongoClient(mongoURI)

filter = {'languages': {'$all': ['Korean', 'English']}}

clear_output()
pprint.pprint(list(client.mflix.movies_initial.find(filter)))

filter = {'languages': ['Korean', 'English']}

clear_output()
pprint.pprint(list(client.mflix.movies_initial.find(filter)))

filter = {
  'languages.0': 'Korean'
}

projection = {'languages': 1, 'title': 1, '_id': 0}

clear_output()
pprint.pprint(list(client.mflix.movies_initial.find(filter, projection)))
```
- Movies listed in decreasing order of frequency used in a given movie.
- Exact matches can be made on the array, as well.
  - just eliminate the `$all` operator!
  - Korean is used more, as it's first (index 0)
  - Switching the order, and different movies are found
- Filter for all documents with korean in index 0
  - uses dot notation: `{languages.0: "Korean"}`
- `find()` can accept a dictionary as a second argument in Python
  - These projections explicitly include fields with a `1`, and exclude them with `0`
  - `_id` field is always included, unless explicitly excluded.
  - Order of the returned projections is as stated in the dictionary.

## How MFlix works with MongoDB

- Mflix runs on `flask` a python web interface.
- `db.py` controls access/connection to the database.
- `mflix.py` contains the flask app
  - uses methods in `db.py` to complete requests
  - `auth.py` controls log in/out

## Sort, Skip and limit

- `sort()`, `skip()`, and `limit()` are cursor methods
- Homepage of MFlix only displays 20 titles (from a 40k+ database)
  - in `mflix.py`, the `@app.route('/')` indicates what the app should do when in the '/' directory of the web app (and the method called when the page is visited)
  - `get_movies()` accepts filters, the page, and the movies per page.
    - returns the list of films when called.
    - defined in `db.py` (and below)
    - `sort_key` returns movies by number of reviews
      - `.sort` uses same dot notation for nested documents (i.e. wehn filtering)
    - `.skip(x)` removes x documents from the documents retreived in `movies`
    - `.limit(x)` returns x documents from the remaining list (after skipping)      

``` python
from pymongo import MongoClient, DESCENDING
import pprint

# Replace XXXX with your connection URI from the Atlas UI
course_cluster_uri = "localhost:27017"

course_client = pymongo.MongoClient(course_cluster_uri)
db = course_client['mflix']

filters = {}
page = 0
movies_per_page = 20

sort_key = "tomatoes.viewer.numReviews"
​
# DESCENDING to return most first!
movies = db.movies.find(filters) \
                  .sort(sort_key, DESCENDING)

# count number of total movie documents
total_num_movies = movies.count()
pprint.pprint(total_num_movies)

# limit records based on page number
movies.skip(movies_per_page * page) \
               .limit(movies_per_page)

movie_list = list(movies)

len(movie_list)

movie_list[0]

# Simulate going to the next page
page = 1

movies = db.movies.find(filters) \
                  .sort(sort_key, DESCENDING) \
                  .skip(movies_per_page * page)

# 20 fewer movies, since we're on page 2 (index 1)
len(list(movies))
```

## Query movies using operators
- `$gte` is greater than or equal to.
- `$lt` is less than!
- `$in` can take an array of values and returns any matching entried!
  - `$nin` is the opposite
- `$eq` is equal to.
  - to negate, nest dictionaries, with the first (outermost) key being `$not`
- Queries are dictionary-esque structures on a field (i.e. `year`)
- Additional logical and element operators cab be found in the docs:
  - [Query and Projection Operators](https://docs.mongodb.com/manual/reference/operator/query/)


``` python
import pymongo
import pprint

# Replace XXXX with your connection URI from the Atlas UI\n",
mc = pymongo.MongoClient("localhost:27017")
mflix = mc['mflix']

# find all movies released from 1983 onwards\n",
filters = {'year': { '$gte': 1983 }}

for movie in mflix.movies.find(filters):
    pprint.pprint(movie['title'])

# find all movies between 1989 and 1999\n",
filters = { 'year': {'$gte': 1989, '$lt': 2000} }

for movie in mflix.movies.find(filters):
    pprint.pprint(movie['title'])

# find all movies released in 1995, 2005, 2015\n",
filters = { 'year': { '$in': [ 1995, 2005, 2015 ] } }
for movie in mflix.movies.find(filters):
    pprint.pprint(movie['title'])

# find all movies except the ones which are of adult genre\n",
filters = { 'year': { '$in': [ 1995, 2005, 2015 ] },
                     'genre': { '$not' : {'$eq': 'Adult'} } }
for movie in mflix.movies.find(filters):
    pprint.pprint(movie['title'])
```

## The `$elemMatch` Operator

- Useful when querying subdocuments embedded in an array
  - Here, `comments` stores each comment and meta data as a subdocument within an array.
- Can query embedded documents using dot notation
  - To search on `name` within `comments` - `{'comment.name' : 'James'}`
  - But! When combining multiple queries, it's not restricted to one document within an array - so the name and date could be in separate subdocuments.
- `$elemMatch` makes sure both conditions are met within the same document.

``` python
# When you put a bang (exclamation point) at the beginning of a line, everything that follows
# will be executed in your terminal.
#
# In this case, we're using pip to install the dateparser module.
# This module will help us parse datetimes from strings.
!pip install dateparser

import pymongo
import pprint
import dateparser

# Replace XXXX with your connection URI from the Atlas UI
course_cluster_uri = "localhost:27017"

course_client = pymongo.MongoClient(course_cluster_uri)
movies = course_client['mflix']['movies']

# Find a document with comments, and return just the comments field.
query = {"comments":{"$exists": True}}
projection = {"comments": 1}
​
movie = movies.find_one(query, projection)
​
pprint.pprint(movie)

query = {"comments.name": "Samwell Tarly"}

movie = movies.find_one(query, projection)
​
pprint.pprint(movie)

query = {
  "comments.name": "Samwell Tarly",
  "comments.date": {
    "$lt": dateparser.parse("1995-01-01")
  }
}
​
movie = movies.find_one(query, projection)
​
pprint.pprint(movie)

movie = movies.find(query, projection).skip(1).limit(1)
​
pprint.pprint(list(movie))

betterQuery = {
  "comments": {
    "$elemMatch": {
      "name": "Samwell Tarly",
      "date": {
        "$lt": dateparser.parse("1995-01-01")
      }
    }
  }
}
​
correctMovies = list(movies.find(betterQuery, projection).limit(2))
​
pprint.pprint(correctMovies)
```
## Querying on Rotten Tomatoes Subdocuments

- Use dot notation to access nested elements within an array.


``` python

import pymongo
import pprint

# Replace XXXX with your connection URI from the Atlas UI
uri = "localhost:27017"
mc = pymongo.MongoClient(uri)
mflix = mc.mflix

# Find `Titanic` movie
titanic = mflix.movies.find_one({'title': 'Titanic'})
​
pprint.pprint(titanic)

# tomatoes contains nested documents of Rotten Tomatoes data.
titanic["tomatoes"]

titanic["tomatoes"]["viewer"]

rating = titanic["tomatoes"]["viewer"]["rating"]
rating

# find all movies with the same viewer rating as `Titanic`
cursor = mflix.movies.find({'tomatoes.viewer.rating': rating })
for movie in cursor:
    pprint.pprint(movie['title'])

# find all movies with `Titanic` rating and sort by lastUpdated review
cursor = mflix.movies.find({'tomatoes.viewer.rating': rating })
# Sort by oldest to newest, which modifies the cursor in place
cursor.sort('tomatoes.lastUpdated', pymongo.ASCENDING)
for movie in cursor:
    pprint.pprint( "Title: {0}; LastUpdated: {1}".format( movie['title'],movie['tomatoes']['lastUpdated']))
​
```

## Inserting Comments into MFlix

- Since `comments` is an array of subdocuments, we're inserting a dictionary.
- `bypass_validation` flag is set to false
  - validation is on the collection level.

``` python
# Connect to Atlas

import pymongo
import pprint
import bson.objectid
from datetime import datetime

# Replace XXXX with your connection URI from the Atlas UI
uri = "localhost:27017"
mc = pymongo.MongoClient(uri)
mflix = mc['mflix']
print(mflix)


# Insert a single document

# fake comment, in a dictionary
comment = {
    "name": "some users's name",
    "email": "someuser@email.com",
    "movie_id": bson.objectid.ObjectId(),
    "text": "some nice comment on our movie",
    "date": datetime.utcnow()
    }

bypass_validation = False

insert_result = mflix.comments.insert_one(comment, bypass_validation)

# Was it acknowledged by the server?
pprint.pprint(insert_result.acknowledged)

# Mongo creates an insert ID if none is provided.
pprint.pprint(insert_result.inserted_id)
# Insert document providing _id value

# another fake comment
comment = {
    "_id": "some_id_field",
    "name": "some users's name",
    "email": "someuser@email.com",
    "movie_id": bson.objectid.ObjectId(),
    "text": "Hi, it's me again!",
    "date": datetime.utcnow()
    }
insert_result = mflix.comments.insert_one(comment, bypass_validation)
pprint.pprint(insert_result.acknowledged)
pprint.pprint(insert_result.inserted_id)
# Duplicate documents will raise error

# _id must be unique, so it throws an error for Duplicate keys!
comment = {
    "_id": "some_id_field"}
insert_result = mflix.comments.insert_one(comment, bypass_validation)

```

## Updating Comments

- Users can add comments in the app via `add_comment_to_movie()` method in `db.py`
- A **subset** of data is used
  - First updated in `movies` collection, and then into a collection of all Comments
  - `$push` is used in `movies` collection
    - saves bandwidth, just sends what's needed and not the whole document
- `MOVIE_COMMENT_CACHE_LIMIT` limits the amount stored in a document
  - prevents **unbounded arrays** by limiting the amount of data stored
  - Optimized code but by taking only the most recent 10 comments.
  - `$slice`, `$sort` and `$each` defines how values of the update array are used
    - `$inc` increments the number of comments by 1
    - `$each` appends the value (in this case, the `comment_doc` dictionary) to the array. Duplicates are ignored!
  - Update operators allow inserts, but without asking for a counter or min/max without reading it. (all *server side*)


``` python
def add_comment_to_movie(movieid, user, comment, date):
    MOVIE_COMMENT_CACHE_LIMIT = 10

    comment_doc = {
        "name": user.name,
        "email": user.email,
        "movie_id": movieid,
        "text": comment,
        "date": date
    }

    movie = get_movie(movieid)
    if movie:
        update_doc = {
            "$inc": {
                "num_mflix_comments": 1
            },
            "$push": {
                "comments": {
                    "$each": [comment_doc],
                    "$sort": {"date": -1},
                    "$slice": MOVIE_COMMENT_CACHE_LIMIT
                }
            }
        }

        # let's set an `_id` for the comments collection document
        comment_doc["_id"] = "{0}-{1}-{2}".format(movieid, user.name, \
            date.timestamp())

        db.comments.insert_one( comment_doc )

        db.movies.update_one({"_id": ObjectId(movieid)}, update_doc)
```
- [Advanced Schema Design Patterns](https://www.slideshare.net/mongodb/advanced-schema-design-patterns?jmp=coursera-intro-mongodb)

## Deleting Data in MFlix
- `delete_one` and `delete_many` methods are used to remove Information
- in the App, each comment checks to see if logged in user's email matches the one for a given comment and renders a button if needed.
- in `mflix.py` is the `delete_movie_comment` menthod, which calls  `delete delete_comment_from_movie` in `db.py`
  - Last 10 comments are cashed for a given movie.
  - method operates on `ObjectId`, but any filter can be applied.
  - deletes comment from `comment` collection
  - in the app, comments will still show a count - the number is stored in the `movies` collection
``` python
import pymongo
from bson.objectid import ObjectId

# Replace XXXX with your connection URI from the Atlas UI
course_cluster_uri = "localhost:27017"

course_client = pymongo.MongoClient(course_cluster_uri)
comments = course_client['mflix']['comments']

filter = {"text": "What a great movie."}

list(comments.find(filter))

comments.delete_one(filter)

list(comments.find(filter))

titanic_id = ObjectId("573a139af29313caabcf0d74")
titanic_filter = {"movie_id": titanic_id}
comments.delete_many(titanic_filter)

movies = course_client['mflix']['movies']
update_doc = {
    "$set": {
        "comments": [],
        "num_mflix_comments": 0
    }
}
movies.update_one(titanic_filter, update_doc)

```
