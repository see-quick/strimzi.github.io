---
layout: post
title:  "Introduction to system tests"
date: 2020-09-07
author: maros_orsak
---

Testing is an essential part of the software cycle.
Failures occur when testing is neglected.
In Strimzi, we are well aware of the need to test features.
Follow me on this blog post to see a basic view of our tests.

In this post we'll look at system tests and get **you** motivated to do the same.

## Motivation

Maybe you've been asking yourself the following questions:

- Why do we need to write system tests?
- Is there any value to system tests?
- Do they catch errors that unit or integration tests don't?

The answer is very simple.
Yes!
System tests are a key part of being able to ship Strimzi to users with more confidence that it will not crash in production.
The types of tests that validate the basic properties of a system we call [_Smoke tests_](https://www.guru99.com/smoke-testing.html)
Smoke tests detect major issues, such as issues with the deployment of an application or general configuration.
Moreover, we have regression test suites. 
Regression tests cover most features and will definitely find a bug if one is caused by recent changes in verified code.
In the next section, I will describe the difference between our system tests, integration, and unit and integration tests.

## How system tests differ from our unit and integration tests

### Unit tests

First, we need to know what is the purpose of the unit tests. 
Let's define a unit test.

> A unit test exercises a single behavior of a software module.
> That module is usually a class, and the behavior is usually a public method of the class.
> The test asserts that the actual result matches the expected result.
> These assertions must all pass, or the unit test will fail. (Ryan Cook)

For clarification I can show you an example of a unit test for thee `cluster-operator` module.
This is the test case, which verifies that if we set up `Kafka` with an external `nodeport` listener then `Kafka` in-memory representation
of class contains the external port name, external port 9094 with protocol TCP and more.
Everything what this type of test validates is only single behaviour of a sofware module that we defined
previously.
We don't need a running `Kubernetes cluster` for unit tests.
It is only in-memory representation of class, in this case the `Kafka` model.

```java
@Test
public void testExternalNodePorts() {
    Kafka kafkaAssembly = new KafkaBuilder(ResourceUtils.createKafka(namespace, cluster, replicas,
            image, healthDelay, healthTimeout, metricsCm, configuration, emptyMap()))
            .editSpec()
                .editKafka()
                    .withNewListeners()
                        .withNewKafkaListenerExternalNodePort()
                            .withNewKafkaListenerAuthenticationTlsAuth()
                            .endKafkaListenerAuthenticationTlsAuth()
                        .endKafkaListenerExternalNodePort()
                    .endListeners()
                .endKafka()
            .endSpec()
            .build();
    KafkaCluster kc = KafkaCluster.fromCrd(kafkaAssembly, VERSIONS);

    // Check StatefulSet changes
    StatefulSet sts = kc.generateStatefulSet(true, null, null);

    List<ContainerPort> ports = sts.getSpec().getTemplate().getSpec().getContainers().get(0).getPorts();
    assertThat(ports.contains(kc.createContainerPort(KafkaCluster.EXTERNAL_PORT_NAME, KafkaCluster.EXTERNAL_PORT, "TCP")), is(true));

    // Check external bootstrap service
    Service ext = kc.generateExternalBootstrapService();
    assertThat(ext.getMetadata().getName(), is(KafkaCluster.externalBootstrapServiceName(cluster)));
    assertThat(ext.getSpec().getType(), is("NodePort"));
    assertThat(ext.getSpec().getSelector(), is(kc.getSelectorLabels().toMap()));
    assertThat(ext.getSpec().getPorts(), is(Collections.singletonList(kc.createServicePort(KafkaCluster.EXTERNAL_PORT_NAME, KafkaCluster.EXTERNAL_PORT, KafkaCluster.EXTERNAL_PORT, "TCP"))));
    checkOwnerReference(kc.createOwnerReference(), ext);

    // Check per pod services
    for (int i = 0; i < replicas; i++)  {
        Service srv = kc.generateExternalService(i);
        assertThat(srv.getMetadata().getName(), is(KafkaCluster.externalServiceName(cluster, i)));
        assertThat(srv.getSpec().getType(), is("NodePort"));
        assertThat(srv.getSpec().getSelector().get(Labels.KUBERNETES_STATEFULSET_POD_LABEL), is(KafkaCluster.kafkaPodName(cluster, i)));
        assertThat(srv.getSpec().getPorts(), is(Collections.singletonList(kc.createServicePort(KafkaCluster.EXTERNAL_PORT_NAME, KafkaCluster.EXTERNAL_PORT, KafkaCluster.EXTERNAL_PORT, "TCP"))));
        checkOwnerReference(kc.createOwnerReference(), srv);
    }
}
```

### Integration tests

Next up are the integration tests, which move one level higher.
You can find all these tests inside our `api` module.
Before we get into the details again, we should understand the informal definition of integration tests.

> Integration tests determine if independently developed units of software work correctly when they are connected to each other. (Martin Fowler)

In our case, the purpose of integration tests is to ensure that we can create a resource from the POJOs, serialize it and
create the resource in Kubernetes.
You should notice that in this level of testing we are using the Kubernetes cluster for validation of custom resources.
To make this all easy to understand let me clarify it with example:

```java
@Test
public void testKafkaWithEntityOperator() {
    createDelete(Kafka.class, "Kafka-with-entity-operator.yaml");
}

protected <T extends CustomResource> void createDelete(Class<T> resourceClass, String resource) {
    T model = loadResource(resourceClass, resource);  // load model in that case Kafka model representation
    String modelStr = TestUtils.toYamlString(model);  
    assertDoesNotThrow(() -> createDelete(modelStr), 
    "Create delete failed after first round-trip -- maybe a problem with a defaulted value?\nApplied string: " + modelStr);
}
```

You can see that the test case basically applies the YAML representation of Kafka and expects the creation of a `Kafka` customer 
resource object in Kubernetes cluster is correct.
This is applied for every custom resource supported by `Strimzi`, such as `KafkaConnect`, `KafkaMirrorMaker`, `KafkaBridge`,
`KafkaTopic`, `KafkaUser` and so on.

### System tests

After this level is successfully completed we can talk about our system tests.
Take a look at the informal definition again:

> System testing is a level of testing that validates the complete and fully integrated software product.
> The purpose of a system test is to evaluate the end-to-end system specifications.
> Usually, the software is only one element of a larger computer-based system.
> Ultimately, the software is interfaced with other software/hardware systems.
> System Testing is actually a series of different tests whose sole purpose is to exercise the full computer-based system. (Guru)

In our case the system tests validates all components and features which that Strimzi provides.
Everything is tested in a Kubernetes environment to fully represent a production or user environment.
Again, to clarify I have an example:

```java
@Test
void testReceiveSimpleMessage() {
    KafkaTopicResource.topic(CLUSTER_NAME, TOPIC_NAME).done();

    kafkaBridgeClientJob.consumerStrimziBridge().done();

    // Send messages to Kafka
    InternalKafkaClient internalKafkaClient = new InternalKafkaClient.Builder()
        .withTopicName(TOPIC_NAME)
        .withNamespaceName(NAMESPACE)
        .withClusterName(CLUSTER_NAME)
        .withMessageCount(MESSAGE_COUNT)
        .withKafkaUsername(USER_NAME)
        .withUsingPodName(kafkaClientsPodName)
        .build();

    assertThat(internalKafkaClient.sendMessagesPlain(), is(MESSAGE_COUNT));

    ClientUtils.waitForClientSuccess(consumerName, NAMESPACE, MESSAGE_COUNT);
}

@BeforeAll
void createClassResources() throws Exception {
    deployClusterOperator(NAMESPACE);
    LOGGER.info("Deploy Kafka and KafkaBridge before tests");
    // Deploy kafka
    KafkaResource.kafkaEphemeral(CLUSTER_NAME, 1, 1).done();

    KafkaClientsResource.deployKafkaClients(false, KAFKA_CLIENTS_NAME).done();

    kafkaClientsPodName = kubeClient().listPodsByPrefixInName(KAFKA_CLIENTS_NAME).get(0).getMetadata().getName();

    // Deploy http bridge
    KafkaBridgeResource.kafkaBridge(CLUSTER_NAME, KafkaResources.plainBootstrapAddress(CLUSTER_NAME), 1)
        .editSpec()
            .withNewConsumer()
                .addToConfig(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest")
            .endConsumer()
        .endSpec()
        .done();
}
```

As you can see, this test case ensures that we are able to receive messages from the `KafkaBridge` component.
In the setup phase we deploy `cluster-operator`, a single one node Kafka cluster and also the `KafkaBridge` itself.
Everything is deployed into the real Kubernetes cluster.
All test cases with related test suites can be found in the [systemtest](https://github.com/strimzi/strimzi-kafka-operator/tree/master/systemtest) 
module.
Also worth mentioning is that our system tests currently take around ~30 hours. 
We have 3 separate sub-sets of tests, which run in parallel so it is "only" approximately 10h per each sub-set.
Additionally, there is also a mvn profile for the main groups - `acceptance`, `regression`, `smoke`, `bridge` and `all`,
but we suggest to use a profile with id `all` (default) and then include or exclude specific groups.
If you want specify the profile, use the `-P` flag, for example `-Psmoke`. Here is an example of how we trigger some profiles:

```
Generic definition

mvn -f ${workspace}/systemtest/pom.xml -P all verify -Dgroups=${testProfile} -DexcludedGroups=${excludeGroups}

Acceptance profile

mvn -f ${workspace}/systemtest/pom.xml -P all verify -Dgroups=acceptance -DexcludedGroups=""
```

All available test groups are listed in [Constants](https://github.com/strimzi/strimzi-kafka-operator/blob/master/systemtest/src/main/java/io/strimzi/systemtest/Constants.java)
class. If you're interested, you can view our testing [document](https://github.com/strimzi/strimzi-kafka-operator/blob/master/development-docs/TESTING.md).

## How system tests run in relation with Kubernetes

As mentioned in the previous section, system tests run against real Kubernetes cluster.
You can run these tests against any Kubernetes cluster type such as `Minikube`, `Minishift`, big `Kubernetes` or `Openshift`.
In the `test` module we have our a representation of a cluster which to decide on which environment it is running.
Moreover, we are using `Azure pipelines` to trigger these tests and using `Minikube`.

## How to run them

Running system tests is nothing to be scared of. In fact, it's easy.

The steps are as follows:

1. Fork the [strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator)
2. Build the whole project in root directory - `mvn clean install -DskipTests`
3. Set up your Kubernetes cluster
4. Log in to your Kubernetes cluster
5. Try to start some of our tests inside system the `test` module, for instance - `io.strimzi.systemtest.bridge.HttpBridgeST.testSendSimpleMessage()` (this takes around 5 minutes)

### Using custom images

If you are interested in running custom images, you should take a look at the file that is located in the install directory: [here](https://github.com/strimzi/strimzi-kafka-operator/blob/master/install/cluster-operator/060-Deployment-strimzi-cluster-operator.yaml).
There are few attributes that you need to change:

- `Cluster Operator` image = specify [here](https://github.com/strimzi/strimzi-kafka-operator/blob/master/install/cluster-operator/060-Deployment-strimzi-cluster-operator.yaml#L26)
- `Kafka` image = STRIMZI_KAFKA_IMAGES
- `KafkaConnect` image  = STRIMZI_KAFKA_CONNECT_IMAGES
- `KafkaConnectS2I` image =  STRIMZI_KAFKA_CONNECT_S2I_IMAGES
- `KafkaMirrorMaker` image =  STRIMZI_KAFKA_MIRROR_MAKER_IMAGES
- `KafkaMirrorMake2` image =  STRIMZI_KAFKA_MIRROR_MAKER_2_IMAGES
- `Topic Operator` image =  STRIMZI_DEFAULT_TOPIC_OPERATOR_IMAGE
- `User Operator` image =  STRIMZI_DEFAULT_USER_OPERATOR_IMAGE
- `KafkaBridge` image =  STRIMZI_DEFAULT_KAFKA_BRIDGE_IMAGE
- `CruiseControl` image =  STRIMZI_DEFAULT_CRUISE_CONTROL_IMAGE

And so on.

Here is also the list of important environment variables:

1. DOCKER_ORG = Specify the organization/repo containing the image used in system tests |`strimzi`|
2. DOCKER_TAG = Specify the image tags used in system tests |`latest`|
3. DOCKER_REGISTRY = Specify the docker registry used in system tests |`docker.io`|

### Using IDE

You can also build and run tests from your favourite IDE.
In my case, I am using InteliJ and the steps are as follows.

1. Fork the [strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator)
2. Open the strimzi-kafka-operator project
3. You should probably see on the right side maven modules. Select [systemtest](https://github.com/strimzi/strimzi-kafka-operator/tree/master/systemtest) 
module.
4. Click on Lifecycle and then you will see `clean` and `install`
5. Run any test you want

## Conclusion

In this post we had a quick dive into Strimzi's system tests domain.
Simply, We learned about the motivation behind writing system tests.
Furthermore, we talked about the main differences between system tests and unit and integration tests.
After, we looked at the relationship between system tests and Kubernetes.
Lastly, we showed how to start a system test.
Now you will be able to create your completely new system test in `Strimzi`!

Happy testing!!!