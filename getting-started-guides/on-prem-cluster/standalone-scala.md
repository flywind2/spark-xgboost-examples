Get Started with XGBoost4J-Spark on an Apache Spark Standalone Cluster
======================================================================
This is a getting started guide to XGBoost4J-Spark on an Apache Spark3.0 Standalone Cluster. At the end of this guide, the reader will be able to run a sample Apache Spark application that runs on NVIDIA GPUs.

Prerequisites
-------------
* Apache Spark 3.0+ Standalone Cluster (e.g.: Spark 3.0)
* Hardware Requirements
  * NVIDIA Pascal™ GPU architecture or better
  * Multi-node clusters with homogenous GPU configuration
* Software Requirements
  * Ubuntu 16.04/CentOS7
  * CUDA V10.1/10.2  （CUDA 10.0 is no longer supported）
  * NVIDIA driver compatible with your CUDA
  * NCCL 2.4.7
  
* The number of GPUs in each host dictates the number of Spark executors that can run there. Additionally, cores per Spark executor and cores per Spark task must match, such that each executor can run 1 task at any given time. For example, if each host has 4 GPUs, there should be 4 or less executors running on each host, and each executor should run at most 1 task (e.g.: a total of 4 tasks running on 4 GPUs).
* In Spark Standalone mode, the default configuration is for an executor to take up all the cores assigned to each Spark Worker. In this example, we will limit the number of cores to 1, to match our dataset. Please see https://spark.apache.org/docs/latest/spark-standalone.html for more documentation regarding Standalone configuration.
* The `SPARK_HOME` environment variable is assumed to point to the cluster's Apache Spark installation.
* Follow the steps below to enable the GPU discovery for Spark on each host, since Spark3.0 now supports GPU scheduling, and this will let Spark3 find all available GPUs on standalone cluster.
  * Copy the spark config file from template
  ```
  cd ${SPARK_HOME}/conf/
  cp spark-defaults.conf.template spark-defaults.conf
  ```
  * Add the following configs to the file `spark-defaults.conf`. The number in first config should NOT larger than the actual number of the GPUs on current host. This example uses 1 as below for one GPU on the host.
  ```
  spark.worker.resource.gpu.amount 1
  spark.worker.resource.gpu.discoveryScript ${SPARK_HOME}/examples/src/main/scripts/getGpusResources.sh
  ```

