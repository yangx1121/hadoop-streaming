# Lab: Hadoop Streaming

1. Start an Amazon Elastic MapReduce (EMR) Cluster using Quickstart with the following setup:
	*  Give the cluster a name that is meaningful to you
	*  Use Release `emr-5.11.1`
	*  Select the first option under Applications
	*  Select 1 master and 2 core nodes, using `m4.large` instance types
	*  Select your correct EC2 keypair or you will not be able to connect to the cluster
	*  Click **Create Cluster**

2. Once the cluster is up and running and in "waiting" state, ssh into the master node: `ssh hadoop@[[master-node-dns-name]]`

3. Install git on the master node: `sudo yum install -y git`

3. Clone this repository to the master node. Note: since this is a public repository you do can use the `http` GitHub URL: `https://github.com/bigdatateaching/hadoop-streaming.git`

4. Change directory into the lab: `cd hadoop-streaming` 

5. In this example, you will run a **simulated MapReduce job** on a text file. We say simulated because you will not be using Hadoop to do this but rather a combination of command line functions and pipes that resemble what happens when you run a Hadoop Streaming job on a cluster on a large file. Page 50 of the book shows an example of how to test your mapper and reducer. 

	There is a file in this repository called `shakespeare.txt` which contains all of William Shakespeare's works in a single file. There are also two Python files: a mapper called `basic-mapper.py` and a reducer called `basic-reducer.py`. Open the files and look at the code so you get familiar with what is going on.

	Using Linux pipes, you will pipe the file into the mapper which produces a (key, value) pair in the form of `word\t1`. The `\t` is a tab character. 
	
	The output from the mapper is then piped into the `sort` command, which takes all of the mapper output and sorts it by key.
	
	The output of the `sort` is piped into the reducer, which totals up the sum, by, key.
	
	- Try just the mapper: `cat shakespeare.txt | ./basic-mapper.py` and see the output
	- Now add the `sort` command: `cat shakespeare.txt | ./basic-mapper.py | sort`
	- Now add the reducer: `cat shakespeare.txt | ./basic-mapper.py | sort | ./basic-reducer.py`
	- If you want to save the output to a file, just add a redirect and filename to the end of the string of commands: `cat shakespeare.txt | ./basic-mapper.py | sort | ./basic-reducer.py > shakespeare-word-count.txt`

	This process simulates what happens at scale when running a Hadoop Streaming job on a a larger file. This was a linear process. When run at scale, the following is happening:
	
	- Every block of input data is piped through the mapper program, so you will have a mapper task per block
	- After all the map processes are done, the "Shuffle and Sort" part of the Hadoop framework is performing on large `sort` operation. 
	- Many keys are sent together to a single reduce process. There is a guarantee that all the records with the same key are sent to the same reducer. **A single reduce program will process multiple keys.**

6. Now let's run an actual Hadoop Streaming job, using this mapper and reducer but on a relatively larger dataset. We have compiled the entire [Project Gutenberg](https://www.gutenberg.org/) collection, a set of over 50,000 ebooks in plain text files, into a single file on S3 - `s3://bigdatateaching/gutenberg-single-file.txt`. This is single, large, text file with approximately 700 million (yes, million) lines of text, and about ~30GB in size. 

	For the purposes of this lab, you will work with a subset of the file, the first 25 million lines in a file that is about ~1.3GB, for the sake of speed. You will run a Hadoop Streaming job to do run the word count mapper and reducer to gain experience using Hadoop Streaming.
	
	Run the following command (you can cut/paste this into the command line):
	
	```
	hadoop jar /usr/lib/hadoop/hadoop-streaming.jar \
	-files /home/hadoop/hadoop-streaming/basic-mapper.py,/home/hadoop/hadoop-streaming/basic-reducer.py \
	-input s3://bigdatateaching/gutenberg-single-file-25m.txt \
	-output gutenberg-word-count \
	-mapper basic-mapper.py \
	-reducer basic-reducer.py
	```
	* The first line `hadoop jar /usr/lib/hadoop/hadoop-streaming.jar` is launching Hadoop with the Hadoop Streaming jar. A jar is a Java Archive file, and Hadoop Streaming is a special kind of jar that allows you to run non Java programs.
	* The line `-files /home/hadoop/hadoop-streaming/basic-mapper.py,/home/hadoop/hadoop-streaming/basic-reducer.py` tells Hadoop that it needs to "ship" the executable mapper and reducer scripts to every node on the cluster. If you are working on the master node, these files need to be on your **master (remote) filesystem.** Remember, these files do not exist before the job is run, so you need to package those files with the job so they run
	* The line `-input [[input-file]]` tells the job where your source file(s) are. These files need to be either in HDFS or S3. If you specify a directory, all files in the directory will be used as inputs
	* The line line `-output [[output-location]]` tells the job where to store the output of the job, either in HDFS or S3. **This parameter is just a name of a location, and it must not exist before running the job otherwise the job will fail.**
	* The line `-mapper basic-mapper.py` specifies the name of the executable for the mapper. Note that you need to ship the programs if they are custom programs
	* The line `-reducer basic-reducer.py` specifies the name of the executable for the mapper. Note that you need to ship the programs if they are custom programs
	* For more information about the Hadoop Streaming parameters, look at the documentation: [https://hadoop.apache.org/docs/r2.7.3/hadoop-streaming/HadoopStreaming.html](https://hadoop.apache.org/docs/r2.7.3/hadoop-streaming/HadoopStreaming.html)
	
1. We have provided multiple cuts of the Gutenberg data, from the whole file to the first 10, 25, and 50 million. We encourage you to experiment with these files, using clusters of different size.

