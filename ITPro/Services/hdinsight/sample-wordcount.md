<properties linkid="manage-services-hdinsight-sample-wordcount" urlDisplayName="HDInsight Samples" pageTitle="Samples topic title TBD - Windows Azure" metaKeywords="hdinsight, hdinsight sample, mapreduce" metaDescription="Learn how to run a simple MapReduce sample on HDInsight." umbracoNaviHide="0" disqusComments="1" writer="bradsev" editor="cgronlun" manager="paulettm" />

# The HDInsight WordCount sample
 
This sample topic shows how to run a MapReduce program that counts word occurrences in a text file with Windows Azure HDInsight using Windows Azure PowerShell. The WordCount MapReduce program is written in Java and runs on an HDInsight cluster. The text file analyzed here is the Project Gutenberg eBook edition of The Notebooks of Leonardo Da Vinci. 

The Hadoop MapReduce program reads the text file and counts how often each word occurs. The output is a new text file that consists of lines, each of which contains a word and the count (a key/value tab-separated pair) of how often that word occurred in the document. This process is done in two stages. The mapper takes each line from the input text as an input and breaks it into words. It emits a key/value pair each time a work occurs of the word followed by a 1. The reducer then sums these individual counts for each word and emits a single key/value pair that contains the word followed by the sum of its occurrences.

 
**You will learn:**
		
* How to use Windows Azure PowerShell to run a MapReduce program on an HDInsight cluster.
* How to write MapReduce programs in Java.


**Prerequisites**:	

