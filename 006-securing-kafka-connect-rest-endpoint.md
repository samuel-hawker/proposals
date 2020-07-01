# Securing the Kafka Connect REST API endpoint

## Current situation

Currently, instances of Kafka Connect that are deployed by the Strimzi operators are configured with the default REST API endpoint settings. This means that the Kafka Connect REST API endpoint uses HTTP on port 8083 and that the Strimzi KafkaConnect, KafkaConnector and KafkaMirrorMaker2 operators make unsecured REST client calls based on this default configuration.

The default network policies created by the Strimzi operators restrict incoming REST API calls to only allow access from the operator pod. If required, further network policies can be provided to override the default policy and allow wider access.

## Motivation for change

Many users have asked for the ability to secure the Kafka Connect REST API endpoint.

## Proposed changes

This proposal adds a `rest` property to the KafkaConnect spec, allowing users to configure Connect REST API  endpoints.

Simple example CR:

```
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaConnect
spec:
  rest:
  - tls: {}
```

This CR would generate a cert and key in a secret (for use by the operator clients), create SSL truststores and keystores from the cert and key and set the following properties in the generated connect configuration to enable an HTTPS connection:

```
listeners: https://:8443
rest.advertised.listener: https
rest.advertised.port: 8443

listeners.https.ssl.truststore.location: /tmp/kafka/kafka-connect-rest.truststore.p12
listeners.https.ssl.truststore.password: ***generated password***
listeners.https.ssl.truststore.type: PKCS12
listeners.https.ssl.keystore.location: /tmp/kafka/kafka-connect-rest.keystore.p12
listeners.https.ssl.keystore.password: ***generated password***
listeners.https.ssl.keystore.type: PKCS12

```

A more complex example CR:

```
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaConnect
spec:
  rest:
  - port: 8083
  - port: 8483
    tls:
      configuration:
        brokerCertChainAndKey:
          secretName: mysecret
          certificate: keystore.crt
          key: keystore.key
```

Here the operator configures two listeners: an unsecured HTTP listener on port 8083 and a secured HTTPS listener on port 8483 using the certificate and key from the provided secret. This uses a similar interface as used for the Kafka CR `spec.listeners.tls` property. The operator mounts the secret certificates in the Kafka Connect pods and sets a failed status and logging if the listener options clash or are mis-configured.

The operator REST API client code will prefer HTTPS over HTTP when both are configured, in a similar way to how the Kafka Connect workers coordinate via HTTPS when it is available (see [KIP 507](https://cwiki.apache.org/confluence/display/KAFKA/KIP-507%3A+Securing+Internal+Connect+REST+Endpoints)).

The above proposal will require modifications to the `AbstractConnectOperator` to correctly read and use the additional CR configuration, apply the TLS certs as keystore credentials to a WebClient truststore and then the webclient (or its configuration) will have to be passed into the `KafkaConnectApi` interface and `Impl` so that each call can optionally use this. This should mean for regular http calls it will continue to work as is, but the calls will also be able to speak to the secured endpoint if configured.

### Compatibility

This proposal is backwards compatible - the `spec.rest` property is optional and, if not supplied (or contains an empty list), the Kafka Connect defaults will be applied (unsecured HTTP calls to port 8083) as they are today.

### Future extensions

It is also worth noting that this opens Strimzi up to further extensions in the future, such as:
 - adding support for client authentication at the Kafka Connect REST API endpoint.
 - adding external listeners (such as new routes explicitly for Kafka Connect REST calls).


## Rejected alternatives

### Use the current Kafka Connect CR to configure HTTPS

Using the currently exposed interface, users can change the Kafka Connect REST API endpoint to use HTTPS
by configuring a KafkaConnect CR as follows:

```
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaConnect
metadata:
  name: my-connect-cluster
  annotations:
    # set to true so admin point is used by operator
    strimzi.io/use-connector-resources: "true"
spec:
  version: 2.4.0
  replicas: 1

  bootstrapServers: my-cluster-kafka-bootstrap:9093
  tls:
    trustedCertificates:
      - secretName: my-cluster-cluster-ca-cert
        certificate: ca.crt
  externalConfiguration:
    volumes:
      # The cert containing all your data
      - name: cert
        secret:
          secretName: mysecret

  config:
    # important config

    # change the advertised port and protocol
    rest.advertised.listener: https
    rest.advertised.port: 8083
    listeners.https.ssl.client.auth: none
    listeners: https://:8083

    # Configure keystore with a valid cert and pass (in this case a pkcs12)
    listeners.https.ssl.keystore.location: /opt/kafka/external-configuration/cert/user.crt
    listeners.https.ssl.keystore.password: ${file:/opt/kafka/external-configuration/cert/connector.properties:user.password}
    listeners.https.ssl.keystore.type: PKCS12

    # Supplied so that  ${file:*} syntax can be used with connect config (providing password)
    config.providers: file
    config.providers.file.class: org.apache.kafka.common.config.provider.FileConfigProvider

    # non-HTTPS related config
    group.id: connect-cluster-2
    offset.storage.topic: connect-cluster-offsets
    config.storage.topic: connect-cluster-configs
    status.storage.topic: connect-cluster-status

    config.storage.replication.factor: 1
    offset.storage.replication.factor: 1
    status.storage.replication.factor: 1
```

This works to secure the Kafka Connect REST API endpoint, but requires several modifications to the operator to allow it to send REST API requests:
  - the `listener` and `rest` keys must be removed from the forbidden list
  - the vertx WebClient must be used in place of the vertx HttpClient, with each call being configured with the correct certificates.
  - the operator needs to be able to parse the `${file:*}` and password strings to infer their mount locations and thereby the secret(s) they come from.