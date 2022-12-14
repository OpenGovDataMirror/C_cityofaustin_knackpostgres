# knackpostgres

### *** This Library is Under Construction ****

Convert Knack applications to a PostgreSQL database.

## Installation

1. Clone this repo

```bash
$ git clone http://github.com/cityofaustin/knackpostgres
```

2. Install the library

```bash
$ pip install knackpostgres
```

## Usage

*If you're new to Knack + Python, consider learning via [Knackpy](https://github.com/cityofaustin/knackpy).*

### Convert your App to PostgreSQL

`knackpostgres` will generate a series of Postgres-compliant SQL commands which translate your Knack app's schema to a PostgreSQL schema. 

```python
>>> from knackpostgres import App

# find your app id: https://www.knack.com/developer-documentation/#find-your-api-key-amp-application-id

# optionally include a list of object keys defining which objects to include
>>> app = App(
    "myappidstring",
    obj_filter=["object_12", "object_13"],
    schema="public", # default
    metadata_schema="__meta__", # default
)
```

If you want to execute the SQL commandsd manually, you can write the App's SQL commands to files:

```python
>>> app.to_sql(path="sql") # `sql` is the default relative path
```

Alternatively, you can use the `Loader` class to execute your app's SQL. Read on...

### Quick Create PostgreSQL databse

If you're in need of a postgresql database for development. Consider the [official docker images](https://hub.docker.com/_/postgres).

To start your database with the default username (`postgres`) in a new container:

```bash
$ docker run -p 5432:5432 --name postgres -e POSTGRES_PASSWORD=my_password -d my-db-name
```

You can explore the database by connecting with a `psql` container: 

```bash
docker run -it --rm --network host my-db-name psql -h localhost -U postgres
```

### Deploy Knack Database Schema to PostgreSQL

*You'll need to manually install [`psycopg2`](https://pypi.org/project/psycopg2/) into your Python environment. Because of installation headaches, it is not installed automatically.
TODO: use docker ;)*

The `Loader` class is available to execute your App's data into a PostgreSQL database.

Start by passing your `App` to a new `Loader` instance.

```python
>>> from knackpostgres import App, Loader

>>> app = App("myappidstring")

# This will overwrite the schema in the destination DB!
>>> loader = Loader(app, overwrite=True)
```

Connect to your database:

```python
>>> loader.connect(
    host="localhost", # default
    dbname="postgres", # default
    port=5432, # default
    user="postgres", # default
    password="myunguessabledatabasepassword",
    )
```

Execute your `App`'s sql commands:

```python
>>> loader.create_schema()

>>> loader.create_tables()

# read-only fields (formule & equations) are only available in views
>>> loader.create_views()

```

Your schema has been created! Now you can load data.

```python
    [...]
    
    import KnackTranslator, MetaTranslator
    from knackpy import Knack

    [...]
    """
    We first load the apps field metadata `_meta._fields`
    """
    for table in app.metadata:
         translator = MetaTranslator(table)
         translator.to_sql()
         loader.execute(translator.sql)

    for table in app.tables:
        # for each knack table download knack data, translate it, and load it
        if "object" not in table.key:
            # ignore associative tables, which are generated by many-to-many connections
            continue

        kn = Knack(
            obj=table.key,
            app_id=app_id,
            api_key=api_key
        )

        try:
            # translate the knack data to Postgres schema
            translator = KnackTranslator(kn, table)

        except IndexError:
            # raised when no records are found in the specified object 
            continue

        # convert the translated data to `INSERT` statements
        translator.to_sql()

        loader.execute(translator.sql)

        # save foreign key references
        loader.connections_sql += translator.connections_sql()

    # now that all records have been loaded, we can update foreign key references
    loader.update_connections()
```

### Knack Feature Coverage

This is a work in progress. Currently supported Knack features include:

Validation and conditional rules are not supported. These should be implemented outside of the database. See [React](#react) notes.

All **connection field** types are supported, although self-connections are not well tested.

**Address** and **Name** fields are supported and stored as `JSON` types.

Standard **[formula fields](https://support.knack.com/hc/en-us/articles/226583008-Formulas)** are supported

**Equation** fields are not yet supported.

For **Concatentation (aka Text Formula)** fields, all **[Text Functions](https://support.knack.com/hc/en-us/articles/115005002328-Text-Formula-Functions)** are supported.

The only supported **text formula** date functions are `getDateDayOfWeekName` and `getDateMonthOfYearName`. You can add support for others by writing your own [`MethodHandler`](https://github.com/cityofaustin/knackpostgres/blob/master/knackpostgres/method_handler.py) method.