- You must have a Windows Azure Account. For options on signing up for an account see [Try Windows Azure out for free](http://www.windowsazure.com/en-us/pricing/free-trial/) page.

- You must have provisioned an HDInsight cluster. For instructions on the various ways in which such clusters can be created, see [Get Started with Windows Azure HDInsight][hdinsight-get-started] or [Provision HDInsight Clusters](/en-us/manage/services/hdinsight/provision-hdinsight-clusters/)

- You must have installed Windows Azure PowerShell, and have configured them for use with your account. For instructions on how to do this, see [Install and configure PowerShell for HDInsight][hdinsight-configure-powershell]

##In this article	
This topic shows you how to run the sample, presents the Java code for the MapReduce program, summarizes what you have learned, and outlines some next steps. It has the following sections.
	
1. [Run the sample using Windows Azure PowerShell](#run-sample)	
2. [The Java code for the WordCount MapReduce program](#java-code)
3. [Summary](#summary)	
4. [Next steps](#next-steps)	

<h2><a id="run-sample"></a>Run the sample using Windows Azure PowerShell</h2> 

**To submit the MapReduce job**

1.	Open **Windows Azure PowerShell**. For instructions of opening Windows Azure PowerShell console window, see [Install and configure Windows Azure PowerShell][powershell-install-configure].

3. Set the two variables in the following commands, and then run them:
		
		$subscriptionName = "<SubscriptionName>"   # Windows Azure subscription name
		$clusterName = "<ClusterName>"             # HDInsight cluster name
		
5. Run the following command to create a MapReduce job definition:

		# Define the MapReduce job
		$wordCountJobDefinition = New-AzureHDInsightMapReduceJobDefinition -JarFile "wasb:///example/jars/hadoop-examples.jar" -ClassName "wordcount" -Arguments "wasb:///example/data/gutenberg/davinci.txt", "wasb:///example/data/WordCountOutput" 

	The hadoop-examples.jar file comes with the HDInsight cluster. There are two arguments for the MapReduce job. The first one is the source file name, and the second is the output file path. The source file comes with the HDInsight cluster, and the output file path will be created at the run-time.

6. Run the following command to submit the MapReduce job:

		# Submit the job
		Select-AzureSubscription $subscriptionName
		$wordCountJob = Start-AzureHDInsightJob -Cluster $clusterName -JobDefinition $wordCountJobDefinition | Wait-AzureHDInsightJob -WaitTimeoutInSeconds 3600  

	In addition to the MapReduce job definition, you also provide the HDInsight cluster name where you want to run the MapReduce job.

8. Run the following command to check any errors with running the MapReduce job:	
	
		# Get the job output
		Get-AzureHDInsightJobOutput -Cluster $clusterName -JobId $wordCountJob.JobId -StandardError 
		
**To retrieve the results of the MapReduce job**

1. Open **Windows Azure PowerShell**.
2. Set the three variables in the following commands, and then run them:

		$subscriptionName = "<SubscriptionName>"       # Windows Azure subscription name
		
		$storageAccountName = "<StorageAccountName>"   # Windows Azure storage account name
		$containerName = "<ContainerName>"			   # Blob storage container name

	The Windows Azure Storage account is the one you created earlier in the tutorial. The storage account is used to host the Blob container that is used as the default HDInsight cluster file system.  The Blob storage container name usually share the same name as the HDInsight cluster unless you specify a different name when you provision the cluster.

3. Run the following commands to create a Windows Azure storage context object:
		
		# Select the current subscription
		Select-AzureSubscription $subscriptionName

		# Create the storage account context object
		$storageAccountKey = Get-AzureStorageKey $storageAccountName | %{ $_.Primary }
		$storageContext = New-AzureStorageContext –StorageAccountName $storageAccountName –StorageAccountKey $storageAccountKey  

	The *Select-AzureSubscription* is used to set the current subscription in case you have multiple subscriptions, and the default subscription is not the one to use. 

4. Run the following command to download the MapReduce job output from the Blob container to the workstation:

		# Download the job output to the workstation
		Get-AzureStorageBlobContent -Container $ContainerName -Blob example/data/WordCountOutput/part-r-00000 -Context $storageContext -Force

	The */example/data/WordCountOutput* folder is the output folder specified when you run the MapReduce job. *part-r-00000* is the default file name for MapReduce job output.  The file will be downloaded to the same folder structure on the local folder. For example, in the following screenshot, the current folder is the C root folder.  The file will be downloaded to the *C:\example\data\WordCountOutput* folder. 

5. Run the following command to print the MapReduce job output file:

		cat ./example/data/WordCountOutput/part-r-00000 | findstr "there"


	The MapReduce job produces a file named *part-r-00000* with the words and the counts.  The script uses the findstr command to list all of the words that contains *"there"*.

The output from the WordCount script should appear in the cmd window:

![HDI.Sample.WordCount.Output][image-hdi-sample-wordcount-output]

Note that the output files of a MapReduce job are immutable. So if you rerun this sample you will need to change the name of the output file.

<h2><a id="java-code"></a>The Java code for the WordCount MapReduce program</h2>



	package org.apache.hadoop.examples;
	import java.io.IOException;
	import java.util.StringTokenizer;
	import org.apache.hadoop.conf.Configuration;
	import org.apache.hadoop.fs.Path;
	import org.apache.hadoop.io.IntWritable;
	import org.apache.hadoop.io.Text;
	import org.apache.hadoop.mapreduce.Job;
	import org.apache.hadoop.mapreduce.Mapper;
	import org.apache.hadoop.mapreduce.Reducer;
	import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
	import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
	import org.apache.hadoop.util.GenericOptionsParser;

	public class WordCount {

  	public static class TokenizerMapper 
       extends Mapper<Object, Text, Text, IntWritable>{
    
    private final static IntWritable one = new IntWritable(1);
    private Text word = new Text();
      
    public void map(Object key, Text value, Context context
                    ) throws IOException, InterruptedException {
      StringTokenizer itr = new StringTokenizer(value.toString());
      while (itr.hasMoreTokens()) {
        word.set(itr.nextToken());
        context.write(word, one);
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
    String[] otherArgs = new GenericOptionsParser(conf, args).getRemainingArgs();
    if (otherArgs.length != 2) {
      System.err.println("Usage: wordcount <in> <out>");
      System.exit(2);
    	}
    Job job = new Job(conf, "word count");
    job.setJarByClass(WordCount.class);
    job.setMapperClass(TokenizerMapper.class);
    job.setCombinerClass(IntSumReducer.class);
    job.setReducerClass(IntSumReducer.class);
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    FileInputFormat.addInputPath(job, new Path(otherArgs[0]));
    FileOutputFormat.setOutputPath(job, new Path(otherArgs[1]));
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  	}
  	}



<h2><a id="summary"></a>Summary</h2>

In this tutorial, you have seen how to run a MapReduce program that counts word occurrences in a text file with HDInsight using Windows Azure PowerShell.

<h2><a id="next-steps"></a>Next steps</h2>

For tutorials runnng other samples and providing instructions on using Pig, Hive, and MapReduce jobs on Windows Azure HDInsight with Windows Azure PowerShell, see the following topics:

* [Get Started with Windows Azure HDInsight][hdinsight-get-started]
* [Sample: 10GB GraySort][10gb-graysort]
* [Sample: Pi Estimator][pi-estimator]
* [Sample: C# Steaming][cs-streaming]
* [Use Pig with HDInsight][pig]
* [Use Hive with HDInsight][hive]
* [Windows Azure HDInsight SDK documentation][hdinsight-sdk-documentation]

[hdinsight-sdk-documentation]: http://msdnstage.redmond.corp.microsoft.com/en-us/library/dn479185.aspx
[getting-started]: /en-us/manage/services/hdinsight/get-started-hdinsight/
[10gb-graysort]: /en-us/manage/services/hdinsight/howto-run-samples/sample-10gb-graysort/
[pi-estimator]: /en-us/manage/services/hdinsight/howto-run-samples/sample-pi-estimator/
[cs-streaming]: /en-us/manage/services/hdinsight/howto-run-samples/sample-csharp-streaming/
[scoop]: /en-us/manage/services/hdinsight/howto-run-samples/sample-sqoop-import-export/
[mapreduce]: /en-us/manage/services/hdinsight/using-mapreduce-with-hdinsight/
[hive]: /en-us/manage/services/hdinsight/using-hive-with-hdinsight/
[pig]: /en-us/manage/services/hdinsight/using-pig-with-hdinsight/
 
[hdinsight-configure-powershell]: /en-us/manage/services/hdinsight/install-and-configure-powershell-for-hdinsight/
[hdinsight-get-started]: /en-us/manage/services/hdinsight/get-started-hdinsight/

[powershell-install-configure]: /en-us/manage/install-and-configure-windows-powershell/

[image-hdi-sample-wordcount-output]: ../media/HDI.Sample.WordCount.Output.png

