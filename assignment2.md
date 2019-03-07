Hello dear reader, this is my blog post for Assignemnt 2 - hello hadoop.
more info on the files can be found on: https://github.com/rubigdata/hello-hadoop-2019-Jvdjvdjvd

# Getting started
## Getting the Hadoop Docker
The first challenge of this assignment was to get the HDFS working. After following the steps of the tutorial, some errors occured. As I am new to using Linux, it turned out that these often had to do with permission problems. They were solved by adding ``` -sudo ``` in front of the commands or by adding myself to the docker group in Ubuntu.
Some problems alos occured when I tried to enter the docker a second time. It is wise to remember that ```start-all.sh``` is needed to actually start the dfs. 

## Getting Wordcount
More problems were encountered (and solved) when getting the Wordcount java file. At first, I tried extracting it from the example zip file. However, this resulted in it being extracted in a ```./org/apache/``` path. Furthermore, problems running the wordcount were also encontered due to problems referencing this path. In the end, I just copied the original code from the Wordcount into a new file. This solved it.

## Understanding the commands
A next problem was that the output file already existed. This was solved with 
``` bin.hdfs dfs -rm -r output ```. Knowing the ```-rm -r output ``` I finally understood the previous commands. It turns out that ```bin/hdfs -dfs``` calls commands within the hdfs system. ```bin/hadoop fs``` calls commands from hadoop. Although they are related, it is good to know that the hdfs commands concern itself with the file system (adding files, removing files etc.), while the hadoop commands are used to utilize this filesystem (running programs and such).

# Using Wordcount
## basics / first look
After a test of the wordcount functions, it was seen that this function consideres all files in the input directory to count the words of.
Closer inspection of the Worcount shows that it has a map and reduce function. Looking closer at the documentation, it can be seen that for each line each word (separated by a whitespace) is emitted with a count of 1. This means that words in the same line can be emitted multiple times in different pairs. This is reflected by the fact that 
```
while (itr.hasMoreTokens()) {
    word.set(itr.nextToken());
    context.write(word, one);
```
does not look back to the previously found words. The combiner then merges the counts for the same words (per line?) before it goes to the reducer for final reduction.
The reducer iteratres over each value, sums the counts and emits the key and the sum.
It seems to do this per key, rather than sorting on the keys itself. This would mean however that the reducer must be used within a loop. 

## Using mapreduce
### number of words
To use the mapreduce to count the number of words in Shakespear, you can run the following job:
```
bin/hadoop jar wc.jar WordCount input output
```
This results in a file where each words is listed, together with its count.
However, the words and characters are merged. This means that words character combinations are considered differently.
```
straws	1
straws,	4
straws.	1
```

It is assumed this is no problem.


### Romeo or Juliet
We now want to see whether Romeo or Juliet appears more often in each play. This can be done in a simple way (looking at the output of the original WordCount) or a harder way (editing the mapper).
We first try the simple way.
#### simple way
```
bin/hdfs dfs -get output/100S/part-r-00000 ./output/100S
cat ./output/100S | grep -i "romeo" 

'Romeo	2
ROMEO	2
ROMEO.	163
Romeo	45
Romeo!	11
Romeo!—	1
Romeo's	15
Romeo's,	1
Romeo,	35
Romeo,—	1
Romeo.	8
Romeo.]	11
Romeo.]—Peter!	1
Romeo:	2
Romeo;	2
Romeo;—come,	1
Romeo?	8
Romeo?—saw	1
it?—Romeo!	1
there?—Romeo,	1
title:—Romeo,	1

cat ./output/100S | grep -i "juliet"
'Juliet.']	1
JULIET	4
JULIET,	2
JULIET.	125
JULIET]	1
Juliet	17
Juliet!	1
Juliet's	8
Juliet,	19
Juliet.	6
Juliet.]	8
Juliet.—	1
Juliet:	1
Juliet;	2
Juliet?	5
Julietta	1
Julietta's	1
[Juliet	2
mistress!—Juliet!—fast,	1
```
A quick estiamtion shows that tome is used around 250 times, while juliete below 200 times. Romeo is thus used more often.

#### harder way
In this method, we will only count a word if it contains Romeo or juliet.
Since coding in mapreduce is quite a pain however, only pseud0-code will be given.

```
public void map(Object key, Text value, Context context
                ) throws IOException, InterruptedException {
  StringTokenizer itr = new StringTokenizer(value.toString());
  while (itr.hasMoreTokens()) {
    word.set(itr.nextToken());
    if ((word.toString()).contains("Romeo") || (word.toString()).contains("juliet")
    	context.write(word, one);
  }
}
```

If all goes well, the output now only has keys with this substrings
## extra challenges

Here some other applications are tried. Namely to count how often each character occurs (spaces not included) and how many lines there are.
Keep in minde that ```StringTokenizer itr = new StringTokenizer(value.toString());``` often remains necisarry for it splits the lines.

### number of characters
To count the number of characters, the code has to be modified in such a way that iteration takes place over each character of a word.

