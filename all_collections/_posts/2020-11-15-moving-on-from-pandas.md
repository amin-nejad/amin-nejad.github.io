---
title: "Moving on from Pandas"
date: 2020-11-15 23:49:16
categories: ["pandas", "spark"]
---

## The problem with pandas

Okay, so `pandas` is great. It's often one of the first libraries people use when learning python for data science/ machine learning. Something along the lines of the below might seem familiar...

```python
import pandas as pd
data = pd.read_csv("/path/to/data")
```

And it's great for what it does - it provides a powerful and versatile framework for reading, analysing and manipulating large quantities of data. Not to mention it has a very large and active community providing plenty of support. In fact it is the most popular python library and its popularity is only growing:

<a href="https://stackoverflow.blog/2017/09/14/python-growing-quickly" target="_blank">
<img class="blog_image" src="/assets/images/pandas-popularity.png">
</a>

And this will likely suffice for most people, for most problems, most of the time. But if you are working in industry trying to build a scalable data-first product, or train a deep neural network on a _massive_ dataset, pandas simply won't cut it. At least not if you care about speed. It is limited to one CPU core at a time and suffers from huge performance problems when you start to work with :rotating_light: **big data** :rotating_light: (sorry). This is in spite of the vectorised operations (i.e. super fast C++ code) pandas uses under the hood. Matters are exacerbated when you mistakenly end up using significantly slower python loops when performing operations - something that's very easy to do and a very common class of mistakes when working with pandas.

