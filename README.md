# Apache Spark

This example shows how to deploy a stateless Apache Spark cluster with S3 support on Kubernetes. This is based on the "official" [kubernetes/spark](https://github.com/kubernetes/kubernetes/tree/master/examples/spark) example, which also contains a few more details on the deployment steps.

## Deploying Spark on Kubernetes

Create a Docker Container and push to public repo

```
docker build -t nitishtiwari/spark-hadoop:2.3.0 .
docker push nitishtiwari/spark-hadoop:2.3.0
```

Deploy the Spark master Replication Controller and Service:

```
$ kubectl create -f spark-master-controller.yaml
$ kubectl create -f spark-master-service.yaml
```

Next, start your Spark workers:

```
$ kubectl create -f spark-worker-controller.yaml
```

Let's wait until everything is up and running:

```
$ kubectl get all
NAME                               READY     STATUS    RESTARTS   AGE
po/spark-master-controller-5rgz2   1/1       Running   0          9m
po/spark-worker-controller-0pts6   1/1       Running   0          9m
po/spark-worker-controller-cq6ng   1/1       Running   0          9m

NAME                         DESIRED   CURRENT   READY     AGE
rc/spark-master-controller   1         1         1         9m
rc/spark-worker-controller   2         2         2         9m

NAME               CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
svc/spark-master   10.108.94.160   <none>        7077/TCP,8080/TCP   9m
```

## Running queries against S3

Now, let's fire up a Spark shell and try out some commands:

```
kubectl exec -it spark-master-controller-v2hjb /bin/bash
export SPARK_DIST_CLASSPATH=$(hadoop classpath)
spark-shell
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
Spark context Web UI available at http://192.168.132.147:4040
Spark context available as 'sc' (master = spark://spark-master:7077, app id = app-20170405152342-0000).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.1.0
      /_/

Using Scala version 2.11.8 (OpenJDK 64-Bit Server VM, Java 1.8.0_111)
Type in expressions to have them evaluated.
Type :help for more information.

scala>
```

Excellent, now let's tell our Spark cluster the details of our S3 target, this will use https by default:

```
sc.hadoopConfiguration.set("fs.s3a.endpoint", "http://minio-hot.default.svc.cluster.local:9000")
sc.hadoopConfiguration.set("fs.s3a.access.key", "minio")
sc.hadoopConfiguration.set("fs.s3a.secret.key", "minio123")
sc.hadoopConfiguration.set("fs.s3a.path.style.access", "true")
```

Spark job against Kaggle Dataset: https://www.kaggle.com/datasnaek/youtube-new#USvideos.csv

```
import org.apache.spark._
import org.apache.spark.rdd.RDD
import org.apache.spark.util.IntParam
import org.apache.spark.sql.SQLContext
import org.apache.spark.graphx._
import org.apache.spark.graphx.util.GraphGenerators
import org.apache.spark.mllib.regression.LabeledPoint
import org.apache.spark.mllib.linalg.Vectors
import org.apache.spark.mllib.tree.DecisionTree
import org.apache.spark.mllib.tree.model.DecisionTreeModel
import org.apache.spark.mllib.util.MLUtils

val conf = new SparkConf().setAppName("YouTube")
val sqlContext = new SQLContext(sc)

import sqlContext.implicits._
import sqlContext._

val youtubeDF = spark.read.format("csv").option("sep", ",").option("inferSchema", "true").option("header", "true").load("s3a://spark/data.csv")

youtubeDF.registerTempTable("popular")

val fltCountsql = sqlContext.sql("select s.title,s.views from popular s")
fltCountsql.show()
val fltCountsql = sqlContext.sql("select s.Title,s.MaterialType,s.UsageClass from popular s where s.CheckoutYear = 2009")
fltCountsql.show()
```

## Further notes

This setup is a just an inital introduction on getting S3 working with Apache Spark on Kubernetes. Getting insights out of your data is the next step, but also optimizing performance is an important topic. For example, using Spark's `parallelize` call to parallelize object reads can yield massive performance improvements over using a simple `sc.textFiles(s3a://spark/*)` as used in this example.
