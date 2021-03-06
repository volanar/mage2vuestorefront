# mage2vuestorefront
For those who would love to work with Magento on backend but use NoSQL power on the frontend. Two way / real time data synchronizer.

It's part of [vue-storefront project - first Progressive Web App for eCommerce](https://github.com/DivanteLtd/vue-storefront) with Magento2 support.
Some details about the rationale and our goals [here](https://www.linkedin.com/pulse/magento2-nosql-database-pwa-support-piotr-karwatka)

It synchronizes all the products, attributes, taxrules, categories and links between products and categories.

This is multi-process data synchronizer between Magento (and in further versions Shopify / other platforms) to local MongoDB AND ElasticSearch databases.

At this point synchronization works with following entities:
- Products
- Categories
- Taxrules
- Attributes
- Product-to-categories
- Reviews (require custom module Divante/ReviewApi to work)

Categories and Product-to-categories links are additionaly stored in Redis cache for rapid-requests (for example from your WebAPI). Our other project [vue-storefront-api](https://github.com/DivanteLtd/vue-storefront-api) exposes this databse to be used in PWA/JS webapps.

Datasync uses oauth + magento2 rest API to get the data.
KUE is used for job queueing and multi-process/multi-tenant processing is enabled by default
MongoDB OR ElasticSearch is used for NoSQL database
Redis is used for KUE queue backend

By default all services are used without authorization and on default ports (check out config.js or ENV variables for change of this behavior). 


**Tutorial on installation / integration [manual for Vue Storefront connectivity](https://medium.com/@piotrkarwatka/vue-storefront-how-to-install-and-integrate-with-magento2-227767dd65b2)**


## How to perform full / initial import for Vue Storefront

To get started with VS we must start with some very basics about the architecture; the project is backed by three separate Node.js applications

### Vue Storefront Architecture
[vue-storefront (Github)](https://github.com/DivanteLtd/vue-storefront) — is the main project where you can also find most of the documentation, issues mapped to further releases and other resources to start with — Vue.js on webpack.

[vue-storefront-api (Github)](https://github.com/DivanteLtd/vue-storefront-api) — is the API layer which provides the data to vue-storefront app — Node.js, Express; This project consist of docker instances for Redis and ElasticSearch required by mage2vuestorefront and pimcore2vuestorefront

mage2vuestorefront — THIS project -data bridges which are in charge of moving data back from Magento2 to Vue Storefront data store.

You must install `vue-storefront-api` locally. You may install it using the Vue Storefront installer - see details. Or manually by executing the sequence of commands:

```bash
git clone https://github.com/DivanteLtd/vue-storefront-api
cd vue-storefront-api
npm install
npm run migrate
docker-compose up -d
npm run dev
```

The key command is `docker-compose up -d` which runs the ElasticSearch and Redis instances - both required by `mage2vuestorefront`

### Initial Vue Storefront import

Now, You're ready to run the importer. Please check the [config file](https://github.com/DivanteLtd/mage2vuestorefront/blob/master/src/config.js). You may setup the Magento access data and URLs by config values or ENV variables.

We'll use in the following example - the ENV variables. The simplest command sequence to perform full reindex is:

```bash
export TIME_TO_EXIT=2000
export MAGENTO_CONSUMER_KEY=byv3730rhoulpopcq64don8ukb8lf2gq
export MAGENTO_CONSUMER_SECRET=u9q4fcobv7vfx9td80oupa6uhexc27rb
export MAGENTO_ACCESS_TOKEN=040xx3qy7s0j28o3q0exrfop579cy20m
export MAGENTO_ACCESS_TOKEN_SECRET=7qunl3p505rubmr7u1ijt7odyialnih9

echo 'Default store - in our case United States / en'
export MAGENTO_URL=http://demo-magento2.vuestorefront.io/rest
export INDEX_NAME=vue_storefront_catalog

node --harmony cli.js categories --removeNonExistent=true
node --harmony cli.js productcategories --partitions=1
node --harmony cli.js attributes --removeNonExistent=true
node --harmony cli.js taxrule --removeNonExistent=true
node --harmony cli.js products --removeNonExistent=true --partitions=1
node --harmony cli.js reviews
```

Please note:
- `--removeNonExistent` option means - all records that were found in the index but currently don't exist in the API feed - will be removed. Please use this option ONLY for the full reindex!
- `INDEX_NAME` by default is set to the `vue_storefront_catalog` but You may set it to any other elastic search index name.

**Cache invalidation:** Recent version of Vue Storefront do support output caching. Output cache is being tagged with the product and categories id (products and categories used on specific page). Mage2vuestorefront can invalidate cache of product and category pages if You set the following ENV variables:

```bash
export VS_INVALIDATE_CACHE_URL=http://localhost:3000/invalidate?key=aeSu7aip&tag=
export VS_INVALIDATE_CACHE=1
```

- `VS_INVALIDATE_CACHE_URL` is a cache to the Vue Storefront instance - used as a webhook to clear the output cache.

Please note:
After data import - especially when You're not sure about the product attributes data types - please **reindex** ElasticSearch to estaplish the correct / current database schema. You may do this using [Database tool](https://github.com/DivanteLtd/vue-storefront/blob/master/doc/Database%20tool.md) in the `vue-storefront-api` folder:

```bash
cd vue-storefront-api
npm run db rebuild -- --indexName=vue_storefront_catalog
```

If You like to create a new, emtpy index please run:
```bash
cd vue-storefront-api
npm run db new -- --indexName=vue_storefront_catalog
```

### Delta indexer

After initial setup and full-reindex You may want to add indexer to the `crontab` to index only modified product records.
This is fairly easy - You just need to add the following command to crontab:

```bash
node --harmony cli.js productsdelta --partitions=1
```

This command will execute full reindex at first call - and then will be storing the last index date in the `.lastIndex.json` and downloading only these products which have `updated_at` > last index date.

Please note: Magento2 has a bug with altering `updated_at` field. Please install [a fix for that](https://github.com/codepeak/magento2-productfix) before using this method: 

```bash
composer require codepeak/magento2-productfix
php bin/magento cache:flush
```

### On-demand indexer (experimental!)

Mage2nosql supports an on-demand indexer - where Magento calls a special webhook to update modified products.
In the current version the webhook notifies mage2vuestorefront about changed product SKUs and mage2vuestorefront pulls the modified products data via Magento2 APIs.

First. You should install [Magento2 module called VsBridge](https://github.com/DivanteLtd/magento2-vsbridge)
Second. Deploy mage2vuestorefront on the server.

Then You may want to start a webapi process:

```bash
cd mage2vuestorefront/src
export TIME_TO_EXIT=2000
export MAGENTO_CONSUMER_KEY=byv3730rhoulpopcq64don8ukb8lf2gq
export MAGENTO_CONSUMER_SECRET=u9q4fcobv7vfx9td80oupa6uhexc27rb
export MAGENTO_ACCESS_TOKEN=040xx3qy7s0j28o3q0exrfop579cy20m
export MAGENTO_ACCESS_TOKEN_SECRET=7qunl3p505rubmr7u1ijt7odyialnih9
export PORT=6060
export MAGENTO_URL=http://demo-magento2.vuestorefront.io/rest

node --harmony webapi.js
```

The API will be listening on port 6060. Typically non-standard ports like this one are not exposed on the firewall. Please consider setting up [simple nginx proxy for this service](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-node-js-application-for-production-on-ubuntu-16-04).

Anyway - this API must be publicly available via Internet OR You must have the mage2vuestorefront installed on the same machine like Magento2.

Go to Your Magento2 admin panel, then to Stores -> Configuration -> VsBridge and set-up "Edit product" url to: `http://localhost:6060/magento/products/update`. **Please note:** Product delete endpoint hasn't been implemented yet and it's good chance for Your PR.

After having the webapi up and runing and this endpoint set, any Product save action will call `POST http://localhost:6060/magento/products/update` with the body set to `{"sku": ["modified-sku-list"]}`.

Webapi will [add the products to the queue](https://github.com/DivanteLtd/mage2vuestorefront/blob/b44e7ede9aeb27f308e2a87033251a2491640da8/src/api/routes/magento.js#L19).

Please run the queue worker to process all the queued updates (You may run multiple queue workers even distributed across many machines):

```bash
cd mage2vuestorefront/src
export TIME_TO_EXIT=2000
export MAGENTO_CONSUMER_KEY=byv3730rhoulpopcq64don8ukb8lf2gq
export MAGENTO_CONSUMER_SECRET=u9q4fcobv7vfx9td80oupa6uhexc27rb
export MAGENTO_ACCESS_TOKEN=040xx3qy7s0j28o3q0exrfop579cy20m
export MAGENTO_ACCESS_TOKEN_SECRET=7qunl3p505rubmr7u1ijt7odyialnih9
export PORT=6060
export MAGENTO_URL=http://demo-magento2.vuestorefront.io/rest

node --harmony cli.js productsworker
```

**Please note:** We're using [kue based on Redis queue](https://github.com/Automattic/kue) which may be configured via `src/config.js` - `kue` + `redis` section.
**Please note:** There is no authorization mechanism in place for the webapi calls. Please keep it local / private networked or add some kind of authorization as a PR to this project please :-)


### Multistore setup

Multiwebsite support starts with the ElasticSearch indexing. Basically - each store has it's own ElasticSearch index and should be populated separately using [mage2vuestorefront](https://github.com/DivanteLtd/mage2vuestorefront) tool.

The simplest script to index multi site:

```bash
export TIME_TO_EXIT=2000
export MAGENTO_CONSUMER_KEY=byv3730rhoulpopcq64don8ukb8lf2gq
export MAGENTO_CONSUMER_SECRET=u9q4fcobv7vfx9td80oupa6uhexc27rb
export MAGENTO_ACCESS_TOKEN=040xx3qy7s0j28o3q0exrfop579cy20m
export MAGENTO_ACCESS_TOKEN_SECRET=7qunl3p505rubmr7u1ijt7odyialnih9

echo 'German store - de'
export MAGENTO_URL=http://demo-magento2.vuestorefront.io/rest/de
export INDEX_NAME=vue_storefront_catalog_de

node --harmony cli.js categories --removeNonExistent=true
node --harmony cli.js productcategories --partitions=1
node --harmony cli.js attributes --removeNonExistent=true
node --harmony cli.js taxrule --removeNonExistent=true
node --harmony cli.js products --removeNonExistent=true --partitions=1

echo 'Italian store - it'
export MAGENTO_URL=http://demo-magento2.vuestorefront.io/rest/it  
export INDEX_NAME=vue_storefront_catalog_it

node --harmony cli.js categories --removeNonExistent=true
node --harmony cli.js productcategories --partitions=1
node --harmony cli.js attributes --removeNonExistent=true
node --harmony cli.js taxrule --removeNonExistent=true
node --harmony cli.js products --removeNonExistent=true --partitions=1

echo 'Default store - in our case United States / en'
export MAGENTO_URL=http://demo-magento2.vuestorefront.io/rest
export INDEX_NAME=vue_storefront_catalog

node --harmony cli.js categories --removeNonExistent=true
node --harmony cli.js productcategories --partitions=1
node --harmony cli.js attributes --removeNonExistent=true
node --harmony cli.js taxrule --removeNonExistent=true
node --harmony cli.js products --removeNonExistent=true --partitions=1
```

As You may see it's just a **it** or **de** store code which is added to the base Magento2 REST API urls that makes the difference and then the **INDEX_NAME** set to the dedicated index name.

In the result You should get:
- *vue_storefront_catalog_it* - populated with the "it" store data
- *vue_storefront_catalog_de* - populated with the "it" store data
- *vue_storefront_catalog* - populated with the "default" store data

Then, to use these indexes in the Vue Storefront You should index the database schema using the `vue-storefront-api` db tool:

```bash
npm run db rebuild -- --indexName=vue_storefront_catalog_it
npm run db rebuild -- --indexName=vue_storefront_catalog_de
npm run db rebuild -- --indexName=vue_storefront_catalog
```

More on <a href="https://github.com/DivanteLtd/vue-storefront/blob/master/doc/Multistore%20setup.md">how to setup Vue Storefront in the Multistore mode</a>.

### Indexing configurable products attributes for filters

If You like to have Category filter working with configurable products - You need to expand the `product.configurable_children.attrName` to `product.attrName_options` array. This is automatically done by [mage2vuestorefront](https://github.com/DivanteLtd/mage2vuestorefront) for all attributes set as `product.configurable_options` (by default: color, size). If You like to add additional fields like `manufacturer` to the filters You need to expand `product.manufacturer_options` field. The easiest way to do so is to set `config.product.expandConfigurableFilters` to `['manufacturer']` and re-run the `mage2vuestorefront` indexer.


## Advanced usage

Start Elasticsearch and Redis:
- `docker-compose up`

Install:
- `cd src/`
- `npm install`

Config - see: config.js or use following ENV variables: 
- `MAGENTO_URL`
- `MAGENTO_CONSUMER_KEY`
- `MAGENTO_CONSUMER_SECRET`
- `MAGENTO_ACCESS_TOKEN`
- `MAGENTO_ACCESS_TOKEN_SECRET`
- `DATABASE_URL` (default: 'mongodb://localhost:27017/rcom')


Run:
- `cd src/` and then:
- `node --harmony cli.js fullreindex` - synchronizes all the categories, products and links between products and categories

Other commands supported:
- `node --harmony cli.js products --partitions=10`
- `node --harmony cli.js products --partitions=10 --initQueue=false` - run the products sync worker (product sync jobs should be populated eslewhere - it's used to run multi-tenant environment of workers)
- `node --harmony cli.js products --partitions=10 --delta=true` - check products changed since last run (last run data is stored in mongodb); compared by updated_at field
- `node --harmony cli.js productcategories` - to synchronize the links between products and categories it *should be run before* products synchronization because it populates Redis cache assigments for product-to-category link
- `node --harmony cli.js categories`
- `node --harmony cli.js products --adapter=magento --partitions=1 --skus=24-WG082-blue,24-WG082-pink`  - to pull out only selected SKUs
- `node --harmony cli.js productsworker --adapter=magento --partitions=10`  - run queue worker for pulling out individual products (jobs can be assigned by webapi.js microservice triggers; it can be called by webhook for example from within Magento2 plugin)
- `node --harmony webapi.js` - run localhost:3000 service endpoint for adding queue tasks

WebAPI:
- `node --harmony webapi.js`
- `curl localhost:8080/api/magento/products/pull/WT09-XS-Purple` - to schedule data refresh for SKU=WT09-XS-Purple
- `node --harmony cli.js productsworker` - to run pull request processor 

Available options:
- `partitions=10` - number of concurent processes, by default number of CPUs core given
- `adapter=magento` - for now only Magento is supported
- `delta` - sync products changed from last run
- command names: `products` / `attributes` / `taxrule` / `categories` / `productsworker` / `productcategories`


