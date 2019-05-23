In this blogpost I will describe my experience in using the Spark RDD's via Scala. I will also discuss the Spark sql impelementation

# making a shared folder within docker
Befor starting the assignment, it was needed to create a docker container. In doing so, it was recommended to share a folder within the docker with the PC itself. It took some time understanding the command, but in the end it was nice to learn a but more about docker and bas. I made a shared folder with:
```
docker run --name snbMounted -p 9000:9000 -p 4040-4045:4040-4045 -v $Sfiles/Big_data/assignment3/mntedDocker:/data -d rubigdata/hadoop
```
Here the 9000:9000 signifies which hosts to send and accept data from, as is also done for the porst with 4040-4045. It is then specified where the shared folder on the pc is located and where it is located in the docker (```/data```). Finally, the image is specified again.

# Scala
Scala has some things that took getting use to. For example the ```_``` symbol can have many meanings. In case of lamda-expressions, it can be used as a sort of shorthand notation in a lambda expression. For example, ```.filter(x -> x != n) ``` can be shortened to ```.filter(_ != n)``` Another example: ```val wc = words.reduceByKey(_+_)``` is the same as
```val wc = words.reduceByKey((a,b) => a + b)```.
Another use is to signify the nth element of a variable. For example ```x._2``` means the second element of x.


# spark
As mentiones in the lectures, in order to use spark an RDD (Resilient Distributed Dataset) must first be defined. In the example, an RDD was made with an array from 1 to 999, split into 8 partitions. Operarions could then in theory be made on all partitions at the same time.
Lateron RDDs are implicitly made on the 100.txt, plit into two partitions. This explains why all the results were split in two. Within the whole notebook, operations could be called but nothing would happen besides lazy evaluation. This is because spark would not act untill an actions (such as show or take) was called. This is once more in line with the spark dataframe.
Another interesting aspect of the programming itself was that each code was written in a stream-like fashion, ```A1.A2.A3... etc```. This is understandable when it is taken into consideration that the data on which the transformations is performed is partitioned and thus streamed parallel to eachother. In this particular case, the ```val lines = sc.textFile("/data/100.txt")``` made each line go through the stream of operations. 
Having some experience now in both spart and mapreduce, I find that spark is a lot more versitile and user friendly. I espeically liked how the mapping transfers more smootly into the reducing. 

# SQL
The dataframes took some getting use to, but once I got the hang of them their usefullness became quite evident.
During the assignment some data occured with coordinates having null values. Before solving this problem, as was done lateron, I was first interested in seeing if this data could be deleted altogeter. A first approach was to select the null values and drop them by ```cleanXDF = fullDF.filter($"X_COORD".isNull).na.drop()```. This however gave problems for it will return an empty dataframe. The new dataframe consiststs of only the null coordinates which are dropped. 
A second attempt yielded more success, ```val cleanXDF = fullDF.filter($"X_COORD".isNull).drop()``` filtered for null values and the drop function then dropped these rows from the dataframe.

Another problem, which I was not able to fix, was to nicelty loop through each column to check for NaN values. I attempted to get the names of columns with ```df.columns```, which worked. However, it was not simply possible to loop though them using ```println(addrDF.filter( $VAR.isNull ).count)``` wth a variable VAR, for the $ would not recognise the string. In the end it was done by hand, but in the future I would like to find a solution for this.

A final note is the relation between the dataframe and the RDD in general. It seems that the dataframe can be seen as a layer on top of the RDD (though this is not technically correct). A df.show() command will cause the RDD to execute each parition and collect its results, which can then be transformed/fed into a dataframe. This arrchitecture allows for the dataframe to be very compatible with the RDD while also allowing some good overview. In practise, it is likely that the dataframe (indirectly) tells the parition what to do in order to get nice compatibility with a dataframe format.

# end
All in all, I found the experience in Spark and the getting to know Scala helpfull. Although I find that it remains difficult, it was nice to get to know some basics.
