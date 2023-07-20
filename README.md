# Exact
Exact is simple, secure and very fast REST API for searching structured public data with python expressions syntax. 

Or maybe it's better to call Exact as "in-memory database for public data with anonymous read access".

Or "REST API for searching records inside JSON files".

## Why to use Exact?

### Save development time and money
Maybe your project is online computer store or IMDB-like movie database. Anyway you need fast, flexible and secure search backend for it. Not just for simplest queries like "smartphones from lowest price to highest" + "smartphones of brand X and price between Y and Z" but for any complex search query. "Smartphones with price from X to Y, brand Samsung or Apple, with Retina screen, and what is min/max price?". If someone could not find specific product to buy, he could not buy it from you. 

How long to develop and debug this kind of search API (and what is estimated price)? What if you can get it in a minute? Fast, secure and very flexible search API, which is good for computer store, dating site, imdb and (*almost?*) anything. 

"Smartphones with price from X to Y, brand Samsung or Apple, with Retina screen" (`category=="smartphones" and price>1 and price<1000 and brand in ["Apple", "Samsung"] and "retina" in description.lower()`), "Movies, where Jack Nicholson played with Audrey Hepburn", "Green or red t-shirts, XXL size, cotton>80, sorted by price, min and max price".

And if later you will add more data to search, no need to modify backend, Exact already can search for it in no time. You only need to write front-end JS code to send queries to Exact.

### Secure by design: Isolation from main database
All software products are developed to be secure. Many of them are developed by brilliant high-paid programmers and security specialists. And most of them had [at least one](https://cve.mitre.org/cgi-bin/cvekey.cgi?keyword=google) vulnerability. Errare humanum est.

With Exact it's possible to isolate search backend as reliable as you want (even put it on other server without database access if you are paranoid like me). Even if (just theory) there is an vulnerability in Exact or [Evalidate](https://github.com/yaroslaff/evalidate), hacker can get access only to public data. 


## Quick start

To play with Exact, you can use our demo server at [back4app](https://www.back4app.com/) ([httpie](https://github.com/httpie/httpie) is recommended):
~~~
http POST https://exact-yaroslaff.b4a.run/ds/dummy limit=3
~~~

This is free virtual docker container, if no reply - it's sleeping, just repeat request in a few seconds and it will reply very quickly. Or run container locally.

Or if you prefer curl:
~~~
curl -H 'Content-Type: application/json' -X POST https://exact-yaroslaff.b4a.run/ds/dummy -d '{"expr": "price<800 and brand==\"Apple\""}'
~~~

(pipe output to [jq](https://github.com/jqlang/jq) to get it formatted and colored)

See - [QUERY.md](doc/QUERY.md) for example queries.

## Running your own instance (Alternative 1 (recommended): docker container)

If you want to run your own instance of exact, better to start with docker image.

Create following directory structure (/tmp/data):
~~~
mkdir -p /tmp/data/data
mkdir /tmp/data/etc

# make example dataset
wget -O /tmp/data/data/test.json https://fakestoreapi.com/products
~~~

create basic config file `/tmp/data/etc/exact.yml`:
~~~yaml
limit: 20
datadir:
  - /data/data

datasets:
  dummy:
    url: https://dummyjson.com/products?limit=100
    keypath:
      - products
    format: json
    limit: 20
~~~

This will create exact instance with two datasets, "dummy" (loaded from network) and "test" loaded from local file test.json from datadir.


Now you can start docker container:
~~~
sudo docker run --rm --name exact -p 8000:80 -it -v /tmp/data/:/data/  yaroslaff/exact
~~~

And make test query: `http POST http://localhost:8000/ds/test 'expr=price<10' limit=5`

## Running your own instance (Alternative 2: as python app)
1. Clone repo: `git clone https://github.com/yaroslaff/exact.git`
2. install dependencties: `cd exact; poetry install`
3. activate virtualenv: `poetry shell`
4. `uvicorn exact:app`

## Documentation
Please see files in `doc/`:
- [QUERY.md](doc/QUERY.md)
- [CONFIG.md](doc/CONFIG.md)
- [SECURITY.md](doc/SECURITY.md)
- [TROUBLESHOOTING.md](doc/TROUBLESHOOTING.md)

## Memory usage
Docker container with small JSON dataset consumes 41Mb (use plain python app "alternative 2", if you need even smaller memory footprint). When loading large file (1mil.json. 500+Mb), container takes 1.5Gb. Rule of thumb - container will use 3x times of JSON file size for large datasets.

## Performance
For test, we use 1mil.json file, list of 1 million of products (each of 100 unique items is duplicated 10 000 times, see below). Searching for items with `price<200` and limit=10 (820 000 matches), takes little more then 0.2 seconds. Aggregation request to find min and max price among whole 1 million dataset takes 0.43 seconds.

## Tips and tricks
- If you will always use upper/lower case in JSON datasets and in frontend, you can disable `upper`/`lower` functions and save few milliseconds on each request.
- Remove all sensitive/not-needed fields when exporting to JSON. Leave only key fields and fields used for searching, such as price, size, color.
- Use `limit` for every dataset, and set default `limit` globally in `exact.yml`. Sending your full database in response is probably never needed, but such requests will consume RAM/CPU/Bandwidth.


## MySQL, MariaDB, PostgreSQL and other databases support
Exact uses [SQLAlchemy](https://www.sqlalchemy.org/) to work with database, so it can work with any sqlalchemy-compatible RDBMS, but you need to install proper python modules, e.g. `pip install mysqlclient` (for mysql/mariadb).

https://docs.sqlalchemy.org/en/20/core/engines.html

Example config
~~~yaml
datasets:
  contact:
    db: mysql://scott:tiger@127.0.0.1/contacts
    sql: SELECT * FROM contact
~~~

This will create dataset contact from `contacts.contact` table.

## Build docker image
~~~
sudo docker build -t yaroslaff/exact ./
~~~

## Sample data sources
- https://fakestoreapi.com/products
- https://www.mockaroo.com/
- https://dummyjson.com/
- https://github.com/prust/wikipedia-movie-data/


Prepare 1 million items list '1mil.json':
~~~
$ python
Python 3.9.2 (default, Feb 28 2021, 17:03:44) 
[GCC 10.2.1 20210110] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import requests
>>> import  json
>>> data = requests.get('https://dummyjson.com/products?limit=100').json()['products'] * 10000
>>> with open('1mil.json', 'w') as fh:
...   fh.write(json.dumps(data))
... 
~~~
This makes file `1mil.json` (568Mb).
