# Week 1 (through 8 April 2018)

## MongoDB Document Model
- Instead of hard-coded tables of a relational database, in which each row is an observation entered into a pre-defined structure, [MongoDB](mongodb.com) is a collection of **documents**.
  - These are more like `JSON` structures, or `python` dictionaries.
  - **Documents** consist of **fields**, which are themselves sets of **key**/**value** pairs.
  - **Values** can be the usual strings, integers, dates, etc. They can also be **arrays**, additional documents, or even arrays of documents.
- The document system (in theory) saves time by aggregating data in a single queriable object, instead of creating one thorugh a sequence of joins (as in a relational database).

## Installation

- In order to interact with Atlas, it needs the MongoDB shell
- In order to access the database, we'll download and install the latest Enterprise installer.
  - The executables need to be found by the terminal.
  - Locations are stored in the `PATH` variable in the `.bash_profile` file in the user directory
    - Insert the following into `.bash_profile`
    - the `:$PATH` at the end concatenates the string onto the existing PATH, retaining existing paths.

``` bash
export PATH="~/mongodb-osx-x86_64-enterprise-3.6.3/bin:$PATH"
```

## Setting up the Course Environment

- [Anaconda](www.anaconda.org) can help manage the python environments.
  - Environments are separate from each other and can contain different versions of files and/or packages.
- Create an environment via terminal:
  -
  ``` bash
  conda create -n intro-to-mongodb
  ```
- Activate the environment:
  ``` bash
  source activate intro-to-mongodb
  ```
  - Once activated, the environment appears within parentheses at the beginning of the prompt
    - i.e. `(intro-to-mongodb)`
    - Dependencies can now be installed into the active environment
    - `pymongo` connects Python to MongoDB:
    ``` bash
    pip install pymongo dnspython
    ```
- Deactivating the environment:
  ``` bash
  source deactivate intro-to-mongodb
  ```
- Anaconda also supports Jupyter notebooks
  - Within a given folder, run
  ``` bash
  jupyter notebook
  ```
  - And access via http://localhost:8888 in a web browset.

## Importing to an Atlas cluster

- Mongo has a Database as a service platform called [Atlas](cloud.mongodb.com).
  - There's a free tier, which we'll use for this course.
- Run the following to insert the `movies_initial.csv` file into the Database:
```bash
mongoimport --type csv --headerline --db mflix --collection movies_initial --host "<CLUSTER>/<SEED_LIST>" --authenticationDatabase admin --ssl --username analytics --password analytics-password --file movies_initial.csv
```
- Once run, the import states how many documents were uploaded.

## Using Compass
- GUI for mongodb
- Documents are records in collections (analogous to rows in a tables)
- Can filter/find in Compass `{key:"value"}`
- Can edit values directly in documents.
- `pymongo` library links to MongoDB, allowing further aggregation and query framework.
- Multiple replicas in Atlas - one is primary, rest hold complete copies of database.
  - Compass and pymongo switch seamlessly if the primary goes down.
- [PyMongo documentation](https://api.mongodb.com/python/current)

## Connecting to Atlas from Python
```python
# Import the URI from keys.py in the data folder.
from data.keys import mongoURI

from pymongo import MongoClient
client = MongoClient(mongoURI)
print(client.mflix)
```

## Aggregation framework
- General idea: take data from a collection, pass it through several *stages* (operations), and produce an output.
  - Inputs/outputs of all stages are documents (a stream of documents).
  - stages expect a specific kind of input and produce a specific type of output
- Stages process stream of documents one at a time
- Example: looking at languages
```python
from data.keys import mongoURI

import pymongo
from pymongo import MongoClient
import pprint

client = MongoClient(mongoURI)

pipeline = [
  {
    '$group' : {
      '_id' : {"language" : "$language"},
      'count' : {'$sum' : 1}
    }
  }]

pprint.pprint(list(client.mflix.movies_initial.aggregate(pipeline)))
```
- identifier for group stage starts with a `$`
  - Here, groups identified by a dictionary with a single value: "language"
    - The value is "$language", or the string values found in the collection.
      - *field-path identifier* - value used where the placeholder is found
    - *accumulator* is $sum, which operates on the group as defined.
      - See more [accumulators](https://docs.mongodb.com/manual/reference/operator/aggregation/group/index.html#accumulator-operator) in the documentation.
- `$group` operates of the `movies_initial` collection found in `mflix` on the `client`
  - `aggregate` is a collection class method, receives the pipleine (dictionary) as an argument
- Group stages let any number of accumulators apply to a group
  - Outputs one document for each group
  - Here: two fields
    - `_id`: the language being accumulated
    - `count`: Number of documents in the group
- `pprint` produces a tidier output.
- languages are comma separated - additional stages can help!
- add the `$sort` stage
  - specify the field to be sorted on - "1" is asc, "-1" is dsc
  - "count" can be the field, as it receives the grouped output
```python
  from data.keys import mongoURI

  import pymongo
  from pymongo import MongoClient
  import pprint

  client = MongoClient(mongoURI)

  pipeline = [
    {
      '$group' : {
        '_id' : {"language" : "$language"},
        'count' : {'$sum' : 1}
      }
    },
    {
      '$sort' : {'count' : -1}
    }]

  pprint.pprint(list(client.mflix.movies_initial.aggregate(pipeline)))
```
- All of the above can be simplified, as counting by group is common:
```python
from data.keys import mongoURI

import pymongo
from pymongo import MongoClient
import pprint

client = MongoClient(mongoURI)

pipeline = [
  {
    '$sortByCount' : "$language"
    }
]

pprint.pprint(list(client.mflix.movies_initial.aggregate(pipeline)))
```
- `$facet` stage can run multiple pipelines at once
  - Here, details on specific language combos and raw counts on them.
```python
from data.keys import mongoURI

import pymongo
from pymongo import MongoClient
import pprint

client = MongoClient(mongoURI)

pipeline = [
    {
      '$sortByCount' : "$language"
      },
    {
      '$facet' : {
        'top language combinations' : [{'$limit' : 100}],
        'unusual combinations shared by' : [{
          '$skip' : 100
        },
        {
          '$bucketAuto' : {
            'groupBy' : "$count",
            'buckets' : 5,
            'output' : {
              'language combinations' : {'$sum':1}
            }
          }
        }]
      }
    }
  ]

pprint.pprint(list(client.mflix.movies_initial.aggregate(pipeline)))
```
- Each array is a pipeline processed in parallel, following the `sortByCount`
  - First one finds top 100 Documents
  - Second skips first 100 documents, passes rest on to next stage
    - `bucketAuto` collapses groups into buckets (5 or fewer here)
      - must specify output
        - here, number of language combinations (`'$sum' : 1`)
        - output shows how language combinations were used by a minimum and maximum number of movies.


## Filtering
- Try and obtain documents which match certain criteria.



#### System Information

- The course was completed on Mac OSX v10.11.6
- MongoDB shell version v3.6.3
- python v3.5.2
