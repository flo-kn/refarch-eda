---
title: Kafka Connect to S3 Source & Sink
description: Apache Kafka to AWS S3 object storage Source & Sink Connector usecase
---

## Overview

This scenario walkthrough will cover the usage of [IBM Event Streams](https://ibm.github.io/event-streams/about/overview/) as a Kafka provider and [Amazon S3](https://aws.amazon.com/s3/) as an object storage service as systems to integrate with the [Kafka Connect framework](https://ibm-cloud-architecture.github.io/refarch-eda/kafka/connect/). Through the use of the [Apache Camel opensource project](https://camel.apache.org/), we are able to use the [Apache Camel Kafka Connector](https://camel.apache.org/camel-kafka-connector/latest/index.html) in both a source and a sink capacity to provide bidirectional communication between [IBM Event Streams](https://ibm.github.io/event-streams/about/overview/) and [AWS S3](https://aws.amazon.com/s3/).


As different use cases will require different configuration details to accommodate different situational requirements, the Kafka to S3 Source and Sink capabilities described here can be used to move data between S3 buckets with a Kafka topic being the middle-man or move data between Kafka topics with an S3 Bucket being the middle-man. However, take care to ensure that you do not create an infinite processing loop by writing to the same Kafka topics and the same S3 buckets with both a Source and Sink connector deployed at the same time.

## Scenario prereqs

**OpenShift Container Platform**

- This deployment scenario was developed for use on the OpenShift Container Platform, with a minimum version of `4.2`.

**Strimzi**
- This deployment scenario will make use of the [Strimzi Operator](https://strimzi.io/docs/operators/latest/deploying.html) for Kafka deployments and the custom resources it manages.
- A minimum version of `0.17.0` is required for this scenario. This scenario has been explicitly validated with version `0.17.0`.
- The simplest scenario is to deploy the Strimzi Operator to watch all namespaces for relevant custom resource creation and management.
- This can be done in the OpenShift console via the **Operators > Operator Hub** page.

**Amazon Web Services account**
- As this scenario will make use of [AWS S3](https://aws.amazon.com/s3/), an active Amazon Web Services account is required.
- Using the configuration described in this walkthrough, an additional IAM user can be created for further separation of permission, roles, and responsibilities.
- This new IAM user should contain:
  -  The [`AmazonS3FullAccess` policy](https://console.aws.amazon.com/iam/home?region=us-east-1#/policies/arn:aws:iam::aws:policy/AmazonS3FullAccess$serviceLevelSummary) attached to it _(as it will need both read and write access to S3)_,
  -  An [S3 Bucket Policy](https://docs.aws.amazon.com/AmazonS3/latest/dev/walkthrough1.html) set on the Bucket to allow the IAM user to perform CRUD actions on the bucket and its objects.
- Create a file named `aws-credentials.properties` with the following format:

   ```
   aws_access_key_id=AKIA123456EXAMPLE
   aws_secret_access_key=strWrz/bb8%c3po/r2d2EXAMPLEKEY
   ```

- Create a Kubernetes Secret from this file to inject into the Kafka Connect cluster at runtime:

   ```shell
   kubectl create secret generic aws-credentials --from-file=aws-credentials.properties
   ```
- _Additional work is underway to enable configuration of the components to make use of IAM Roles instead._


**IBM Event Streams API Key**
- This scenario is written with [IBM Event Streams](https://ibm.github.io/event-streams/about/overview/) as the provider of the Kafka endpoints.
- API Keys are required for connectivity to the Kafka brokers and interaction with Kafka topics.
- An API key should be created with (at minimum) read and write access to the source and target Kafka topics the connectors will interact with.
- A Kubernetes Secret must be created with the Event Streams API to inject into the Kafka Connect cluster at runtime:

   ```shell
   kubectl create secret generic eventstreams-apikey --from-literal=password=<eventstreams_api_key>
   ```

**IBM Event Streams Certificates on IBM Cloud Pak for Integration**
- If you are using an IBM Event Streams instance deployed via the IBM Cloud Pak for Integration, you must also download the generated truststore file to provide TLS communication between the connectors and the Kafka brokers.
- This file, along with its password, can be found on the **Connect to this cluster** dialog in the Event Streams UI.
- Once downloaded, it must be configured to work with the Kafka Connect certificate deployment pattern:

   ```shell
   keytool -importkeystore -srckeystore es-cert.jks -destkeystore es-cert.p12 -deststoretype PKCS12
   openssl pkcs12 -in es-cert.p12 -nokeys -out es-cert.crt
   kubectl create secret generic eventstreams-truststore-cert --from-file=es-cert.crt
   ```

**IBM Event Streams Certificates on IBM Cloud**
- If you are using an IBM Event Streams instance deployed on IBM Cloud, you need to provide the root CA certificate to connect correctly from the Kafka Connect instance.
- This file can be downloaded as `eventstreams.cloud.ibm.com.cer` from the endpoint defined in your service credentials via the `kafka_http_url` property.
- Once downloaded, it must be configured to work with the Kafka Connect certificate deployment pattern:

   ```shell
   openssl x509 -inform DER -in eventstreams.cloud.ibm.com.cer -out es-cert.crt
   kubectl create secret generic eventstreams-truststore-cert --from-file=es-cert.crt
   ```

## Kafka Connect Cluster

We will take advantage of some of the developer experience improvements that OpenShift and the Strimi Operator brings to the Kafka Connect framework. The Strimzi Operator provides a `KafkaConnect` custom resource which will manage a Kafka Connect cluster for us with minimal system interaction. The only work we need to do is to update the container image that the Kafka Connect deployment will use with the necessary Camel Kafka Connector binaries, which OpenShift can help us with through the use of its Build capabilities.

#### (Optional) Create ConfigMap for log4j configuration

Due to the robust nature of Apache Camel, the default logging settings for the Apache Kafka Connect classes will send potentially sensitive information to the logs during Apache Camel context configuration. To avoid this, we can provide an updated logging configuration to the `log4j` configuration that is used by our deployments.

Save the properties file below and name it `log4j.properties`. Then create a ConfigMap via `kubectl create configmap custom-connect-log4j --from-file=log4j.properties`. This ConfigMap will then be used in our KafkaConnect cluster creation to filter out any logging output containing `accesskey` or `secretkey` permutations.

```properties
# Do not change this generated file. Logging can be configured in the corresponding kubernetes/openshift resource.
log4j.appender.CONSOLE=org.apache.log4j.ConsoleAppender
log4j.appender.CONSOLE.layout=org.apache.log4j.PatternLayout
log4j.appender.CONSOLE.layout.ConversionPattern=%d{ISO8601} %p %m (%c) [%t]%n
connect.root.logger.level=INFO
log4j.rootLogger=${connect.root.logger.level}, CONSOLE
log4j.logger.org.apache.zookeeper=ERROR
log4j.logger.org.I0Itec.zkclient=ERROR
log4j.logger.org.reflections=ERROR

# Due to back-leveled version of log4j that is included in Kafka Connect,
# we can use multiple StringMatchFilters to remove all the permutations
# of the AWS accessKey and secretKey values that may get dumped to stdout
# and thus into any connected logging system.
log4j.appender.CONSOLE.filter.a=org.apache.log4j.varia.StringMatchFilter
log4j.appender.CONSOLE.filter.a.StringToMatch=accesskey
log4j.appender.CONSOLE.filter.a.AcceptOnMatch=false
log4j.appender.CONSOLE.filter.b=org.apache.log4j.varia.StringMatchFilter
log4j.appender.CONSOLE.filter.b.StringToMatch=accessKey
log4j.appender.CONSOLE.filter.b.AcceptOnMatch=false
log4j.appender.CONSOLE.filter.c=org.apache.log4j.varia.StringMatchFilter
log4j.appender.CONSOLE.filter.c.StringToMatch=AccessKey
log4j.appender.CONSOLE.filter.c.AcceptOnMatch=false
log4j.appender.CONSOLE.filter.d=org.apache.log4j.varia.StringMatchFilter
log4j.appender.CONSOLE.filter.d.StringToMatch=ACCESSKEY
log4j.appender.CONSOLE.filter.d.AcceptOnMatch=false

log4j.appender.CONSOLE.filter.e=org.apache.log4j.varia.StringMatchFilter
log4j.appender.CONSOLE.filter.e.StringToMatch=secretkey
log4j.appender.CONSOLE.filter.e.AcceptOnMatch=false
log4j.appender.CONSOLE.filter.f=org.apache.log4j.varia.StringMatchFilter
log4j.appender.CONSOLE.filter.f.StringToMatch=secretKey
log4j.appender.CONSOLE.filter.f.AcceptOnMatch=false
log4j.appender.CONSOLE.filter.g=org.apache.log4j.varia.StringMatchFilter
log4j.appender.CONSOLE.filter.g.StringToMatch=SecretKey
log4j.appender.CONSOLE.filter.g.AcceptOnMatch=false
log4j.appender.CONSOLE.filter.h=org.apache.log4j.varia.StringMatchFilter
log4j.appender.CONSOLE.filter.h.StringToMatch=SECRETKEY
log4j.appender.CONSOLE.filter.h.AcceptOnMatch=false
```

#### Deploy the baseline Kafka Connect Cluster

Review the YAML description for our `KafkaConnectS2I` custom resource below, named `connect-cluster-101`. Pay close attention to _(using YAML notation)_:
- `spec.logging.name` should point to the name of the ConfigMap created in the previous step to configure custom `log4j` logging filters _(optional)_
- `spec.bootstrapServers` should be updated with your local Event Streams endpoints
- `spec.tls.trustedCertificates[0].secretName` should match the Kubernetes Secret containing the IBM Event Streams certificate
- `spec.authentication.passwordSecret.secretName` should match the Kubernetes Secret containing the IBM Event Streams API key
- `spec.externalConfiguration.volumes[0].secret.secretName` should match the Kubernetes Secret containing your AWS credentials
- `spec.config['group.id']` should be a unique name for this Kafka Connect cluster across all Kafka Connect instances that will be communicating with the same set of Kafka brokers.
- `spec.config['*.storage.topic']` should be updated to provide unique topics for this Kafka Connect cluster inside your Kafka deployment. Distinct Kafka Connect clusters should not share metadata topics.

```yaml
apiVersion: kafka.strimzi.io/v1alpha1
kind: KafkaConnectS2I
metadata:
  name: connect-cluster-101
  annotations:
    strimzi.io/use-connector-resources: "true"
spec:
  #logging:
  #  type: external
  #  name: custom-connect-log4j
  replicas: 1
  bootstrapServers: es-1-ibm-es-proxy-route-bootstrap-eventstreams.apps.cluster.local:443
  tls:
    trustedCertificates:
      - certificate: es-cert.crt
        secretName: eventstreams-truststore-cert
  authentication:
    passwordSecret:
      secretName: eventstreams-apikey
      password: password
    username: token
    type: plain
  externalConfiguration:
    volumes:
      - name: aws-credentials
        secret:
          secretName: aws-credentials
  config:
    group.id: connect-cluster-101
    config.providers: file
    config.providers.file.class: org.apache.kafka.common.config.provider.FileConfigProvider
    key.converter: org.apache.kafka.connect.json.JsonConverter
    value.converter: org.apache.kafka.connect.json.JsonConverter
    key.converter.schemas.enable: false
    value.converter.schemas.enable: false
    offset.storage.topic: connect-cluster-101-offsets
    config.storage.topic: connect-cluster-101-configs
    status.storage.topic: connect-cluster-101-status
```

Save the YAML above into a file named `kafka-connect.yaml`. If you created the ConfigMap in the previous step to filter out `accesskey` and `secretkey` values from the logs, uncomment the `spec.logging` lines to allow for the custom logging filters to be enabled during Kafka Connect cluster creation. Then this resource can be created via `kubectl apply -f kafka-connect.yaml`. You can then tail the output of the `connect-cluster-101` pods for updates on the connector status.

#### Build the Camel Kafka Connector

The next step is to build the Camel Kafka Connector binaries so that they can be loaded into the just-deployed Kafka Connect cluster's container images.

1. Clone the repository https://github.com/osowski/camel-kafka-connector to your local machine:

    - `git clone https://github.com/osowski/camel-kafka-connector.git`

2. Check out the `camel-kafka-connector-0.1.0-branch`:
   
   - `git checkout camel-kafka-connector-0.1.0-branch`

3. From the root directory of the repository, build the project components:
   
    - `mvn clean package`
4. Go get a coffee and take a walk... as this build will take around 30 minutes on a normal developer workstation.
   
    - To reduce the overall build scope of the project, you can comment out the undesired modules from the `<modules>` element of the `connectors/pom.xml` using `<!-- -->` notation.

5. Copy the generated S3 artifacts to the core package build artifacts:
   
    - `cp connectors/camel-aws-s3-kafka-connector/target/camel-aws-s3-kafka-connector-0.1.0.jar core/target/camel-kafka-connector-0.1.0-package/share/java/camel-kafka-connector/`

Some items to note:

- The repository used here (https://github.com/osowski/camel-kafka-connector) is a fork of the official repository (https://github.com/apache/camel-kafka-connector) with a minor update applied to allow for dynamic endpoints to be specified via configuration, which is critical for our [Kafka to S3 Sink Connector](#kafka-to-s3-sink-connector) scenario.
- This step (and the next step) will eventually be eliminated by providing an existing container image with the necessary Camel Kafka Connector binaries as part of a build system.

#### Deploy the Camel Kafka Connector binaries

Now that the Camel Kafka Connector binaries have been built, we need to include them on the classpath inside of the container image which our Kafka Connect clusters are using.

1. From the root directory of the repository, start a new OpenShift Build, using the generated build artifacts:

    ```shell
    oc start-build connect-cluster-101-connect --from-dir=./core/target/camel-kafka-connector-0.1.0-package/share/java --follow
    ```

2. Watch the Kubernetes pods as they are updated with the new build and rollout of the Kafka Connect Cluster using the updated container image _(which now includes the Camel Kafka Connector binaries)_:

    ```shell
    kubectl get pods -w
    ```

3. Once the `connect-cluster-101-connect-2-[random-suffix]` pod is in a `Running` state, you can proceed.

## Kafka to S3 Sink Connector

Now that you have a Kafka Connect cluster up and running, you will need to configure a connector to actually begin the transmission of data from one system to the other. This will be done by taking advantage of Strimzi and using the [`KafkaConnector` custom resource](https://strimzi.io/docs/0.17.0/#kafkaconnector_resources) the Strimzi Operator manages for us.

Review the YAML description for our `KafkaConnector` custom resource below, named `s3-sink-connector`. Pay close attention to:
- The `strimzi.io/cluster` label must match the deployed Kafka Connect cluster you previously deployed _(or else Strimzi will not connect the `KafkaConnector` to your `KafkaConnect` cluster)_
- The `topics` parameter _(named `my-source-topic` here)_
- The S3 Bucket parameter of the `camel.sink.url` configuration option _(named `my-s3-bucket` here)_

```yaml
apiVersion: kafka.strimzi.io/v1alpha1
kind: KafkaConnector
metadata:
  name: s3-sink-connector
  labels:
    strimzi.io/cluster: connect-cluster-101
spec:
  class: org.apache.camel.kafkaconnector.CamelSinkConnector
  tasksMax: 1
  config:
    key.converter: org.apache.kafka.connect.storage.StringConverter
    value.converter: org.apache.kafka.connect.storage.StringConverter
    topics: my-source-topic
    camel.sink.url: aws-s3://my-s3-bucket?keyName=${date:now:yyyyMMdd-HHmmssSSS}-${exchangeId}
    camel.sink.maxPollDuration: 10000
    camel.component.aws-s3.configuration.autocloseBody: false
    camel.component.aws-s3.accessKey: ${file:/opt/kafka/external-configuration/aws-credentials/aws-credentials.properties:aws_access_key_id}
    camel.component.aws-s3.secretKey: ${file:/opt/kafka/external-configuration/aws-credentials/aws-credentials.properties:aws_secret_access_key}
    camel.component.aws-s3.region: US_EAST_1
```

Once you have updated the YAML and saved it in a file named `kafka-sink-connector.yaml`, this resource can be created via `kubectl apply -f kafka-sink-connector.yaml`. You can then tail the output of the `connect-cluster-101` pods for updates on the connector status.

**NOTE:** If you require objects in S3 to reside in a sub-folder of the bucket root, you can place a folder name prefix in the `keyName` query parameter of the `camel.sink.url` configuration option above. For example, `camel.sink.url: aws-s3://my-s3-bucket?keyName=myfoldername/${date:now:yyyyMMdd-HHmmssSSS}-${exchangeId}`.

## S3 to Kafka Source Connector

Similar to the [Kafka to S3 Sink Connector](#kafka-to-s3-sink-connector) scenario, this scenario will make use of the Strimzi [`KafkaConnector` custom resource](https://strimzi.io/docs/0.17.0/#kafkaconnector_resources) to configure the specific connector instance.

Review the YAML description for our `KafkaConnector` custom resource below, named `s3-source-connector`. Pay close attention to:
- The `strimzi.io/cluster` label must match the deployed Kafka Connect cluster you previously deployed _(or else Strimzi will not connect the `KafkaConnector` to your `KafkaConnect` cluster)_
- The `topics` and `camel.source.kafka.topic` parameters _(named `my-target-topic` here)_
- The S3 Bucket parameter of the `camel.sink.url` configuration option _(named `my-s3-bucket` here)_

**Please note** that it is an explicit intention that the topics used in the [Kafka to S3 Sink Connector](#kafka-to-s3-sink-connector) configuration and the [S3 to Kafka Source Connector](#s3-to-kafka-source-connector) configuration are different. If these configurations were to use the **same Kafka topic** and the **same S3 Bucket**, we would create an infinite processing loop of the same information being endlessly recycled through the system. In our example deployments here, we are deploying to different topics but the same S3 Bucket.

```yaml
apiVersion: kafka.strimzi.io/v1alpha1
kind: KafkaConnector
metadata:
  name: s3-source-connector
  labels:
    strimzi.io/cluster: connect-cluster-101
spec:
  class: org.apache.camel.kafkaconnector.CamelSourceConnector
  tasksMax: 1
  config:
    key.converter: org.apache.kafka.connect.storage.StringConverter
    value.converter: org.apache.camel.kafkaconnector.awss3.converters.S3ObjectConverter
    topics: my-target-topic
    camel.source.kafka.topic: my-target-topic
    camel.source.url: aws-s3://my-s3-bucket?autocloseBody=false
    camel.source.maxPollDuration: 10000
    camel.component.aws-s3.accessKey: ${file:/opt/kafka/external-configuration/aws-credentials/aws-credentials.properties:aws_access_key_id}
    camel.component.aws-s3.secretKey: ${file:/opt/kafka/external-configuration/aws-credentials/aws-credentials.properties:aws_secret_access_key}
    camel.component.aws-s3.region: US_EAST_1
```

Once you have updated the YAML and saved it in a file named `kafka-source-connector.yaml`, this resource can be created via `kubectl apply -f kafka-source-connector.yaml`. You can then tail the output of the `connect-cluster-101` pods for updates on the connector status.

**NOTE:** If you require the connector to only read objects from a subdirecotry of the S3 bucket root, you can set the `camel.component.aws-s3.configuration.prefix` configuration option with the value of the subdirectory name. For example, `camel.component.aws-s3.configuration.prefix: myfoldername` .


## Event Streams v10 Addendum

- The steps up to now assumed usage of an OpenShift Container Platform v4.3.x cluster, IBM Cloud Pak for Integration v2020.1.1 and IBM Event Streams v2019.4.2
- With the release of Event Streams v10 (which is built on top of the Strimzi Kafka Operator) and the new CP4I there are some things need that need to be adjusted in the yamls to reflect that.

- You can edit the previous `kafka-connect.yaml` or create a new file.
**kafka-connect-esv10.yaml**
```yaml
apiVersion: eventstreams.ibm.com/v1beta1
kind: KafkaConnectS2I
metadata:
  name: connect-cluster-101
  annotations:
    eventstreams.ibm.com/use-connector-resources: "true"
spec:
  #logging:
  #  type: external
  #  name: custom-connect-log4j
  version: 2.5.0
  replicas: 1
  bootstrapServers: {internal-bootstrap-server-address}
  template:
    pod:
      imagePullSecrets: []
      metadata:
        annotations:
          eventstreams.production.type: CloudPakForIntegrationNonProduction
          productID: {product-id}
          productName: IBM Event Streams for Non Production
          productVersion: 10.0.0
          productMetric: VIRTUAL_PROCESSOR_CORE
          productChargedContainers: {connect-cluster-101}-connect
          cloudpakId: {cloudpak-id}
          cloudpakName: IBM Cloud Pak for Integration
          cloudpakVersion: 2020.2.1
          productCloudpakRatio: "2:1"
  tls:
      trustedCertificates:
        - secretName: {your-es-instance-name}-cluster-ca-cert
          certificate: ca.crt
  authentication:
    type: tls
    certificateAndKey:
      certificate: user.crt
      key: user.key
      secretName: {generated-secret-from-ui}
  externalConfiguration:
    volumes:
      - name: aws-credentials
        secret:
          secretName: aws-credentials
  config:
    group.id: connect-cluster-101
    config.providers: file
    config.providers.file.class: org.apache.kafka.common.config.provider.FileConfigProvider
    key.converter: org.apache.kafka.connect.json.JsonConverter
    value.converter: org.apache.kafka.connect.json.JsonConverter
    key.converter.schemas.enable: false
    value.converter.schemas.enable: false
    offset.storage.topic: connect-cluster-101-offsets
    config.storage.topic: connect-cluster-101-configs
    status.storage.topic: connect-cluster-101-status
```

- There are a couple changes of note here.
  - `apiVersion: eventstreams.ibm.com/v1beta1` instead of the previous Strimzi one.
  - `metadata.annotations.eventstreams.ibm.com/use-connector-resources: "true"` instead of the previous Strimzi one as well.
  - `spec.bootstrapservers:` Previously in Event Streams v2019.4.2 and prior there was only a single external facing route for this bootstrap server address. In Event Streams v10 there is an `External` and `Internal` one now. Replace `{internal-bootstrap-server-address}` with the `Internal` bootstrap server address from the Event Streams v10 UI.
  - `spec.template.pod.metadata.annotations.*` This entire section is new and represents new metadata that the Event Streams instance needs.
  - `spec.template.pod.metadata.annotations.productID` You can find this value when trying to deploy an Event Streams Custom Resource from the Installed Event Streams Operator YAML deployment or when applying this YAML through the command-line, OCP will provide you the proper values if they're wrong.
  - `spec.template.pod.metadata.annotations.cloudpakId` Same as productID.
  - `spec.tls.trustedCertificates` This is different from the previous one in that this certificate was automatically generated on creation of your Event Streams Kafka Cluster. If your Event Streams instance is named `eventstreams-dev` then this value should be `eventstreams-dev-cluster-ca-cert`.
  - `spec.authentication.*` This section is mostly different from the previous section. Prior we used a generated API Key, but on Event Streams v10 we will need to generate TLS credentials. In the `Internal` connection details there will be a `Generate TLS Credentials` button. Here you can name your secret and provide the proper access to it. **Note 1** - This is automatically created by the Event Streams Operator into the same namespace as both a `KafkaUser` Custom Resource as well as a secret. If your secret is named `internal-secret` then there will be an automatically generated Kubernetes/OpenShift secret named that as well as a `KafkaUser`. `user.crt` and `user.key` are certificates automatically generated within said secret. **Note 2** - Deletion of that secret will replicate itself. You will need to delete the associated and created `KafkaUser`.

**kafka-sink-connector.yaml-esv10.yaml**
```yaml
apiVersion: eventstreams.ibm.com/v1alpha1
kind: KafkaConnector
metadata:
  name: s3-sink-connector
  labels:
    eventstreams.ibm.com/cluster: connect-cluster-101
spec:
  class: org.apache.camel.kafkaconnector.CamelSinkConnector
  tasksMax: 1
  config:
    key.converter: org.apache.kafka.connect.storage.StringConverter
    value.converter: org.apache.kafka.connect.storage.StringConverter
    topics: my-topic
    camel.sink.url: aws-s3://my-s3-bucket?keyName=${date:now:yyyyMMdd-HHmmssSSS}-${exchangeId}
    camel.sink.maxPollDuration: 10000
    camel.component.aws-s3.configuration.autocloseBody: false
    camel.component.aws-s3.accessKey: ${file:/opt/kafka/external-configuration/aws-credentials/aws-credentials.properties:aws_access_key_id}
    camel.component.aws-s3.secretKey: ${file:/opt/kafka/external-configuration/aws-credentials/aws-credentials.properties:aws_secret_access_key}
    camel.component.aws-s3.region: my-s3-bucket-region
```
- For the most part identical besides two minor changes.
  - `apiVersion: eventstreams.ibm.com/v1alpha1` instead of Strimzi.
  - `metadata.labels.eventstreams.ibm.com/cluster: connect-cluster-101` again instead of Strimzi.

**kafka-source-connector.yaml-esv10.yaml**
```yaml
apiVersion: eventstreams.ibm.com/v1alpha1
kind: KafkaConnector
metadata:
  name: s3-source-connector
  labels:
    eventstreams.ibm.com/cluster: connect-cluster-101
spec:
  class: org.apache.camel.kafkaconnector.CamelSourceConnector
  tasksMax: 1
  config:
    key.converter: org.apache.kafka.connect.storage.StringConverter
    value.converter: org.apache.camel.kafkaconnector.awss3.converters.S3ObjectConverter
    topics: my-topic
    camel.source.kafka.topic: my-topic
    camel.source.url: aws-s3://my-s3-bucket?autocloseBody=false
    camel.source.maxPollDuration: 10000
    camel.component.aws-s3.accessKey: ${file:/opt/kafka/external-configuration/aws-credentials/aws-credentials.properties:aws_access_key_id}
    camel.component.aws-s3.secretKey: ${file:/opt/kafka/external-configuration/aws-credentials/aws-credentials.properties:aws_secret_access_key}
    camel.component.aws-s3.region: my-s3-bucket-region
```
- Similar to the S3 sink connector yaml.
  - `apiVersion: eventstreams.ibm.com/v1alpha1` instead of Strimzi.
  - `metadata.labels.eventstreams.ibm.com/cluster: connect-cluster-101` again instead of Strimzi.



## Troubleshooting

* The log output from the Kafka Connector instances will be available in the KafkaConnectS2I pods, named in the pattern of `{connect-cluster-name}-connect-{latest-build-id}-{random-suffix}`. This pattern will be referred to as `{connect-pod}` throughout the rest of the **Troubleshooting** section.


* To increase the logging output of the deployed connector instances:
  - Follow the steps for **"Create ConfigMap for log4j configuration"** in [Kafka Connect Cluster](#kafka-connect-cluster) above
  - Update the `log4j.properties` ConfigMap to include the line `log4j.logger.org.apache.camel.kafkaconnector=DEBUG`
  - Reapply the YAML configuration of the KafkaConnectS2I cluster


* To check the deployment status of applied Kafka Connect configuration via Kafka Connect's REST API, run the following commands:
  - `oc exec {connect-pod} bash -- -c "curl --silent http://localhost:8083/connectors/"`
  - `oc exec {connect-pod} bash -- -c "curl --silent http://localhost:8083/connectors/{deployed-connector-name}/status"`

## Next steps

- Enable use  of IAM Credentials in the Connector configuration _(as the default Java code currently outputs `aws_access_key_id` and `aws_secret_access_key` to container runtime logs due to their existence as configuration properties)_
- Optionally configure individual connector instances to startup with offset value of -1 _(to enable to run from beginning of available messages)_
- Implement a build system to produce container images with the Camel Kafka Connector binaries already present

## References
- [Apache Camel Kafka Connector - Try it out on OpenShift with Strimzi](https://camel.apache.org/camel-kafka-connector/latest/try-it-out-on-openshift-with-strimzi.html)
- [Apache Camel - Available pattern elements](https://camel.apache.org/components/latest/languages/simple-language.html) for use in the `keyName` parameter of the `camel.sink.url` property.
- [Apache Camel - Dynamic Endpoint reference](https://camel.apache.org/components/latest/eips/toD-eip.html)
- [Red Hat Developer Blog - Using secrets in Kafka Connect configuration](https://developers.redhat.com/blog/2020/02/14/using-secrets-in-apache-kafka-connect-configuration/)
- [Apache Kafka - Connect Overview](http://kafka.apache.org/documentation/#connect)
