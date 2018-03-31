# Week 1 (through 8 April 2018)

## MongoDB Document Model
- Instead of hard-coded tables of a relational database, in which each row is an observation entered into a pre-defined structure, [MongoDB](mongodb.com) is a collection of **documents**.
  - These are more like `JSON` structures, or `python` dictionaries.
  - **Documents** consist of **fields**, which are themselves sets of **key**/**value** pairs.
  - **Values** can be the usual strings, integers, dates, etc. They can also be **arrays**, additional documents, or even arrays of documents.
- The document system (in theory) saves time by aggregating data in a single queriable object, instead of creating one thorugh a sequence of joins (as in a relational database).

## Installation

## Setting up the Course Environment

## Importing to an Atlas cluster

- Mongo has a Database as a service platform called [Atlas](cloud.mongodb.com).
  - There's a free tier, which we'll use for this course.
- In order to access the database, we'll download and install the latest Enterprise installer.
  - The `.bash_profile` file in the user directory

```bash
mongoimport --type csv --headerline --db mflix --collection movies_initial --host "<CLUSTER>/<SEED_LIST>" --authenticationDatabase admin --ssl --username analytics --password analytics-password --file movies_initial.csv
```


#### System Information

- The course was completed on OSX v10.11.6
- MongoDB shell version v3.6.3
- python v3.5.2