Get Jars and Dataset
-------------------------------
1. Application Jar: Please build the sample_xgboost_apps jar with dependencies as specified in the [guide](/getting-started-guides/building-sample-apps/scala.md)
2. Rapids Plugin Jar: [*rapids-4-spark_2.12-0.3.0.jar*](https://repo1.maven.org/maven2/com/nvidia/rapids-4-spark_2.12/0.3.0/)
3. Dataset: https://rapidsai.github.io/demos/datasets/mortgage-data (The dataset needs to run with ETL first.)

Place the required jar and dataset in a local directory. In this example the jar is in the `xgboost4j_spark/jars` directory, and the `mortgage.zip` dataset was unzipped in the `xgboost4j_spark/data` directory. 

```
[xgboost4j_spark]$ find . -type f -print|sort
./data/mortgage/csv/test/mortgage_eval_merged.csv
./data/mortgage/csv/train/mortgage_train_merged.csv
./jars/rapids-4-spark_2.12-0.3.0.jar
./jars/sample_xgboost_apps-0.2.2-jar-with-dependencies.jar
``` 
Note: the `mortgage_eval_merged.csv` and `mortgage_train_merged.csv` are not Mortgage raw data, they are the data after Mortgage ETL job. If user wants to use a larger size Mortgage data, please refer to [Launch ETL job](#etl)

Launch a Standalone Spark Cluster
---------------------------------

0. Copy required jars to `$SPARK_HOME/jars` folder

```
cp rapids-4-spark_2.12-0.3.0.jar $SPARK_HOME/jars/
cp cudf-0.17-cuda10-2.jar $SPARK_HOME/jars/
```

1. Start the Spark Master process:

```
${SPARK_HOME}/sbin/start-master.sh
```

Note the hostname or ip address of the Master host, so that it can be given to each Worker process, in this example the Master and Worker will run on the same host.

2. Start a Spark slave process:

```
export SPARK_MASTER=spark://`hostname -f`:7077
export SPARK_CORES_PER_WORKER=1

${SPARK_HOME}/sbin/start-slave.sh ${SPARK_MASTER} -c ${SPARK_CORES_PER_WORKER} 
```

Note that in this example the Master and Worker processes are both running on the same host. This is not a requirement, as long as all hosts that are used to run the Spark app have access to the dataset.

<span id="etl">Launch Mortgage ETL</span>
---------------------------
If user wants to use a larger size dataset other than the default one, we provide an ETL app to process raw Mortgage data.

Variables required to run spark-submit command:
```
# path to xgboost4j_spark/libs
export LIBS_PATH=/home/xgboost4j_spark/lib

# Example jar built via mvn command
export JAR_EXAMPLE=${LIBS_PATH}/sample_xgboost_apps-0.2.2.jar

```

Run spark-submit
```
${SPARK_HOME}/bin/spark-submit \
    --master spark://$HOSTNAME:7077 \
    --executor-memory 32G \
    --conf spark.rapids.memory.gpu.pooling.enabled=false \
    --conf spark.executor.resource.gpu.amount=1 \
    --conf spark.task.resource.gpu.amount=1 \
    --conf spark.plugins=com.nvidia.spark.SQLPlugin \
    $JAR_EXAMPLE \
    --class com.nvidia.spark.examples.Mortgage.ETLMain  \
    -format=csv \
    -dataPath="perf::/home/xgboost4j_spark/data/mortgage/perf-train/" \
    -dataPath="acq::/home/xgboost4j_spark/data/mortgage/acq-train/" \
    -dataPath="out::/home/xgboost4j_spark/data/mortgage/out/train/"

# if generating eval data, change the data path to eval as well as the corresponding perf-eval and acq-eval data
# -dataPath="perf::/home/xgboost4j_spark/data/mortgage/perf-eval"
# -dataPath="acq::/home/xgboost4j_spark/data/mortgage/acq-eval"
# -dataPath="out::/home/xgboost4j_spark/data/mortgage/out/eval/"
```

Launch GPU Mortgage Example
---------------------------
Variables required to run spark-submit command:

```
# this is the same master host we defined while launching the cluster
export SPARK_MASTER=spark://`hostname -f`:7077

# location where data was downloaded 
export DATA_PATH=./xgboost4j_spark/data

# location for the required jar
export JARS_PATH=./xgboost4j_spark/jars

# Currently the number of tasks and executors must match the number of input files.
# For this example, we will set these such that we have 1 executor, with 1 core per executor

## take up the the whole worker
export SPARK_CORES_PER_EXECUTOR=${SPARK_CORES_PER_WORKER}

## run 1 executor
export SPARK_NUM_EXECUTORS=1

## cores/executor * num_executors, which in this case is also 1, limits
## the number of cores given to the application
export TOTAL_CORES=$((SPARK_CORES_PER_EXECUTOR * SPARK_NUM_EXECUTORS))

# spark driver memory
export SPARK_DRIVER_MEMORY=4g

# spark executor memory
export SPARK_EXECUTOR_MEMORY=8g

# example class to use
export EXAMPLE_CLASS=com.nvidia.spark.examples.mortgage.GPUMain

# XGBoost4J example jar (holds example classes):
export JAR_EXAMPLE=${JARS_PATH}/sample_xgboost_apps-0.2.2-jar-with-dependencies.jar

# Rapids plugin jar, working as the sql plugin on Spark3.0
export JAR_RAPIDS=${JARS_PATH}/rapids-4-spark_2.12-0.3.0.jar

# tree construction algorithm
export TREE_METHOD=gpu_hist
```

Run spark-submit:

```
${SPARK_HOME}/bin/spark-submit                                                  \
 --conf spark.plugins=com.nvidia.spark.SQLPlugin                       \
 --conf spark.rapids.memory.gpu.pooling.enabled=false                     \
 --conf spark.executor.resource.gpu.amount=1                           \
 --conf spark.task.resource.gpu.amount=1                              \
 --jars ${JAR_RAPIDS}                                           \
 --master ${SPARK_MASTER}                                                       \
 --driver-memory ${SPARK_DRIVER_MEMORY}                                         \
 --executor-memory ${SPARK_EXECUTOR_MEMORY}                                     \
 --conf spark.cores.max=${TOTAL_CORES}                                          \
 --class ${EXAMPLE_CLASS}                                                       \
 ${JAR_EXAMPLE}                                                                 \
 -dataPath=train::${DATA_PATH}/mortgage/csv/train/mortgage_train_merged.csv       \
 -dataPath=trans::${DATA_PATH}/mortgage/csv/test/mortgage_eval_merged.csv          \
 -format=csv                                                                    \
 -numWorkers=${SPARK_NUM_EXECUTORS}                                             \
 -treeMethod=${TREE_METHOD}                                                     \
 -numRound=100                                                                  \
 -maxDepth=8                                                                    
```

In `stdout` log on driver side, you should see timings<sup>*</sup> (in seconds), and the accuracy metric:

```
--------------
==> Benchmark: Elapsed time for [Mortgage GPU train csv stub Unknown Unknown Unknown]: 26.572s
--------------

--------------
==> Benchmark: Elapsed time for [Mortgage GPU transform csv stub Unknown Unknown Unknown]: 10.323s
--------------

--------------
==> Benchmark: Accuracy for [Mortgage GPU Accuracy csv stub Unknown Unknown Unknown]: 0.9869227318579323
--------------
```

Launch CPU Mortgage Example
---------------------------
If you are running this example after running the GPU example above, please set these variables, to set both training and testing to run on the CPU exclusively:

```
# example class to use
export EXAMPLE_CLASS=com.nvidia.spark.examples.mortgage.CPUMain

# tree construction algorithm
export TREE_METHOD=hist
```

This is the full variable listing, if you are running the CPU example from scratch:

```
# this is the same master host we defined while launching the cluster
export SPARK_MASTER=spark://`hostname -f`:7077

# location where data was downloaded 
export DATA_PATH=./xgboost4j_spark/data

# location where the required jar was downloaded
export JARS_PATH=./xgboost4j_spark/jars

# Currently the number of tasks and executors must match the number of input files.
# For this example, we will set these such that we have 1 executor, with 1 core per executor

## take up the the whole worker
export SPARK_CORES_PER_EXECUTOR=${SPARK_CORES_PER_WORKER}

## run 1 executor
export SPARK_NUM_EXECUTORS=1

## cores/executor * num_executors, which in this case is also 1, limits
## the number of cores given to the application
export TOTAL_CORES=$((SPARK_CORES_PER_EXECUTOR * SPARK_NUM_EXECUTORS))

# spark driver memory
export SPARK_DRIVER_MEMORY=4g

# spark executor memory
export SPARK_EXECUTOR_MEMORY=8g

# example class to use
export EXAMPLE_CLASS=com.nvidia.spark.examples.mortgage.CPUMain

# XGBoost4J example jar (holds example classes):
export JAR_EXAMPLE=${JARS_PATH}/sample_xgboost_apps-0.2.2-jar-with-dependencies.jar

# tree construction algorithm
export TREE_METHOD=hist
```

This is the same command as for the GPU example, repeated for convenience:

```
${SPARK_HOME}/bin/spark-submit                                                  \
 --master ${SPARK_MASTER}                                                       \
 --driver-memory ${SPARK_DRIVER_MEMORY}                                         \
 --executor-memory ${SPARK_EXECUTOR_MEMORY}                                     \
 --conf spark.cores.max=${TOTAL_CORES}                                          \
 --class ${EXAMPLE_CLASS}                                                       \
 ${JAR_EXAMPLE}                                                                 \
 -dataPath=train::${DATA_PATH}/mortgage/csv/train/mortgage_train_merged.csv       \
 -dataPath=trans::${DATA_PATH}/mortgage/csv/test/mortgage_eval_merged.csv          \
 -format=csv                                                                    \
 -numWorkers=${SPARK_NUM_EXECUTORS}                                             \
 -treeMethod=${TREE_METHOD}                                                     \
 -numRound=100                                                                  \
 -maxDepth=8                                                                    
```

In the `stdout` log on driver side, you should see timings<sup>*</sup> (in seconds), and the accuracy metric:

```
--------------
==> Benchmark: Elapsed time for [Mortgage CPU train csv stub Unknown Unknown Unknown]: 305.535s
--------------

--------------
==> Benchmark: Elapsed time for [Mortgage CPU transform csv stub Unknown Unknown Unknown]: 52.867s
--------------

--------------
==> Benchmark: Accuracy for [Mortgage CPU Accuracy csv stub Unknown Unknown Unknown]: 0.9872234894511343
--------------
```

<sup>*</sup> The timings in this Getting Started guide are only illustrative. Please see our [release announcement](https://medium.com/rapids-ai/nvidia-gpus-and-apache-spark-one-step-closer-2d99e37ac8fd) for official benchmarks.