```
import java.io.IOException;
import java.util.StringTokenizer;
import java.text.CharacterIterator;
import java.text.StringCharacterIterator;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class LetterCount {

  public static class TokenizerMapper
       extends Mapper<Object, Text, Text, IntWritable>{

    private final static IntWritable one = new IntWritable(1);
    private Text word = new Text();
    private Text charactertxt = new Text();

    public void map(Object key, Text value, Context context
                    ) throws IOException, InterruptedException {
      StringTokenizer itr = new StringTokenizer(value.toString());
        while (itr.hasMoreTokens()) {
        	word.set(itr.nextToken());
        	CharacterIterator itr2 = new StringCharacterIterator(word.toString());
        	for(char c = itr2.first(); c!= itr2.DONE; c = itr2.next()){
        		charactertxt.set(Character.toString(c));
        		context.write(charactertxt, one);
      		}
    	}
  	}
  }

  public static class IntSumReducer
       extends Reducer<Text,IntWritable,Text,IntWritable> {
    private IntWritable result = new IntWritable();

    public void reduce(Text key, Iterable<IntWritable> values,
                       Context context
                       ) throws IOException, InterruptedException {
      int sum = 0;
      for (IntWritable val : values) {
        sum += val.get();
      }
      result.set(sum);
      context.write(key, result);
    }
  }

  public static void main(String[] args) throws Exception {
    Configuration conf = new Configuration();
    Job job = Job.getInstance(conf, "word count");
    job.setJarByClass(LetterCount.class);
    job.setMapperClass(TokenizerMapper.class);
    job.setCombinerClass(IntSumReducer.class);
    job.setReducerClass(IntSumReducer.class);
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  }
}

```

This code still had to be exported however.

```
export HADOOP_CLASSPATH=${JAVA_HOME}/lib/tools.jar
bin/hadoop com.sun.tools.javac.Main LetterCount.java
jar cf lt.jar LetterCount*.class
bin/hadoop jar lt.jar LetterCount input output/lettercount
```
Looking into this file, it was found that it worked.
```
root@d6b9a6ddbdc4:/opt/docker/hadoop-2.9.2# bin/hadoop fs -cat output/char/part-r-00000
!	8908
"	474
#	1
$	2
%	1
&	48
'	26484
(	193
)	191
*	38
,	91199
-	7428
.	81140
/	13
0	185
1	433
2	258
3	195
4	223
5	141
6	175
7	98
8	154
9	123
:	4223
;	19338
?	10979
@	1
A	45370
B	13466
C	18987
D	13475
E	37797
F	11444
G	10725
H	17244
I	52102
J	1859
K	6045
L	22521
M	14849
N	25958
O	29213
P	10728
Q	1237
R	25114
S	31276
T	39199
U	13677
V	3154
W	16480
X	393
Y	7497
Z	605
[	2811
\	1
]	2801
_	1855
`	1
a	263741
b	50577
c	72571
d	145094
e	442637
f	74636
g	62204
h	238451
i	216326
j	3042
k	31855
l	158054
m	102497
n	233793
o	303325
p	50703
q	2723
r	226357
s	234698
t	314714
u	123759
v	36947
w	79560
x	4940
y	91987
z	1225
|	32
}	4
Æ	1
à	1
æ	7
—	1246
‘	193
’	6215
“	150
”	68
```

Changing this code was quite hard. This is because little good documentation was present for the Text class in java.
In the end, it worked to import another class within the text class (StringCharacterIterator) to run over each character.


### number of lines
To count the number of lines, you can modify the mapper to output a dummy ("Lines: ") together with a count of 1 for each line. The trick is to do this once per line. 
```
import java.io.IOException;
import java.util.StringTokenizer;
import java.text.CharacterIterator;
import java.text.StringCharacterIterator;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class LineCount {

  public static class TokenizerMapper
       extends Mapper<Object, Text, Text, IntWritable>{

    private final static IntWritable one = new IntWritable(1);
    private Text word = new Text();

    public void map(Object key, Text value, Context context
                    ) throws IOException, InterruptedException {
      StringTokenizer itr = new StringTokenizer(value.toString());
      word.set("Lines:");
      context.write(word, one);
      		
    	}
  	}

  public static class IntSumReducer
       extends Reducer<Text,IntWritable,Text,IntWritable> {
    private IntWritable result = new IntWritable();

    public void reduce(Text key, Iterable<IntWritable> values,
                       Context context
                       ) throws IOException, InterruptedException {
      int sum = 0;
      for (IntWritable val : values) {
        sum += val.get();
      }
      result.set(sum);
      context.write(key, result);
    }
  }

  public static void main(String[] args) throws Exception {
    Configuration conf = new Configuration();
    Job job = Job.getInstance(conf, "word count");
    job.setJarByClass(LineCount.class);
    job.setMapperClass(TokenizerMapper.class);
    job.setCombinerClass(IntSumReducer.class);
    job.setReducerClass(IntSumReducer.class);
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  }
}

```
This was then run with
```
jar cf ln.jar LineCount*.class
bin/hadoop jar ln.jar LineCount input output/lin
```
Inspecting the output showed it was succesfull.
```
root@d6b9a6ddbdc4:/opt/docker/hadoop-2.9.2# bin/hadoop fs -cat output/lin/part-r-00000
Lines:	147838
```


### final note
The files were written in gedit inside my machine, and moved to the docker using for examle
``` docker cp LetterCount.java d6b9a6ddbdc4:/opt/docker/hadoop-2.9.2/LetterCount.java ```
Each output was save to a new output by specifying an `output/PATH` path when hadoop ran the program.

# Conclusions
All in all it was an intereting experience. I learned to use a docker container, use basic mapreduce in hadoop and how to make small edits to mapreduce code.
The biggest takeaways for me were understanding that:
1) the docker is similar to a virtual machine.
2) HDFS and Hadoop are related, but different. One uses HDFS, the other is HDFS.
3) Mapreduce follows a very strict paradigm of mapping and reducing. This means that editing what is count can be simple, for only the mapper has to be altered. However, it was difficult for me to get java code to work properly due to only limited amount of packages and the usage of the Text class rather than simple strings.



