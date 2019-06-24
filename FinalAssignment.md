# Introduction
This is the blog for the final assignment of the Big Data Course. I worked on the commoncrawl data of the rubigdata course on github.

# Getting started
## create docker
The first step of all was to follow the tutorial links provided on the assignment. In these tutorials files and commands were given to start the spark cluster:
```
docker network create spark-net
docker create --name spark-master -h spark-master --network spark-net -p 8080:8080 -p 7077:7077 -e ENABLE_INIT_DAEMON=false bde2020/spark-master:2.4.1-hadoop2.7
docker create --name spark-worker-1 --network spark-net -p 8081:8081 --link spark-master:spark-master -e ENABLE_INIT_DAEMON=false bde2020/spark-worker:2.4.1-hadoop2.7
docker create --name spark-worker-2 --network spark-net -p 8082:8081 --link spark-master:spark-master -e ENABLE_INIT_DAEMON=false bde2020/spark-worker:2.4.1-hadoop2.7

wget http://rubigdata.github.io/course/background/rubigdata.tgz
tar xzvfp rubigdata.tgz
```
## get wanted file
To start the docker containers, I just ran ```docker start spark-master spark-worker-1 spark-worker-2``` 
Once everthing was running smoothly I decided to work on the webcrawl data. To do so, it had to be taken from the docker with the WARC tutorial. 
```
docker cp snbMountedFinalProj:/opt/docker/data/bigdata/course.warc.gz course.warc.gz
for node in master worker-1 worker-2
do
  docker exec spark-${node} mkdir -p /app
  docker cp course.warc.gz spark-${node}:/app
done
```
Having these files, I coudl get going.

## altering the scala file
The first thing I discovered is that altering the scala file in the extracted rubigdatafolder does not automatically change the spark applicationl. After each alteration it was required to rebuild the spark app by ```docker build --rm=true -t rubigdata/spark-app .```. It could then be executed (if no errors occured) by ```docker run --rm --name RUBigDataApp -e ENABLE_INIT_DAEMON=false --network spark-net rubigdata/spark-app```.

# Doing some Spark
## getting the output to a textfile
The first thing that I tried was getting the results saved into a text file rather than printing them. This however did not prove to be easy. I tried some things, but whenever I made a new text file I could not find it locally. The best attempt was:
```
import org.apache.spark.sql.SparkSession
import java.io.{BufferedWriter, FileWriter}
import java.io.File
import scala.io.Source

val file = new File("SCALARESULTS.txt")
	val bw = new BufferedWriter(new FileWriter(file))
	bw.write("Lines with a: %s, Lines with e: %s".format(numAs, numEs))
	bw.close()
```
Although this was still in vain. Googling a bunch it seemed [I was not the only one with this problem](https://stackoverflow.com/questions/54221228/how-to-read-a-file-in-spark-with-scala-using-new-file). Printing results would have to do for now as to not get too much sidetracked.

## Reading a WARC file
Since I wanted to work on the text withing the WARC files, it was needed to convert the HTML to nice text using the JavaSoup package. This had proven to be less easy than expected.
The first problem was that the ```sc.``` package was not known to the scala app. This was solved by defining a SparkConfig and SparkContext
```
import org.apache.spark.SparkContext
import org.apache.spark.SparkConf
var con = new SparkConf()
var sc = new SparkContext(con)
```
Once this was working, the JavaSoup pacakge from ```import org.jsoup.Jsoup ``` was not recognised as a package. This had to be solved by specifically adding ```"org.jsoup" 		  % "jsoup" 		 % "1.8.2"``` to the ```build.cbt``` depencencies.

Having done all that and having imported the needed packages, I could now replicate the resulst in the WARC notebook.

## Lettercounts
Now that all the supporting stuff was finished, I started with a small task to get familliar with Spark: counting how often the letter 'a' occured per link. To do so I filtered out the "301 Moved Permanently" entries and applies a ``` count``` funtions within a ```map```.

## total word count per link
Next I wanted to step it up a bit. I wanted to find the amount of words present in each link. This required to split each entry on whitespaces, filtering out empty list emelents and then taking the size of the list. In the end this was possible with a single map and basic filtering and array operations.

## percentage of occurence
Now that I could find how many words there are per entry, it would be easy to sum the total amount of words. With this information relative information could easily be derrived. Here I chose to find how much percentage of the words in the whole crawl are either 'spark' or 'rdds' (case-insensitive). Getting the total amount of words required collecting the RDD results and then summing them. Getting the amount of specific words proved to be a bit harder, for the collected result was a list of tuples. 
To 

# Future Ideas
other than html data (meta data)
top occuring words diffferences

# Code
