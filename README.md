# bptf-autopricer
<div align="center"><img src="https://github.com/jack-richards/bptf-autopricer/assets/58331725/203fe808-30ff-4d7d-868c-a3ef6d31497d" alt="logo" style="width: 280px; height: 320px; display: block; margin-left: auto; margin-right: auto;"></div>

#
<div align="center">
  
[![Version](https://img.shields.io/github/v/release/jack-richards/bptf-autopricer.svg)](https://github.com/jack-richards/bptf-autopricer/releases)
[![GitHub forks](https://img.shields.io/github/forks/jack-richards/bptf-autopricer)](https://github.com/jack-richards/bptf-autopricer/network/members)
[![GitHub Repo stars](https://img.shields.io/github/stars/jack-richards/bptf-autopricer)](https://github.com/jack-richards/bptf-autopricer/stargazers)
[![GitHub issues](https://img.shields.io/github/issues/jack-richards/bptf-autopricer)](https://github.com/jack-richards/bptf-autopricer/issues)
[![License](https://img.shields.io/github/license/jack-richards/bptf-autopricer.svg)](https://opensource.org/licenses/MIT)
[![Known Vulnerabilities](https://snyk.io/test/github/jack-richards/bptf-autopricer/badge.svg)](https://snyk.io/test/github/jack-richards/bptf-autopricer)

</div>

An open-source solution that generates item prices for selected items by utilising listing data from [backpack.tf](https://backpack.tf/). Each price created is a result of the meticulous evaluation of both the underlying data and the actual prices, incorporating multiple checks and balances to counteract potential price manipulation.

## Features

- **Automated Pricing:** Automatically generates item prices using both real-time and snapshot [backpack.tf](https://backpack.tf/) listing data, ensuring a profit margin and performing various sanity checks.

- **Baseline Comparison:** Compares generated prices against those from [Prices.tf](https://github.com/prices-tf) - disregarding prices that go over percentage thresholds configured.

- **Trusted/Blacklisted Steam IDs:** Prioritises listings from trusted bots, and filters out untrusted bots when calculating prices. Fully configurable.

- **Excluded Listing Descriptions:** Filters out listings with descriptions containing configured keywords. Useful for removing listings from calculations that include special attributes, such as spells.

- **Filters Outliers:** With a sufficient number of listings, filters out outliers from the mean. Removes listings with prices that deviate too much from the average.
  
- **API Functionality:**
  - *Add and Delete Items:* The API can be used to add or remove items for the auto pricer to track.
  - *Retrieve Prices:* Prices can be requested and retrieved through the API.

- **Socket.IO Server:**
  - Emits item prices to any listeners using a Socket.IO server.
  - Prices are stored and emitted in a format fully supported by the [TF2 Auto Bot](https://github.com/TF2Autobot/tf2autobot) custom-pricer interface.

## Requirements
- Install dependencies by running npm install in the project directory with package.json.
- A PostgreSQL database is required. Create a database and schema according to your preferences and specify them in the config.json file.
- The PostgreSQL table named listings must be created. Use the following SQL statement:
```sql
CREATE TABLE listings (
    name character varying NOT NULL,
    sku character varying NOT NULL,
    currencies json NOT NULL,
    intent character varying NOT NULL,
    updated bigint NOT NULL,
    steamid character varying NOT NULL,
    CONSTRAINT listings_pkey PRIMARY KEY (name, sku, intent, steamid)
);
```
Make sure to specify the database name, schema, user, and other relevant details in the `config.json` file.

## Configuration
To configure the application you need to specify the values for all the fields in `config.json`.
```JSON
{
    "bptfAPIKey": "your bptf api key",
    "bptfToken": "your bptf token",
    "steamAPIKey": "your steam api key",
    "database": {
        "schema": "tf2",
        "host": "localhost",
        "port": 5432,
        "name": "backpacktf-ws",
        "user": "postgres",
        "password": "database password"
    },
    "pricerAPIPort": 3456,
    "pricerSocketPort": 9850,
    "maxPercentageDifferences": {
        "buy": 5,
        "sell": -8
    },
    "excludedSteamIDs": [
        "76561199384015307"
    ],
    "trustedSteamIDs": [
        "76561199110778355"
    ],
    "excludedListingDescriptions": [
        "exorcism",
    ]
}
```
The majority of these fields are self-explanatory. I will explain the ones that may not be.

### `maxPercentageDifferences`
Contains two fields, `buy` and `sell`. These values represent the maximum difference from the baseline (prices.tf) you will accept before a price is rejected. Adjust the maximum percentage differences for buy and sell according to your preferences.
- A higher **buy** percentage means that the auto pricer is willing to buy items for a higher price than prices.tf.
- A lower **sell** percentage means that the auto pricer is willing to sell items for a lower price than prices.tf.
  
```JSON
"maxPercentageDifferences": {
  "buy": 5,
  "sell": -8
}
```

### `excludedSteamIDs`
A list of Steam ID 64s that you don't want to use the listings of in pricing calculations. This is useful for stopping bad actors that you know of from attempting to manipulate the prices created. It's important to note that this is not a fool-proof solution and because of this other methods are also used to reduce the risks of price manipulation.

```JSON
"excludedSteamIDs": [
    "76561199384015307"
]
```

### `trustedSteamIDs`
A list of Steam ID 64s used to prioritise listings owned by these IDs over others during pricing calculations. This feature is beneficial when you want to give preference to bots or users that consistently provide accurate pricing data.

```JSON
"trustedSteamIDs": [
    "76561199110778355"
],
```

### `excludedListingDescriptions`
A list of descriptions that, when detected within a listing's details, causes the listing to be disregarded and not used during pricing calculations. This is useful for excluding listings involving items with 'special attributes,' such as spells, when calculating prices. Leaving such listings in the calculations can risk affecting the average price in unexpected ways.

```JSON
"excludedListingDescriptions": [
    "exorcism",
]
```

## API Routes
Here I'll highlight the different API routes you can make queries to, and what responses you can expect to receive.\
Please note that the API runs locally (localhost) on the port defined in `config.json` in the `pricerAPIPort` key.
#
```plain text
GET /api/:sku
```
Retrieves a particular item object from the pricelist using the Stock Keeping Unit (SKU) provided. Item object returned contains the prices for the item.

**Request:**
- **Parameters:**
  - `sku` (String): The Stock Keeping Unit of the item. E.g., Mann Co. Supply Crate Key has an SKU of 5021;6.

**Response:**
- **Success (200):**
  - JSON object containing information about the item, including the prices.
- **Failure (404):**
  - If the requested item is not found in the pricelist.
- **Failure (400):**
  - Where 5021;6 is the SKU provided and there is an issue with fetching the key price from Prices.tf.
#
```plain text
GET /api/
```
Retrieves the entire pricelist.

**Response:**
- **Success (200):**
  - JSON object containing the entire pricelist.
- **Failure (400):**
  - If there is an issue loading the pricelist.
#
```plain text
POST /api/:sku
```
An endpoint that returns a status code of 200 for each request. Exists so there's no issue in integrating with TF2 Auto Bot.

**Request:**
- **Parameters:**
  - `sku` (String): The Stock Keeping Unit of the item.

**Response:**
- **Success (200):**
  - JSON object indicating the SKU.
#
```plain text
POST /api/add/:name
```
Adds the item to the list of items to auto price.

**Request:**
- **Parameters:**
  - `name` (String): The name of the item to add.

**Response:**
- **Success (200):**
  - Item successfully added. Will now be automatically priced.
- **Failure (400):**
  - If the item already exists in the item list.
#
```plain text
POST /api/delete/:name
```
Deletes an item from the list of items to automatically price.

**Request:**
- **Parameters:**
  - `name` (String): The name of the item to delete.

**Response:**
- **Success (200):**
  - Item successfully deleted.
- **Failure (400):**
  - If the item does not exist in the item list.
    
## Running the Auto Pricer
Once all the requirements have been met, and you have provided the values required in config.json, simply run:
```
node bptf-autopricer.js
```
