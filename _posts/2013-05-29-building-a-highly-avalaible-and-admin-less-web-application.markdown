---
layout: post
title:  "Building a highly available and admin-less web application"
date:   2013-05-29
---

## Story behind

I work for a startup based in Paris, called tinyclues. For our daily processes, we use AWS EC2 and when possible Spot Instances. I am surprized to see that you need to be logged in to the AWS console to follow the evolution of the spot prices.

Thus, as a week-end project, I decided to build a web-app that will collect and display the history of the spot prices.
This geeky weekend second objective was also to approach unknown technologies/services.

Quickly, I understood that I needed to adopt the following constraints: the service should

- run without any admin tasks,
- be highly reliable,
- cost as little as possible.

Furthermore, as I had just returned from pycon where I met people from [cloundant](http://www.cloudant.com) and [mongoHQ](http://www.mongohq.com), I wanted to try one of these [DaaS](http://en.wikipedia.org/wiki/Data_as_a_service).


## Collecting Spot Prices

### Get Spot Prices

Spot prices are available via the AWS API. As a python developer, I use `boto` ([website](http://boto.readthedocs.org/en/latest/)) to grab those prices.

The following code is used to collect them.

```python
from datetime import datetime
from boto import ec2
ISO8601 = '%Y-%m-%dT%H:%M:%S.000Z'
def fetch_spot_prices(aws_access_key, aws_secret_key, start_time):
    """
    Grab spot prices for every available AWS region.
    `start_time` parameter should be provided with the ISO 8061 format.
    """
    end_time = datetime.utcnow().strftime(ISO8601)
    regions = ec2.regions(aws_access_key_id=aws_access_key,
                          aws_secret_access_key=aws_secret_key)
    connect = lambda region: ec2.connect_to_region(region,
                                                   aws_access_key_id=aws_access_key,
                                                   aws_secret_access_key=aws_secret_key)
    prices = []
    for region in regions:
        prices.extend([x for x in connect(region.name).get_spot_price_history(start_time=start_time, end_time=end_time)])
	return prices
```

Unfortunately, the `start_time` parameter is not taken into account (either because of a bug in `boto` or in the AWS API) resulting in the need to filter the results of the `get_spot_price_history` method.

```python
prices.extend([x for x in connect(region.name).get_spot_price_history(start_time=start_time, end_time=end_time))
```

becomes

```python
prices.extend([x for x in connect(region.name).get_spot_price_history(start_time=start_time, 
                                                                      end_time=end_time) if x.timestamp > start_time])
```

### Recording Data

Once retrieved, those price entries should be recorded.
As I already said, I wanted to use MongoHQ (and thus MongoDB) or Cloudant (and thus couchdb) as a Database provider.
MongoHQ pricing is less progressive and is tightly bound to the amount of data you want to store. As I have no plan to reduce the amount of stored data and the cost per Gb is very low on cloudant, I decided to go cloudant.
Disclaimer: at this time, my choices were entirely guided by the cost of those services. As of yet, I have not looked into a MongoDB vs CouchDB features comparison. I know this may sound strange, but low cost was a strong objective for this project.

For this reason I created an account on cloudant platform and setup an `awsspotprices` database.

In order to write on this database, I needed to generate an access key (with the corresponding secret key).
Once done, I added a function to record the returned prices list of the `fetch_spot_prices` method on my cloudant database.

```python
from couchdbkit import Document, Server, StringProperty, FloatProperty, DateTimeProperty
from restkit import BasicAuth


class SpotPriceEntry(Document):
    availability_zone = StringProperty()
    instance_type = StringProperty()
    item = StringProperty()
    price = FloatProperty()
    product_description = StringProperty()
    region = StringProperty()
    timestamp = DateTimeProperty()


def record_prices(prices, cloudant_db, cloudant_access_key, cloudant_secret_key):
    server = Server(cloudant_db, filters=[BasicAuth(cloudant_access_key,
                                                    cloudant_secret_key)])
    db = server.get_db("awsspotprices")
    SpotPriceEntry.set_db(db)
    db.save_docs([SpotPriceEntry(availability_zone=price.availability_zone,
                                 instance_type=price.instance_type,
                                 item=price.item,
                                 price=float(price.price),
                                 product_description=price.product_description,
                                 region=price.region.name,
                                 timestamp=parse_ts(price.timestamp)) for price in prices])
```

### Make things works

Finally, I had to compose the call of my two functions to record aws spot prices. Something like `record_prices(fetch_spot_prices(start_time))`.
But I still could not figure out how to get the start time.

Here, I decided to get the most recent record time, accross all instance types and all regions.
I suspect this method to be deeply *wrong* because one does not know how amazon gives its clients pricing history from one region to the next, from one instance type to the other. 
To get the latest record time, one should inspect the last record time instance type by instance type, region by region.

Computing the max timestamp is done by creating a design document on couchdb.
Couchdb has a special `_stats` reduce function that suits my needs. The design document is:

```json
"stats": {
    "map": "function(doc) {emit(doc.timestamp, Date.parse(doc.timestamp))}",
    "reduce": "_stats"
}
```

For each document, I emit the timestamp, parsed as an int.

The return document contains:

- the sum of all timestamp (meaningless),
- the min timestamp,
- the max timestamp,
- the count of all documents.

```json
{"rows":[
{"key":null,"value":{"sum":109702860597366000,"count":80197,"min":1366732939000,"max":1369084205000,"sumsqr":150064472797683882932844000000}}
]}
```

Note: for the couchdb beginner, it's not easy to understand where to store *design documents*. They should be stored in a special `_design/views` entry where each *views* should be stored in *views* key.


Then all I need is a glue to get the `start_time` value:

```python
import time

def get_start_time(cloudant_db):
    server = Server(cloudant_db, filters=[BasicAuth(cloudant_access_key,
                                                    cloudant_secret_key)])
    db = server.get_db("awsspotprices")
    ts = db.view("views/stats").fetch_raw().json_body['rows'][0]['value']['max']
    return time.strftime(ISO8601, time.gmtime(ts/1000))

def record():
    start_time = get_start_time(CLOUDANT_DB)
    prices = fetch_spot_prices(start_time, AWS_ACCESS_KEY, AWS_SECRET_KEY)
    record_prices(prices, CLOUDANT_DB, CLOUDANT_ACCESS_KEY, CLOUDANT_SECRET_KEY)
```

Last but not least, I needed to run this job periodically. [PiCloud](http://www.picloud.com) offers CRON tasks management (see [here](http://blog.picloud.com/tag/cron/)).


I should mention here the *incredible* support of both PiCloud and Cloudant. For instance, I had a problem with `couchdbkit` dependencies, and PiCloud support was very responsive and helpful.


## Building the WEB APP

At this stage, I had a regularly updated database filled with price entries. Now, all I needed to do was to plot these data.

### Requesting Data and plotting it

`d3js` is, nowadays, maybe the best library for data visualisations. A new ecosystem of Javascript libraries is built on it.
I chose one of these libraries which was specialized in displaying time series: `rickshaw`.

My needs were perfectly suited by this [example](http://code.shutterstock.com/rickshaw/examples/extensions.html) of the library that can be found on the `rickshaw` [website](http://code.shutterstock.com/rickshaw).

If you're looking at the [awsspotprices website](http://www.awsspotprices.com), you'll see that the code is almost identical.

### Serving stuff

My whole application holds in one HTML page. I serve it from a AWS S3 bucket and use the AWS Route 53 DNS server to make [awsspotprices](http://www.awsspotprices.com) available.

## Afterword + pricing estimation

I hope someone will find this application useful. I use it almost everyday to follow AWS pricing history.
Here is a pricing estimation of this service:

- PiCloud usage: ~10 hours computation : *free*
- Cloudant usage: ~100 POSTs per day ($0.015 for 100 POSTS) + 0.05$ of storage/per month + ~100 GETs per day ($0.015 for 500 requests) is far from the 5$ free usage limit (*more than 100000 GETs per month are needed for me to have to pay anything* :))
- AWS usage: *a few cents/per month*
  - S3 $0,095/Go (the served files took 20ko) + $0,004 for 10 000 GET:  
  - Route53: $0.50 per hosted zone + $0.50 for 1 million queries
- Domain Name: 15$

Finally, with domain name registration, this service will work for less than 2$ per month.

More resources about AWS Spot pricing:

- [AWS Pricing Overview](http://d36cz9buwru1tt.cloudfront.net/AWS_Pricing_Overiew.pdf)
- [Deconstructing Amazon EC2 Spot Instance Pricing](http://www.cs.technion.ac.il/~dan/papers/Spotprice11CloudCom.pdf)

## Edit (2013-06-01)



