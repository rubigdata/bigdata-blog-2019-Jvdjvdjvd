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

## Total word count per link
Next I wanted to step it up a bit. I wanted to find the amount of words present in each link. This required to split each entry on whitespaces, filtering out empty list emelents and then taking the size of the list. In the end this was possible with a single map and basic filtering and array operations.

## Percentage of occurence
Now that I could find how many words there are per entry, it would be easy to sum the total amount of words. With this information relative information could easily be derrived. Here I chose to find how much percentage of the words in the whole crawl are either 'spark' or 'rdds' (case-insensitive). Getting the total amount of words required collecting the RDD results and then summing them. Getting the amount of specific words proved to be a bit harder, for the collected result was a list of tuples. 
To count the occurence per tuple element, a ```foldLeft``` was needed with a specification on how to add the elements: ```val wordSum : (Double, Double)  = specificWords.collect().foldLeft((0.0,0.0)){ case ((a0,b0),(a1,b1)) => (a0+a1,b0+b1) }```, Google was a great pal.
Finally, with the specific counts and the total counts the frequency was detemined. There were 8528 words in total. 0.762195 percent were 'spark' and 0.140713 percent were 'rdds'.

## Depency between two words
On the website, it seems that the word RDD and Spark often occur together. To get an estimation on whether or not these words are related, I decided to do a [dependency test](https://www.statisticshowto.datasciencecentral.com/probability-and-statistics/dependent-events-independent/). This test was performed in the following steps:
1. find how often in total 'spark' and 'rdds' are present in a link.
2. find how often in 'spark' and 'rdds' are both present in a link.
3. calculate the probabiliy of each word occuring, using the total amount of links.
4. calcualte and compare the dependent and independent probabilities and see whether they are equal.

The final result was that the probability of 'spark' occuring is 0.750000.
The probability of 'rdds' occuring is 0.300000.
The probability of 'spark' given 'rdds' is 1.000000.
The probability of 'rdds' given 'spark' is 0.400000.

# Reflection
## New knowledge
From the project I got some hands on experience in scala and some idea in how spark works. The main concepts that stuck are that spark applications are build using a specified build file and that a specific way of programming is needed in order to facilitate the splitting of data for the RDDS.

# Future Ideas
A final idea that could have been interesting would have been to get the 10 most occuring non-standard words over all entries and then see which entries deviate in their 10 most occuring words. This would give some sort of 'normality' indication per link. This could be fun to do if time had permitted it or perhaps for another course, but for now this seemded sufficient.
In this project I only used the html data. It would also have been interesting to use other (meta) data present in the WARC file. It could also have been interesting to make a new webcrawl.


# Code
For a (personal) log of activiy, see this [link](https://github.com/rubigdata/cc-2019-Jvdjvdjvd/blob/master/logFinalProject.md).
The code is pasted below.

## Build.cbt
```
name		:= "RUBigDataApp"
version		:= "1.0"
scalaVersion	:= "2.11.12"

val sparkV	= "2.4.1"
val hadoopV	= "2.7.2"
val jwatV	= "1.0.0"

resolvers += "jitpack" at "https://jitpack.io"

libraryDependencies ++= Seq(
  "org.apache.spark" %% "spark-core" % sparkV % "provided",
  "org.apache.spark" %% "spark-sql"  % sparkV % "provided",
  "org.apache.hadoop" %  "hadoop-client" % hadoopV % "provided",
  "org.jwat"          % "jwat-common"    % jwatV,
  "org.jwat"          % "jwat-warc"      % jwatV,
  "org.jwat"          % "jwat-gzip"      % jwatV,
  "org.jsoup" 		  % "jsoup" 		 % "1.8.2"
)

libraryDependencies += "com.github.sara-nl" % "warcutils" % "-SNAPSHOT"	
```

## Running Spark in scala
### General body
Part of the code per experiment was the same. Here I pasted the parts that were always the same
```
package org.rubigdata
import nl.surfsara.warcutils.WarcInputFormat
import org.jwat.warc.{WarcConstants, WarcRecord}
import org.apache.hadoop.io.LongWritable;
import org.apache.commons.lang.StringUtils;
import org.apache.spark.sql.SparkSession
import scala.io.Source
import org.apache.spark.SparkContext
import org.apache.spark.SparkConf
import nl.surfsara.warcutils.WarcInputFormat

import java.io.IOException
import org.jsoup.Jsoup
import org.apache.spark.SparkConf
import java.io.InputStreamReader;


object RUBigDataApp {

 def getContent(record: WarcRecord):String = {
   val cLen = record.header.contentLength.toInt
   //val cStream = record.getPayload.getInputStreamComplete()
   val cStream = record.getPayload.getInputStream()
   val content = new java.io.ByteArrayOutputStream();
   val buf = new Array[Byte](cLen)
   var nRead = cStream.read(buf)
   while (nRead != -1) {
     content.write(buf, 0, nRead)
     nRead = cStream.read(buf)
   } 

   cStream.close()
  
   content.toString("UTF-8");
  }
    
  def HTML2Txt(content: String) = {
    try {
      Jsoup.parse(content).text().replaceAll("[\\r\\n]+", " ")
  	  }
  	catch {
      case e: Exception => throw new IOException("Caught exception processing input row ", e)
   	  }
	}
  def main(args: Array[String]) {
	var con = new SparkConf()
	var sc = new SparkContext(con)
    val fnm = "file:///app/course.warc.gz"
    val spark = SparkSession.builder.appName("RUBigDataApp").getOrCreate()
    //val data = spark.read.textFile(fnm).cache()
    //val numAs = data.filter(line => line.contains("a")).count()
    //val numEs = data.filter(line => line.contains("e")).count()
    
    
    val warcf = sc.newAPIHadoopFile(
              fnm,
              classOf[WarcInputFormat],               // InputFormat
              classOf[LongWritable],                  // Key
              classOf[WarcRecord]                     // Value
    )
    
    
  val warcc = warcf.
     filter{ _._2.header.warcTypeIdx == 2 /* response */ }.
     filter{ _._2.getHttpHeader().contentType.startsWith("text/html") }.
     map{wr => ( wr._2.header.warcTargetUriStr, HTML2Txt(getContent(wr._2)) )}.cache()
  
  println("***************")
  println("***************")
  println("RESULTS")
  
   // ADD SOME MORE DEPENDING ON WHAT YOU WANT
   //
   //
     
  println("***************")
  println("***************")
  spark.stop()
  }
}

```

###  Getting the amount of A's per link

```
val Acounts = warcc.
     filter(w => !(w._2.contains("301 Moved Permanently"))).
     map{wr => (wr._1 , wr._2.count(_ == 'a'))}
     .take(10) 

   Acounts.map(wr => println("%s has %d 'a' characters".format(wr._1,wr._2)))
    
  //println("Lines with a: %s, Lines with e: %s".format(numAs, numEs))

```

## Getting total word count per link
```
val wordCount = warcc.
     filter(w => !(w._2.contains("301 Moved Permanently"))).
     map{wr => (wr._1 , wr._2.
                split(" ").
                filter(_ != "").
                length
                
        )}
     .take(10)
     
  wordCount.map(wr => println("%s has %d words".format(wr._1,wr._2)))

```

## Getting percentage of occurence of a word
```
val word1 = "spark".toLowerCase
val word2 = "RDDs".toLowerCase

val wordLists = warcc.
     map (w => w._2).
     filter(w => !(w.contains("301 Moved Permanently"))).
     map(w => w.toLowerCase).
     map(w => w.split(" ")).cache()

val totalWords = wordLists.
                 map(
                   _.filter(_ != "").
                   length)
                .collect()
                .sum

val specificWords = wordLists.map(w => (w.count(_ == word1), w.count(_==word2)))
val wordSum : (Double, Double)  = specificWords.collect().foldLeft((0.0,0.0)){ case ((a0,b0),(a1,b1)) => (a0+a1,b0+b1) }
val percentage  = ((wordSum._1 / totalWords *100) , (wordSum._2 / totalWords *100))
println("There were %d words in total. %f percent were '%s' and %f percent were '%s'.".format(totalWords, percentage._1, word1, percentage._2, word2))
```

## Getting word1 given word2
```
al word1 = "spark".toLowerCase
val word2 = "RDDs".toLowerCase

val lowerCased = warcc.
     map (w => w._2).
     filter(w => !(w.contains("301 Moved Permanently"))).
     map(w => w.toLowerCase).
     cache()

val w1Amount : Double = lowerCased.filter(_.contains(word1)).collect().size
val w2Amount : Double  = lowerCased.filter(_.contains(word2)).collect().size
val bothAmount : Double = lowerCased.filter(w => w.contains(word1) && w.contains(word2)).collect().size
val totalCases : Double = lowerCased.collect().size
val probs = ((w1Amount / totalCases) , (w2Amount / totalCases) , (bothAmount / w2Amount) , (bothAmount / w1Amount))
println("The probability of '%s' occuring is %f.\nThe probability of '%s' occuring is %f.\nThe probability of '%s' given '%s' is %f.\nThe probability of '%s' given '%s' is %f.\n".format(word1, probs._1 , word2 , probs._2 , word1 ,  word2 , probs._3 , word2 , word1 , probs._4))
```