And this is all if you can even load your data in the first place, since the data cannot be larger than the available RAM on your machine. Sure, there are various workarounds for the memory issue but none of them are ideal. If you are working on a virtual machine, and your use case is a one-off, it might be just be simpler to increase the RAM on your machine, 100GB... 1TB... whatever you need. That will almost certainly be the quickest solution. Alternatively you can try to follow some of the [official recommendations](https://pandas.pydata.org/docs/user_guide/scale.html) on the pandas website such as using more efficient data types, chunking your dataset, etc.

But eventually there there may come a point where these workarounds will be insufficient for the sheer size of data you are dealing with or simply not scalable enough for what you are trying to build. This will become increasingly true as datasets get bigger and bigger, particularly for building machine learning models. This is when you need to start looking further afield.

## What is Spark?

[Apache Spark](https://spark.apache.org) is, in practice, the primary solution to this problem. Originally developed at UC Berkeley in 2009, Spark is a _"unified analytics engine for large-scale data processing."_. Essentially what this means is that it can perform operations on _very_ large datasets and it does this by distributing processing tasks across multiple servers. It does this so well that it has been deployed at scale by some of the [world's largest companies](https://spark.apache.org/powered-by.html) including the likes of [Apple](https://databricks.com/session/apple-talk), [Facebook](https://databricks.com/session/scaling-apache-spark-at-facebook), [Netflix](https://databricks.com/session/netflixs-recommendation-ml-pipeline-using-apache-spark) and so on.

Spark can be broken down into two fundamental components: the **driver** and the **executors**. The driver converts the user's spark code into multiple tasks that can be distributed across _worker nodes_ whilst the executors run on these worker nodes and execute the tasks assigned to them. Together these worker nodes form what's known as a _cluster_ over which our data in the form of what's known as a Resilient Distributed Dataset (RDD) is well, distributed. The RDD is an immutable, schema-less, high level abstraction which represents our data partitioned across a cluster. The idea is that operations on RDDs can be split across the cluster and executed in parallel making them super fast. They also employ lazy evaluation - unlike pandas - which means they will only execute your code right before they need to return so they can construct an optimal execution plan. Much like pandas dataframes however, RDDs can be created from a wide variety of data sources such as CSV/JSON/text files, SQL/NoSQL databases, Amazon S3 buckets and much more.

Much of the Spark Core API is built on this RDD concept, enabling traditional [map and reduce](https://en.wikipedia.org/wiki/MapReduce) functionality, but also providing built-in support for joining data sets, filtering, sampling, aggregation and so on - pretty much everything you would want to do in Pandas. Spark also offers two other data structures known as `DataFrames` and `DataSets` which are built on top of RDDs for specific use cases. Although written in Scala, Spark also provides APIs in multiple languages, the most popular of which is Python, and is known as `pyspark`.

### Setting up a cluster

The cluster manager is responsible for spawning and managing the worker nodes that each run an executor process to do the actual work. One of the key advantages of the Spark design is that the cluster manager is completely decoupled from your application and completely interchangeable. Spark ships with its own cluster manager that you can easily use out of the box but it leaves a lot to be desired. Thankfully, as of Spark 2.3.0, we can use `kubernetes` or `docker swarm` directly as a cluster manager instead! I'm not going to repeat what's already been done so feel free to take a look at [this tutorial](https://towardsdatascience.com/ignite-the-spark-68f3f988f642) (or one of many others) to do just that.

To connect to the cluster you just created using `pyspark`:

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .master("<cluster-IP-address>") \
    .appName("name-of-your-app") \
    .config("spark.some.config.option", "some-value") \
    .getOrCreate()

# perform whatever computation you want here
```

### Koalas by Databricks

Databricks is a company founded in 2013 by the creators of Apache Spark and provides a web platform for managing essentially all things Spark related. In 2019, they released an open source python library called [`koalas`](https://github.com/databricks/koalas) as an alternative to pyspark for interacting with Spark. It has very quickly garnered a lot of traction in the data science community and as of June 2020, [represented 20%](https://databricks.com/blog/2020/06/24/introducing-koalas-1-0.html) of all pyspark downloads (which it has as a dependency)! Koalas essentially provides a pandas API for Spark and avoids users having to learn the somewhat clunky pyspark API. This is a huge value proposition given that most users will already be familiar with pandas. The pandas API is often a lot more succinct too. For example the following 5 lines of code in pyspark can be converted into just 1 line of code in koalas (or pandas):

#### One hot encoding in pyspark:

```python
from pyspark.ml.feature import StringIndexer,OneHotEncoderEstimator

indexer = StringIndexer(inputCol="col_to_encode", outputCol="col_index")
data_indexed = indexer.fit(data).transform(data)
encoder = OneHotEncoderEstimator(inputCols=["col_index"], outputCols=["col_encoding"])
model = encoder.fit(data_indexed)
data_encoded = model.transform(data_indexed)
```

#### One hot encoding in koalas:

```python
import databricks.koalas as ks

data_encoded = ks.get_dummies(data=data, columns=["col_to_encode"])
```

No wonder koalas has become so popular! However, the project is still relatively new meaning pandas API [coverage is only at ~80%](https://databricks.com/blog/2020/06/24/introducing-koalas-1-0.html) (at the time of writing), with some edge cases not properly dealt with in my experience and documentation not quite as detailed as what we have come to expect from pandas. Nevertheless, for anyone wanting to use spark from a pandas background, this is the library I would recommend since the problems I have mentioned will dissipate as its usage and popularity continue to increase.

### Bear in mind...

Before you start switching everything from pandas to pyspark/koalas, bear in mind that running Spark locally is not the idea here. The whole point is that you have a cluster of machines running Spark with which your machine communicates and efficiently distributes the workload. There is actually a material decrease in the throughput of an individual machine in order to provide this horizontal scalability. The ability to run Spark locally is really just there for getting to grips with it and testing your code. Actually using it on just one machine is unlikely to offer you any speedups and for smaller datasets will be _significantly_ slower.

## Dask

Another solution to this problem that you may have heard of is called `Dask`. Dask was built to solve the exact same problem as Spark but specifically with Python in mind, leveraging the traditional python data science stack of Pandas, Scikit-Learn, Numpy, etc. Indeed dask is written in python, for python, with close collaboration with the aforementioned libraries and offers no APIs in other languages. Their intention is that dask can be used to scale python data science workflows to distributed clusters more natively with minimal rewriting of code.

So what are some of the other differences between Spark and Dask?

Well, `Dask` being the newcomer (created in 2014) anticipated this question and has a web page on all the differences, which you can access [here](https://docs.dask.org/en/latest/spark.html). To summarise, not a whole lot. It's almost like choosing between `vim` and `emacs`, people tend to fervently defend whichever one it is that they've chosen but it doesn't actually matter all that much. In fact, as the Dask documentation highlights, there is nothing wrong with choosing both and deploying them on the same cluster and the same data if you want to have the best of both worlds. However, some differences worth pointing out are that Dask does not perform lazy evaluation by default and therefore can suffer from slower performance with more complex operations if the lazy evaluation feature (`dask.delayed`) is not used. Furthermore, whilst it claims to replicate the pandas API, its coverage of the API doesn't actually match that of koalas. Finally, Dask doesn't offer any machine learning or SQL querying tools out of the box like Spark, instead leaving that to the likes of scikit-learn, etc. Naturally, it doesn't play as nice as Spark does with the rest of the Apache suite either if this is priority for you.

### Modin

So what is `modin`? Modin is a completely separate project which aims to be a big data drop-in replacement for pandas. This means they are aiming for 100% pandas API coverage - currently hovering [around 90%](https://github.com/modin-project/modin#pandas-api-coverage). They currently offer two backends for the heavy lifting: `dask` and `ray`. Their main selling point at the moment appears to be that with just a one line code change:

```diff
+ import modin.pandas as pd
- import pandas as pd
```

you can use all the cores of your machine instead of being stuck with just one. Whilst `dask` scales to multi-node clusters and supports 'out-of-core' data reading (i.e. using the disk if data is too large to fit into memory), currently modin offers this only as experimental features. Arguably, these two features are so core that you don't really want to be doing away with them. Nevertheless, it seems the team behind modin have very lofty goals for their project so whilst their use case isn't the most compelling right now, it's certainly one to watch.

## Vaex

Finally we have [`vaex`](https://github.com/vaexio/vaex). Vaex is a new project which aims to deal with big data without resorting to using clusters. It still supports out-of-core computation, lazy evaluation, a pandas-esque API and all the rest of it - but just on one machine. Their philosophy appears to be that cluster computing is overkill for the majority of problems it is used for. Unless absolutely necessary, we should try and use just one machine and avoid the overhead of managing a cluster, and vaex is just the tool to make the most of your machine.

I have to say, I'm a big fan of this philosophy but of course that alone isn't a compelling enough reason to use their solution. So why should you use vaex? Because it's super fast. One of the creators of vaex (admittedly biased) has written a [blog post](https://towardsdatascience.com/beyond-pandas-spark-dask-vaex-and-other-big-data-technologies-battling-head-to-head-a453a1f8cc13) benchmarking their library against the aforementioned incumbents. However what you can't argue with are hard numbers and frankly vaex smokes the rest of the pack. Only spark (both pyspark and koalas) comes close in matching it for speed. So, if speed is your primary objective and you don't care about cluster computing, use vaex.

## Summary

To summarise, you may benefit from this makeshift flowchart. Don't take it too seriously but if you're struggling to decide what is the right tool for the job, it may be of help:

<div class="mermaid">
graph LR
    A[Start] --> B{Single machine?};
    B -->|Yes| C{Speed vs Pandas API?};
    C -->|Speed| vaex[vaex];
    C -->|Pandas API| D{Big data*?};
    D -->|Yes| modin[modin];
    D -->|No| pandas[pandas];
    B -->|No| E{Lightweight + pythonic?}
    E -->|Yes| dask[dask];
    E -->|No| F{Pandas API?};
    F -->|Yes| koalas[koalas];
    F -->|No| pyspark[pyspark];
    style vaex fill:#f96,stroke:#000,stroke-width:4px
    style modin fill:#f96,stroke:#000,stroke-width:4px
    style pandas fill:#f96,stroke:#000,stroke-width:4px
    style koalas fill:#f96,stroke:#000,stroke-width:4px
    style pyspark fill:#f96,stroke:#000,stroke-width:4px
    style dask fill:#f96,stroke:#000,stroke-width:4px
</div>

_\* = What constitutes *big data* in this particular context will require some trial and error and depends on the operations you are performing but it is likely to be in the Gigabytes range_

### Table

Just to remind yourself of some of the main differences:

{:class="table table-bordered"}
| Library | pandas API | Out-of-core | Cluster | Backend |
|:-------:|:----------:|:-----------:|:-------:|:-------:|
| `pandas` | Yes | No | No | pandas |
| `pyspark` | No | Yes | Yes | Spark |
| `koalas` | Mostly | Yes | Yes | Spark |
| `dask` | Partially | Yes | Yes | Dask |
| `modin` | Mostly | Preliminary | Preliminary | Dask |
| `vaex` | Partially | Yes | No | vaex |

### Honourable Mentions

Finally, just wanted to say that there are more than just the six libraries mentioned in this blog post which may have ignored some libraries in the interests of brevity. The three below deserve an honourable mention at the very least and may be the right tool for your particular use case.

- [cuDF](https://github.com/rapidsai/cudf)
  - A GPU DataFrame library built on top of Apache Arrow. Definitely an interesting project. If you have GPUs that aren't being utilised for something else, might be worth trying this out
- [Optimus](https://github.com/ironmussa/Optimus)
  - Another spark library aiming to extend pyspark functionality but with a (mostly) pandas API. Not as popular as `koalas`
- [datatable](https://github.com/h2oai/datatable)
  - Still in beta stage but closely related to R's `data.table` and attempts to mimic its core algorithms and API
