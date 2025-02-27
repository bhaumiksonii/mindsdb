# Predicting Home Rental Prices with MindsDB

## Introduction

Follow these steps to create, train and query a machine learning model (predictor) using SQL that predicts the `rental_price` (label) for new properties given their attributes (features).

## The Data

### Connecting the data

There are a couple of ways you can get the data to follow trough this tutorial.

=== "Connecting as a database via `#!sql CREATE DATABASE`"

    We have prepared a demo database you can connect to that contains the data to be used `#!sql example_db.demo_data.home_rentals` 

    ```sql
    CREATE DATABASE example_db
        WITH ENGINE = "postgres",
        PARAMETERS = {
            "user": "demo_user",
            "password": "demo_password",
            "host": "3.220.66.106",
            "port": "5432",
            "database": "demo"
    }
    ```

    Now you can run queries directly on the demo database. Let's start by previewing the data we will use to train our predictor:

    ```sql
    SELECT * 
    FROM example_db.demo_data.home_rentals 
    LIMIT 10;
    ```

=== "Connecting as a file"
    You can download **[the source file as a `.CSV` here](https://mindsdb-test-file-dataset.s3.amazonaws.com/home_rentals.csv)** and then upload via [MindsDB SQL Editor](connect/mindsdb_editor/)

    <figure markdown> 
        ![log-in](/assets/cloud/import_file.png){ width="600", loading=lazy }
        <figcaption>Navigate to the Upload a file button.</figcaption>
    </figure>

    <figure markdown> 
        ![log-in](/assets/cloud/import_file_2.png){ width="600", loading=lazy }
        <figcaption>Import the file and name it home_rentals</figcaption>
    </figure>

    Now you can run queries directly on the demo file as if it was a database. Let's start by previewing the data we will use to train our predictor:

    ```sql
    SELECT *
    FROM files.home_rentals
    LIMIT 10;
    ```

!!! Warning "From now onwards we will use the table `#!sql example_db.demo_data.home_rentals` make sure you replace it for `files.home_rentals` if you are connecting the data as a file."

### Understanding the Data

```sql
+-----------------+---------------------+------+----------+----------------+----------------+--------------+
| number_of_rooms | number_of_bathrooms | sqft | location | days_on_market | neighborhood   | rental_price |
+-----------------+---------------------+------+----------+----------------+----------------+--------------+
|               2 |                   1 |  917 | great    |             13 | berkeley_hills |         3901 |
|               0 |                   1 |  194 | great    |             10 | berkeley_hills |         2042 |
|               1 |                   1 |  543 | poor     |             18 | westbrae       |         1871 |
|               2 |                   1 |  503 | good     |             10 | downtown       |         3026 |
|               3 |                   2 | 1066 | good     |             13 | thowsand_oaks  |         4774 |
+-----------------+---------------------+------+----------+----------------+----------------+--------------+
```

Where:

| Column                | Description                                                                                  | Data Type           | Usage   |
| :-------------------- | :------------------------------------------------------------------------------------------- | ------------------- | ------- |
| `number_of_rooms`     | Number of rooms of a given house `[0,1,2,3]`                                                 | `integer`           | Feature |
| `number_of_bathrooms` | Number of bathrooms on a given house `[1,2]`                                                 | `integer`           | Feature |
| `sqft`                | Area of a given house in square feet                                                         | `integer`           | Feature |
| `location`            | Rating of the location of a given house `[poor, great, good]`                                | `character varying` | Feature |
| `days_on_market`      | Number of days a given house has been open to be rented                                      | `integer`           | Feature |
| `neighborhood`        | Neighborhood a given house is in `[alcatraz_ave, westbrae, ..., south_side, thowsand_oaks ]` | `character varying` | Feature |
| `rental_price`        | Price for renting a given house in dollars                                                   | `integer`           | Label   |

!!!Info "Labels and Features"

    A **label** is the thing we're predicting—the y variable in simple linear regression ...
    A **feature** is an input variable—the x variable in simple linear regression ..

## Training a Predictor Via [`#!sql CREATE PREDICTOR`](/sql/create/predictor)

Let's create and train your first machine learning predictor. For that we are going to use the [`#!sql CREATE PREDICTOR`](/sql/create/predictor) syntax, where we specify what sub-query to train `#!sql FROM` (features) and what we want to learn to `#!sql PREDICT` (labels):

```sql
CREATE PREDICTOR mindsdb.home_rentals_model
FROM example_db
  (SELECT * FROM demo_data.home_rentals)
PREDICT rental_price;
```

## Checking the Status of a Predictor

A predictor may take a couple of minutes for the training to complete. You can monitor the status of your predictor by copying and pasting this command into your SQL client:

```sql
SELECT status
FROM mindsdb.predictors
WHERE name='home_rentals_predictor';
```

Here we are selecting the status from the table called mindsdb.predictors and using the where statement to only show the model we have just trained, On execution, you we get:

```sql
+----------+
| status   |
+----------+
| training |
+----------+
```

Or after a the model has been trained:

```sql
+----------+
| status   |
+----------+
| complete |
+----------+
```

## Making Predictions

!!! attention "Predictor Status Must be 'complete' Before Making a Prediction"

### Making Predictions Via [`#!sql SELECT`](/sql/api/select)

Once the predictor's status is complete. You can make predictions by querying the predictor as if it was a normal table:
The [`SELECT`](/sql/api/select/) syntax will allow you to make a prediction based on features.

```sql
SELECT rental_price,
       rental_price_explain
FROM mindsdb.home_rentals_model
WHERE sqft = 823
AND location='good'
AND neighborhood='downtown'
AND days_on_market=10;
```

On execution, you should get:

```sql
+--------------+-----------------------------------------------------------------------------------------------------------------------------------------------+
| rental_price | rental_price_explain                                                                                                                          |
+--------------+-----------------------------------------------------------------------------------------------------------------------------------------------+
| 4394         | {"predicted_value": 4394, "confidence": 0.99, "anomaly": null, "truth": null, "confidence_lower_bound": 4313, "confidence_upper_bound": 4475} |
+--------------+-----------------------------------------------------------------------------------------------------------------------------------------------+
```

### Making Batch Predictions Via [`#!sql JOIN`](/sql/api/join)

You can also make bulk predictions by joining a table with your predictor:

```sql
SELECT t.rental_price as real_price, 
       m.rental_price as predicted_price,
       t.number_of_rooms,  t.number_of_bathrooms, t.sqft, t.location, t.days_on_market 
FROM example_db.demo_data.home_rentals as t 
JOIN mindsdb.home_rentals_model as m limit 100
```

```sql
+------------+-----------------+-----------------+---------------------+------+----------+----------------+
| real_price | predicted_price | number_of_rooms | number_of_bathrooms | sqft | location | days_on_market |
+------------+-----------------+-----------------+---------------------+------+----------+----------------+
| 3901       | 3886            | 2               | 1                   | 917  | great    | 13             |
| 2042       | 2007            | 0               | 1                   | 194  | great    | 10             |
| 1871       | 1865            | 1               | 1                   | 543  | poor     | 18             |
| 3026       | 3020            | 2               | 1                   | 503  | good     | 10             |
| 4774       | 4748            | 3               | 2                   | 1066 | good     | 13             |
+------------+-----------------+-----------------+---------------------+------+----------+----------------+
```

## What's Next?

Have fun while trying it out yourself!

* Bookmark [MindsDB repository on GitHub](https://github.com/mindsdb/mindsdb).
* Sign up for a free [MindsDB account](https://cloud.mindsdb.com/register).
* Engage with the MindsDB community on [Slack](https://mindsdb.com/joincommunity) or [GitHub](https://github.com/mindsdb/mindsdb/discussions) to ask questions and share your ideas and thoughts.

If this tutorial was helpful, please give us a GitHub star [here](https://github.com/mindsdb/mindsdb).
