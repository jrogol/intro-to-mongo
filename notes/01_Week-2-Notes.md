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
- Nothing should show ,switch to compass to see the changes in the `movies_scratch` collection.
