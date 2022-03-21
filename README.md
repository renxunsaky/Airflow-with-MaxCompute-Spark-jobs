# Airflow-with-MaxCompute-Spark-jobs
Make the whole CICD GitOps pipeline work for Airflow + MaxCompute Spark Client

# Background
There is no native API provided by MaxCompute to submit Spark jobs remotely, so there are the following technical difficulties to schedule a MaxCompte job (especially Spark job) from Airflow:
* How to submit MaxCompute Spark job from Airflow
* How to read jars, job configuration from OSS, if we want to use OSS as GitOps storage endpoint
* How to show job history or view Spark executor metrics/logs easily

**Here comes a first proposal:**

![Airflow with MaxCompute Architecture](/ressources/Airflow_on_AliCloud_with_MaxCompute.png?raw=true "Airflow with MaxCompute Architecture")

# Manual implementation steps:
## Install Airflow on an ECS instance
## Install Apache Livy (here we take version 0.7.1), on an ECS instance. Download MaxCompute Spark client on the same VM as Apache Livy.
* Configure the environment variables, authentification settings etc. by following these two documentation:
  - [MaxCompute Spark client configuration](https://github.com/aliyun/MaxCompute-Spark/wiki/02.-使用Spark客户端提交任务(Yarn-Cluster模式)?spm=a2c63.p38356.0.0.2cbc5b780Fu7sd&file=02.-使用Spark客户端提交任务(Yarn-Cluster模式))
  - [Detailed Spark configuration for MaxCompute](https://github.com/aliyun/MaxCompute-Spark/wiki/03.-Spark配置详解?spm=a2c63.p38356.0.0.2cbca563LWmzCE&file=03.-Spark配置详解)
  - In order to make the MaxCompute Spark Client to work with OSS, for example, make it possible to submit job by using this command:
    ```
    curl -X POST --data '{"file": "oss://my-test/project/spark-examples_2.12-1.0.0-SNAPSHOT-shaded.jar", "className": "com.aliyun.odps.spark.examples.SparkPi"}' -H "Content-Type: application/json" http://localhost:8998/batches
    ```
    Besides, we need to do some modification on the MaxCompute Spark client to let it support OSS protocol:
    1. Ensure the ${SPARK_HOME}/conf/core-site.xml file supports AliyunOSSFileSystem.
       ```
       <property>
           <name>fs.oss.impl</name>
           <value>org.apache.hadoop.fs.aliyun.oss.AliyunOSSFileSystem</value>
       </property>
       ```
    2. Now that we tell Hadoop environment to use AliyunOSSFileSystem (which supports OSS protocol), we also need to add the implementation jars into the class path:  two jars are necessary inside the directory: ${SPARK_HOME}/jars. They are: aliyun-sdk-oss-2.2.1.jar and hadoop-fs-oss-3.3.8-public.jar.
    **Note: the version of the two jars are very important to avoid any error like "NoSuchMethodError" or "NoClassDefFoundError".**
* Configure Apache Livy to point to the configured MaxCompute Spark. This is used to let Apache Livy use MaxCompute client to submit Spark jobs.
  - Configure conf/livy.conf by setting:
  ```
  livy.server.port = 8998
  livy.spark.master = yarn
  livy.spark.deploy-mode = cluster
  ```
  - Configure conf/livy-env.sh:
  ```
  export JAVA_HOME=${JAVA_HOME:-/etc/alternative/jre}
  export SPARK_HOME=${SPARK_HOME:-/opt/spark-3.1.1-odps0.33.0}
  export HADOOP_CONF_DIR=${HADOOP_CONF_DIR:-/opt/spark-3.1.1-odps0.33.0/conf}
  export LIVY_LOG_DIR=${LIVY_LOG_DIR:-/var/log/livy}
  export LIVY_PID_DIR=${LIVY_PID_DIR:-/var/run/livy}
  ```
  - Optionally configure other settings, like authentification, HTTPs etc. according to your need:
  ```
  livy.server.auth.type = kerberos
  livy.server.auth.kerberos.principal = <spnego principal>
  livy.server.auth.kerberos.keytab = <spnego keytab>
  livy.server.auth.kerberos.name-rules = DEFAULT
  ```
* If necessary, configure the DNS, certification, ELB and make the instances in an auto-scaling group for both Livy and Airflow

# Industrialised implementation steps:
 **The corresponding Packer and Terraform codes will be provided later**
## Use Packer/Ansible to build an ECS Image for Airflow with the basic and common configuration
## Use Packer/Ansible to build an ECS Image for Livy and MaxCompute Spark Client, with the two additional jars added for OSS compatibility
## Use Terraform to deploy an auto-scaling group for Airflow / Livy with MaxCompute Spark with DNS/Certification and ELB configured (with HTTPs)
