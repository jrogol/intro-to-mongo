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
-


#### System Information

- The course was completed on OSX v10.11.6
- MongoDB shell version v3.6.3
- python v3.5.2
