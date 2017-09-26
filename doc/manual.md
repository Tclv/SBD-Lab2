







-   [Introduction](#introduction)
-   [Goal of this Lab](#goal-of-this-lab)
-   [Amazon Web Services](#amazon-web-services)
-   [Common Crawl](#common-crawl)
-   [Apache Spark](#apache-spark)
-   [Building the pipeline](#building-the-pipeline)
    -   [Sense and Store](#sense-and-store)
    -   [Retrieve](#retrieve)
    -   [Filter](#filter)
    -   [Analysis](#analysis)
    -   [Visualization](#visualization)
-   [Chaining the Pipeline Together](#chaining-the-pipeline-together)
-   [AWS](#aws)
-   [Assignment](#assignment)

Introduction
============

In the previous assignments, you have become familiarized with a number of big data frameworks such as Hadoop and Apache Spark, but we haven’t done much supercomputing, let alone on big data. In the second lab of Supercomputing with Big Data we will analyze the [Common Crawl](http://commoncrawl.org), a monthly open-source crawl of the internet. As one might expect, this is a rather large data set: the [July 2017 crawl](http://commoncrawl.org/2017/07/july-2017-crawl-archive-now-available/) is 240 TiB uncompressed. To facilitate this analysis we will make use of Amazon Web Services (AWS). This lab is based on a Yelp Engineering blog post called, [Analyzing the Web For the Price of a Sandwich](https://engineeringblog.yelp.com/2015/03/analyzing-the-web-for-the-price-of-a-sandwich.html), and the [commoncrawl/cc-pyspark examples](https://github.com/commoncrawl/cc-pyspark). In this lab we will be building a lookup table for Dutch phone numbers and the sites they are referenced in.

This is the first year the lab is being held, and as such may experience some teething problems. Feedback is very much appreciated, and all the lab files will be hosted on [GitHub](github.com). Feel free to make issues and/or pull requests to suggest or implement improvements.

Goal of this Lab
================

The goal of this lab is to:

-   get hands-on experience with cloud based systems,
-   learn about the existing infrastructure for big data and the difficulties with these, and
-   learn how to characterize your computation and what machines best fit this profile.

You will work in groups of two. The first part of this lab will be used to develop an application that will be your baseline. After that you will get the freedom to optimize this application as you see fit. This could be in the form of a different analysis, that might force you to re-evaluate the characterization of the application, or in tuning specific parameters or parts of the original application.

This lab will be graded on the basis of your report. In the spirit of giving you freedom to find interesting things to optimize, you will not be graded on the achieved result, but on the quality of your analysis and the originality of your contribution. After the reports are handed in, each group will present their suggested improvements and the results they achieved. Finally each student will have a discussion with the TAs about their work.

For each optimization, you need to provide a hypothesis why you think this will improve some metric. Try and quantify this as well, giving you some expected result. Report how you implemented the suggested improvement, and finally measure the improvement in the system. Did this match your hypothesis? More interestingly, if it did not, why? Let’s illustrate this with an example:

> The analysis at hand takes about 4 hours to run on a cluster of 20 machines. Consider the amount of IO that happens at the start of the computation. The computation consists of analyzing 8TiB of data, thus each machine goes through about 400GB of data. The machines provisioned have a 400 Mbps connection. Each machine spends about 133 minutes downloading. For an extra $0.10 I can upgrade the machines one tier, doubling the connection speed. This moves the price from $0.20 to $0.30 per machine per instance hour. The machines spend half the time downloading, cutting of an hour of the computation, a 33% increase in speed. The provisioning cost of the machines changes from 20 ⋅ 0.20 ⋅ 4, to 20 ⋅ 0.30 ⋅ 3, or 16 to 18: a 10% decrease.
>
> This is tested by provisioning m4.xlarge machines instead of m4.large. This resulted in a computation that ran 50 minutes shorter. Due to the baseline being shorter than 4 hours it still resulted in an entire instance hour less than the baseline, so in practice we achieve a slightly smaller performance increase (26%), while still maintaining a 10% decrease in cost. This can be attributed to the CPUs of the machines not being able to keep up with the network connection. This can be seen from the CloudWatch monitoring tools, where it’s clear that our CPUs are over utilized, but the network connection has leftover bandwidth.

Finally, please test your hypothesis with classmates, as this will improve everybody’s understanding and that is what we are here for after all!

Amazon Web Services
===================

The complete data set we will be looking at is some 8 TiB big, so we need some kind of compute and storage infrastructure to run the pipeline. In this lab we will use Amazon AWS to facilitate this. As a student you are eligible for free credits on this platform. There are two options:

AWS normal account  
Register for a normal account (requires a credit card) and you are eligible for $40 in credits

AWS starter account  
Register for a educative account (does not require a credit card) and you are eligible for $30 in credits

If possible, we strongly recommend you get a normal account, as there are number of limitations associated with the starter accounts. AWS offers a large amount of different services, but for the first part of this lab only three will be relevant:

[EC2](https://aws.amazon.com/ec2/)  
Elastic Compute Cloud allows you to provision a variety of different machines. An overview of the different machines and their use cases can be found on the EC2 website.

[EMR](https://aws.amazon.com/emr/)  
Elastic MapReduce is a layer on top of EC2, that allows you to quickly deploy MapReduce and related applications (Apache Spark).

[S3](https://aws.amazon.com/s3/)  
Simple Storage Server is an object based storage system that is easy to interact with from different AWS services.

Note that the Common Crawl is hosted on AWS S3 in the [US east region](https://aws.amazon.com/public-datasets/common-crawl/), so any machines interacting with this data set should also be provisioned there.

AWS EC2 offers spot instances, a marketplace for unused machines that you can bid on. These spot instances are often a order of magnitude cheaper than on-demand instances. The current price list can be found in the [EC2 website](https://aws.amazon.com/ec2/spot/pricing/).

Common Crawl
============

The Common Crawl provides free access to monthly crawl data of the internet. The results are saved in WARC (Web ARChive file format) files. Three different kind of files are hosted by the [Common Crawl](http://commoncrawl.org/2014/04/navigating-the-warc-file-format/):

-   WARC files which store the raw crawl data,
-   WAT files which store computed metadata for the data stored in the WARC, and
-   WET files which store extracted plaintext from the data stored in the WARC.

The exact specification is [ISO 28500:2017](http://bibnum.bnf.fr/warc/WARC_ISO_28500_version1_latestdraft.pdf). Seeing how we are only interested in the plain text — as these contain the phone numbers — we will use the WET files.

Apache Spark
============

For this assignment we will use Python in cooperation with Apache Spark. The reason we’re using Python (rather than Scala or Java) is the availability of WARC parsing libraries in the Python ecosystem. Every student is free to implement this in his/her language of choice, but the examples will given in Python. Installing Apache Spark (and PySpark) should be straightforward on any Unix based system by using your System’s package manager (apt-get, yum, pacman, brew, etc.). We will be using Python 3, which might not be the default Python Apache Spark uses. This can be verified by running PySpark. Ensure that the Spark executables are in your path and run the following:

    ~ pyspark
    Python 2.7.13 (default, Jun  5 2017, 14:24:39)
    [GCC 4.2.1 Compatible Apple LLVM 8.1.0 (clang-802.0.42)] on darwin
    Type "help", "copyright", "credits" or "license" for more information.
    Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
    Setting default log level to "WARN".
    To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use ...
    17/08/21 14:48:34 WARN NativeCodeLoader: Unable to load native-hadoop ...
    17/08/21 14:48:39 WARN ObjectStore: Failed to get database global_temp ...
    Welcome to
          ____              __
         / __/__  ___ _____/ /__
        _\ \/ _ \/ _ `/ __/  '_/
       /__ / .__/\_,_/_/ /_/\_\   version 2.2.0
          /_/

    Using Python version 2.7.13 (default, Jun  5 2017 14:24:39)
    SparkSession available as 'spark'.

It is clear that Spark is using Python 2 instead of Python 3. This behaviour can be changed by setting an environmental variable.

``` bash
export PYSPARK_PYTHON=python3
```

You can also change in which interactive shell PySpark runs. For example, to use the `ipython` shell the following environmental variable needs to be set.

``` bash
export PYSPARK_DRIVER_PYTHON=ipython3
```

In both cases ensure that both `python3` and `ipython3` are in your `PATH`, or insert the complete path in the previous two exports.

For the pipeline, we will use a couple of libraries. Another bash script that downloads and installs these dependencies[1] will be of great use later on. Depending on your platform, you will need to modify the install command. Linux users might need to add `sudo`. Some platforms require a specific version of Python, e.g. `pip-3.4`.

``` bash
#!/usr/bin/env bash

INSTALL_COMMAND="pip3 install"
dependencies="warcio requests requests_file boto3 botocore py4j spark"

for dep in $dependencies; do
    $INSTALL_COMMAND $dep
done;
```

A complete version of this script can be found in the [lab’s GitHub repository](linktogithub).

Building the pipeline
=====================

In this chapter, we will talk about the different stages of the standard Big Data Pipeline and how they apply to the analysis we are trying to perform. In the next chapter we will demonstrate how to chain these different stages together.

Sense and Store
---------------

As this information is stored on Amazon S3, the sense and store stage are performed by the Common Crawl community.

Retrieve
--------

For the retrieval of data we will write a small bash script that will generate an index of the URLs of the Common Crawl. Additionally we will add functionality that let’s extract a few sample files to use for local development.

First off, let’s define which crawl we are looking at, where it is hosted, the number of example segments we want to use, and the type of file we are interested in.

``` bash
CRAWL=CC-MAIN-2017-13
BASE_URL=https://commoncrawl.s3.amazonaws.com
LOCAL_SEGMENTS=4
FILE_TYPE=wet
```

The Common Crawl is not hosted as a single 8 TiB file, but rather in small segments, each about 150 MiB in size. Each of these segments has a unique URL. We’d like to download a couple of these segments for local development. Furthermore we will generate two more index files showing where these example files are located locally and on the S3 servers. We will download the location of all the segments, select a couple of files for local development, and download these to their corresponding places.

``` bash
local_file_index=input/test_${FILE_TYPE}.txt
s3_sample_file_index=input/test_s3_${FILE_TYPE}.txt

test -d input || mkdir input
test -e $local_files || rm $local_file_index
test -e $s3_sample_file_index || rm $s3_sample_file_index

listing=crawl-data/$CRAWL/$FILE_TYPE.paths.gz
mkdir -p crawl-data/$CRAWL/
wget --timestamping $BASE_URL/$listing -O $listing
gzip -dc $listing | sed 's@^@s3://commoncrawl/@' \
    >input/all_${FILE_TYPE}_$CRAWL.txt

for segment in $(gzip -dc $listing | head -$LOCAL_SEGMENTS ); do
    mkdir -p $(dirname $segment)
    wget --timestamping $BASE_URL/$segment -O $segment
    echo file:$PWD/$segment >> $local_file_index
    echo s3://commoncrawl/$segment >> $s3_sample_file_index
done
```

A complete version of this script can be found in the [lab’s GitHub repository](tolink).

Filter
------

To identify phone numbers in plaintext, we need some way to recognize these numbers. The [Dutch phone number format](https://en.wikipedia.org/wiki/Telephone_numbers_in_the_Netherlands) can be written in two ways, the international and national form. To keep the filtering relatively simple, we will limit ourselves to the international form, as the national form is only identified with a single zero prefix, causing false positives as any 10 digit number starting with a 0 will now be identified as a phone number.

The international format starts with either a +31, or 0031, then an optional “(0)”, and finally 9 digits. We allow a variety of marks to be inserted between the numbers: spaces, parenthesis, and dashes. The following regular expression matches this specification.

``` regex
(?:(?<=\D)00[\(\)\- ]*3[\(\)\- ]*1|\+[\(\)\- ]*3[\(\)\- ]*1)
(?:[\(\)\- ]*\( *?0 *?\))?(?:[\(\)\- ]*[0-9]){9}
```

We would like to convert all matched phone numbers to a single format, to ensure we can match any duplicates. First we remove the optional parenthesised zero, and then we remove any additional punctuation. The following Regex can be used to remove these.

``` regex
(?:[\(\)\- ]*\( *?0 *?\))|[\(\)\- ]*
```

Finally we convert all the following numbers to the “+31” format. Effectively this means we need to replace all the “0031” prefixes to “+31”. The following regex can be used to substitute +31.

``` regex
^00
```

In Python we can compile these regex for faster repeated usage.

``` python
import re

class PhoneFilter:
    phone_nl_filter = re.compile(phone_regex)
    clean_filter = re.compile(replace_regex)
    zero_to_plus_filter = re.compile(zeroplus_regex)

    def find_phone_numbers(self, content):
        numbers = self.phone_nl_filter.findall(content)
        num_filt = {re.sub(self.zeroplus_filter, "+", 
                    re.sub(self.replace_filter, "", num)) for num in numbers}
        for n in num_filt:
            yield n
```

Analysis
--------

To allow for further analysis, we want to store the results in a suitable format. Apache Spark has introduced [DataFrames](https://spark.apache.org/docs/latest/sql-programming-guide.html) for exactly this purpose, from the Spark Website:

> A Dataset is a distributed collection of data. Dataset is a new interface added in Spark 1.6 that provides the benefits of RDDs (strong typing, ability to use powerful lambda functions) with the benefits of Spark SQL’s optimized execution engine.
>
> A DataFrame is a Dataset organized into named columns. It is conceptually equivalent to a table in a relational database or a data frame in R/Python, but with richer optimizations under the hood.

We would like to be able to analyze the matching between phone numbers and the websites they are referenced on. To use DataFrames we need to specify a schema. In our case the schema will be the following.

``` python
from pyspark.sql.types import StructType, StructField, StringType, ArrayType

output_schema = StructType([
    StructField("num", StringType(), True),
    StructField("urls", ArrayType(StringType()), True)
    ])
```

Visualization
-------------

We do not have anything planned here, but if someone knows a cool way to visualize this data, let us know!

Chaining the Pipeline Together
==============================

In this chapter we will demonstrate how we can use Apache Spark to chain this pipeline together. First off we need to load the relevant input document.

``` python
import spark
import spark.sql

class PhoneNumberExtractor:
    def __init__(self, input):
        self.input = input
    def main():
        sc = SparkContext()
        sqlc = SQLContext(sc)

        index = sc.textFile(self.input)
        phone_numbers = index.map(self.process_segments)

    def process_segments(self, segment_uri):
        stream = None
        if segment_uri.startswith('file:'):
            stream = self.process_file_warc(segment_uri)
        elif segment_uri.startswith('s3:/'):
            stream = self.process_s3_warc(segment_uri)
        if stream is None:
            return []
        return self.process_records(stream)

if __name__ == "__main__":
    input = "./input/test_wat.txt"
    output = "./output"
    pn = PhoneNumberExtractor(input)
    pn.main()
```

If we are using the URI refers to a local file, we can simply return the file (in binary mode).

``` python
def get_file(self, file_uri):
    return open(file_uri, 'rb') # Binary mode
```

If the URI refers to an S3 link, we need to download it first. We use [`boto3`](http://boto3.readthedocs.io/en/latest/), a library for interacting with S3, to download the segments. To download, a file handle has to be supplied. We can use `TemporaryFile` to generate temporary files (and handles). These files are automatically removed when the file handle is garbage collected.

``` python
def get_s3(self, s3_uri)
    no_sign_request = botocore.client.Config(signature_version=botocore.UNSIGNED)
    s3client = boto3.client('s3', config=no_sign_request)
    s3pattern = re.compile('^s3://([^/]+)/(.+)')
    s3match = s3pattern.match(uri)
    bucketname = s3match.group(1)
    path = s3match.group(2)
    warctemp = TemporaryFile(mode='w+b')
    s3client.download_fileobj(bucketname, path, warctemp)
    warctemp.seek(0)
    return warctemp
```

We are left with decoding the input stream into different records, filtering the phone numbers, and yielding (number, URL) pairs.

``` python
def process_records(self, stream):
    for rec in ArchiveIterator(stream):
        uri = rec.rec_headers.get_header("WARC-Target-URI")
        if uri is None:
            continue
        content = rec.content_stream().read().decode(utf-8)
        for num in self.find_phone_numbers(content):
            yield (num, uri)
```

Finally we write away these results to the output destination.

``` python
def main(self):
    sc = SparkContext(appName=self.name)
    sqlc = SQLContext(sparkContext=sc)
    index = sc.textFile(self.input, minPartitions=sc.defaultParallelism)
    phone_numbers = index.flatMap(self.process_warcs)
    phone_numbers_grouped = phone_numbers.groupByKey().mapValues(list)
    sqlc.createDataFrame(phone_numbers_grouped, schema=self.output_schema) \
            .write \
            .format("parquet") \
            .save(self.output)
```

A complete version of this program can be found on the lab’s GitHub repository. Play around with the script, and reading the data. How many phone numbers can you find in the two sample files. Try and estimate how many there will be in the complete data set based on this.

AWS
===

We will be using the AWS infrastructure to run the pipeline. I assume everyone has an AWS account at this point. Log in to the AWS console, and open the S3 interface. Create a bucket where we can store the scripts, the segments index files, and the output from the pipeline.

You can transport files via two ways:

1.  The web interface, and
2.  The Command Line Interface.

The web interface is fairly straightforward. To use the CLI, first install the AWS toolkit. This depends on your OS, but it’s available in the OS package managers, and otherwise can be installed via pip.

To copy a file

``` bash
aws s3 cp path/to/file s3://destination-bucket/path/to/file
```

To copy a directory recursively

``` bash
aws 3 cp --recursive s3://origin-bucket/path/to/file
```

To move a file

``` bash
aws 3 mv path/to/file s3://destination-bucket/path/to/file
```

The aws-cli ofcourse contains much more functionality, which can be found on the [AWS-CLI docs](https://aws.amazon.com/cli/).

Move the script, the dependency downloads, and the segment index to your S3 bucket.

We are now ready to provision a cluster. Go to the EMR service, and select *Create Cluster*. Next select *Go to advanced options*, select the latest release, and check the frameworks you want to use (in this case Spark, and Hadoop). We need to enter some software settings, specifically, to ensure we are using Python 3. Enter the following in the *Edit software settings* dialog. A copy paste friendly example can be found on the [AWS site](https://aws.amazon.com/premiumsupport/knowledge-center/emr-pyspark-python-3x/).

``` json
[
    {
    "Classification": "spark-env",
    "Configurations": [
            {
                "Classification": "export",
                "Properties": {
                    "PYSPARK_PYTHON": "/usr/bin/python3"
                }
            }
        ]
    }
]
```

EMR works with steps, which are equivalent to `spark-submits`. You can choose to add steps in the creation of the cluster, but this can also be done at a later point in time. Press *next*.

In the *Hardware Configuration* screen, we can configure the arrangement and selection of the machines. I would suggest starting out with *m4.large* machines on spot pricing. You should be fine running an example workload with a single master node and two core nodes. Be sure to select *spot pricing* and place an appropriate bid. Remember that you can always check the current prices in the information popup or on the [Amazon website](https://aws.amazon.com/ec2/spot/pricing/). After selecting the machines, press *next*.

In the *General Options* you can select a cluster name. You can tune where the system logs and a number of other features (more information in the popups). To install the dependencies on the system you need to add a *Bootstrap Action*. Select *Custom action*, then *Configure and add*. In this pop-up, give the script an appropriate name, the *Script location* should point to the source in your previously created S3 bucket. You can leave the *Optional arguments* dialog empty. After finishing this step, press *next*.

You should now arrive in the *Security Options* screen. If you have not created a EC2 Keypair, I highly recommend that you do so. This will allow you to forward the Yarn and Spark web interfaces to your browser. This makes debugging and monitoring the execution of your Spark Job much more manageable. Just follow the AWS instructions on doing so.

After this has all been completed you are ready to spin up your first cluster by pressing *Create cluster*. Once the cluster has been created, AWS will start provisioning machines. This should take about 10 minutes. In the meantime you can add a step. Go the *Steps* foldout, and select *Spark application* for *Step Type*. Clicking on *Configure* will open a dialogue in which you can select the application location in your S3 bucket, as well as provide a number of argument to the program, spark-submit, as well as your action on failure.

Once the cluster has been provisioned, the bootstrap script has ran (confirm this), and the debugging has taken place, the cluster is ready to execute your step. Whilst this is taking place you can setup a proxy to the cluster that allows you to view the web interfaces. More detailed information can be found on the [AWS website](http://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-web-interfaces.html). You can check the logs in your S3 bucket, or the web interfaces on the progress of your application and whether any errors have occured. If everything went well, you should have output in your S3 bucket. Check this on your machine by copying the files and inspecting the results in a Spark shell by loading it via SQLContext.

Assignment
==========

We have become familiar with both the pipeline in this exercise, as well as the AWS infrastructure. It is now up to you to find areas that you could improve the pipeline in.

[1] An experienced Python developer will wonder why we are not using the more idiomatic approach using a `requirements.txt` together with `pip`. This is due to EC2’s bootstrapping mechanism being somewhat more straightforward to use with a simple Bash script.