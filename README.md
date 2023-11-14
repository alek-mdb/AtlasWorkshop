# Atlas Workshop
[Session slides](Atlas%20Workshop%20__%20Morgan%20StanleyMTL%20__%2016%20Nov%2023.pdf)

## Prerequisites
Workshop attendees MUST bring their own PERSONAL LAPTOP in order to actively participate in the workshop. Please make sure the laptop is FULLY CHARGED.

Please create a free user account on MongoDB Atlas. Accounts MUST be created using your personal email address, NOT your Morgan Stanley email.

Attendees must connect to the Personal-wifi on their personal laptop for the workshop.

## Steps
### 1. Add sample data to your MongoDB Atlas cluster

If you haven't already sone so, [create an Atlas cluster](https://cloud.mongodb.com/).

Add sample data:
- Click the `...` next to your cluster in the Atlas UI
- Select `Load Sample Dataset`

It will take a few minutes for the sample data to be added.

## MongoDB Queries
### 1. Atlas UI

Navigate to Collections tab in Data Explorer by clicking `Browse Collections` in your cluster. We will be using the ```sample_airbnb``` collection. 

#### a. Query 1

Find all properties in the database with text filter:
Market in Hong Kong: {"address.market":"Hong Kong" } and is an Apartment : {"property_type" :"Apartment"}
To specify equality conditions, use : expressions in the query filter document.
```json
{"address.market":"Hong Kong", "property_type" : "Apartment"}
```

#### b. Query 2

Find all properties in the database with a range based filter:
have at least three bedrooms: {"bedrooms": {"$gte" : 3}}  and price range from 1,300 to 1,500: "price": {"$gte" : 3}}  and price range from 1,300 to 1,500: "price": {"$gte": 1300, "$lte": 1500}
```json
{ "bedrooms": {"$gte": 3}, "price": {"$gte": 1400, "$lte": 1500}}
```

#### c. Query 3

Find all properties in the database with a field that is inside an array:
Has both Wifi and Kitchen: { "amenities": { "$all": ["Wifi", "Kitchen"] }}
When specifying compound conditions on array elements, you can specify the query such that either a single array element meets these condition or any combination of array elements meets the conditions.
```json
{ "amenities": { "$all": ["Wifi", "Kitchen"]}}
```


#### d. Query 4

Find all properties in the database with combination of elements:
Market in Hong Kong: {"address.market":"Hong Kong" } and is an Apartment : {"property_type" :"'Apartment"}
Has at least three bedrooms: {"bedrooms": {"gte" : 3}} and price range from 1,300 to 1,500: "price": {"gte" : 3}} and price range from 1,300 to 1,500: "price": {"gte": 1300, "$lte": 1500}
Has both Wifi and Kitchen: { "amenities": { "$all": ["Wifi", "Kitchen"] }}
```json
{"address.market":"Hong Kong", "property_type" : "Apartment", "bedrooms": {"$gte": 3}, "price": {"$gte": 1400, "$lte": 1500}, "amenities": { "$all": ["Wifi", "Kitchen"]}}
```

#### e. Query 5 (Applicable for Compass or CMD Line)

In this example, run .explain() to returns the queryPlanner information for the evaluated method. Navigate to the Explain tab in compass to see query execution. Alternatively, use the command line. 
```js
db.listingsAndReviews.find({"address.market":"Hong Kong", "property_type" : "Apartment", "bedrooms": {"$gte": 3}, "price": {"$gte": 1400, "$lte": 1500}, "amenities": { "$all": ["Wifi", "Kitchen"]}}).explain()
```


## Aggregation Framework
### 1. Atlas UI

Still in the Data Explorer, click the smaller `Aggregation` tab to view the Aggregation Builder. We will be using the ```sample_airbnb``` collection. 

#### a. Query 1

Calculate how many extra guests you can have for each property and add that as a field using $set. Calculate how much it would cost with these extra guests. $project the basic price and the price if fully occupied with $accommodates people. 

```
[
  {
    $set: {
      numExtraGuests: {
        $subtract: [
          "$accommodates",
          "$guests_included",
        ],
      },
    },
  },
  {
    $set: {
      extraGuestCost: {
        $multiply: [
          "$extra_people",
          "$numExtraGuests",
        ],
      },
    },
  },
  {
    $project: {
      price: 1,
      maxprice: {
        $add: ["$price", "$extraGuestCost"],
      },
    },
  },
]
```

#### b. Query 2

Calculate how many Airbnb properties there are in each country, ordered by count, list the one with the largest count first.

```
[
  {
    $group: {
      _id: "$address.country",
      count: {
        $sum: 1,
      },
    },
  },
  {
    $sort: {
      count: -1,
    },
  },
]
```

#### c. Query 3

What are the top 10 most popular amenities across all listings?

```
[
  {
    $unwind: {
      path: "$amenities",
    },
  },
  {
    $group: {
      _id: "$amenities",
      count: {
        $sum: 1,
      },
    },
  },
  {
    $sort: {
      count: -1,
    },
  },
]
```

HOMEWORK: We encourage you to checkout this [book](https://www.practical-mongodb-aggregations.com/) to learn more about aggregations. This is a free resource available to the general public. We would love to hear your feedback and use cases on how you use Aggregation Framework in the future. 


## Search
### 1. Navigate to the Search Index configuration page on the Atlas UI 

### 2. Add default Search index
- Keep the name as `default`
- Databse: `sample_mflix`
- Collection: `movies`
- Leave `Dynamic Mapping` turned on

### 3. Add autocomplete index
- Set the name to `autocomplete`
- Click `Refine Your Index`
- Turn off `Dynamic Mapping`
- Add a new field mapping for the `title` field with a `Data Type` of `Autocomplete`

### 4. Create aggregation pipeline to search for movies
From the Atlas UI, navigate to the `Aggregation` pane for `sample_mflix.movies`.

Create these stages in sequence:

#### a. `$search`
```js
{
  index: 'default',
  text: {
    query: 'Indiana Ones',
    path: ['plot', 'fullplot', 'title'],
    fuzzy: {}
  },
  highlight: {path: 'fullplot'}
}
```

#### b. `$limit`
```
12
```

#### c. `$project`
```js
{
  title: 1,
  year: 1,
  poster: 1,
  plot: 1, 
  imdb: 1,
  score: {$meta: 'searchScore'},
  highlights: {
    '$meta': 'searchHighlights'
  }
}
```

- Click `EXPORT TO LANGUAGE`
- Choose `NODE` as the export language
- Copy the code and keep it safe
- Save the pipeline if using Compass


### 6. Create aggregation pipeline to autocomplete movie titles
From the Atlas UI, navigate to the `Aggregation` pane for `sample_mflix.movies`.

Create these stages in sequence:

#### a. `$search`
```js
{
  index: 'autocomplete',
  autocomplete: {
    query: 'Indiana J',
    path: 'title'
  }
}
```
#### b. `$limit`
```js
10
```
#### c. `$project`
```js
{
  _id: 0,
  title: 1
}
```

## Final demo

A quick demo of some of the features of Atlas Search (including the queries) using [Restaurant Finder](https://www.atlassearchrestaurants.com/). Code can be found [here](https://github.com/mongodb-developer/WhatsCooking).
