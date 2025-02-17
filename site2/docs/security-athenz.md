---
id: security-athenz
title: Authentication using Athenz
sidebar_label: "Authentication using Athenz"
---

[Athenz](https://github.com/AthenZ/athenz) is a role-based authentication/authorization system. In Pulsar, you can use Athenz role tokens (also known as *z-tokens*) to establish the identity of the client.

## Enable Athenz authentication

A [decentralized Athenz system](https://github.com/AthenZ/athenz/blob/master/docs/decent_authz_flow.md) contains an [authori**Z**ation **M**anagement **S**ystem](https://github.com/AthenZ/athenz/blob/master/docs/setup_zms.md) (ZMS) server and an [authori**Z**ation **T**oken **S**ystem](https://github.com/AthenZ/athenz/blob/master/docs/setup_zts) (ZTS) server.

To begin, you need to set up Athenz service access control. You need to create domains for the *provider* (which provides some resources to other services with some authentication/authorization policies) and the *tenant* (which is provisioned to access some resources in a provider). In this case, the provider corresponds to the Pulsar service itself and the tenant corresponds to each application using Pulsar (typically, a [tenant](reference-terminology.md#tenant) in Pulsar).

### Create the tenant domain and service

On the [tenant](reference-terminology.md#tenant) side, you need to do the following things:

1. Create a domain, such as `shopping`
2. Generate a private/public key pair
3. Create a service, such as `some_app`, on the domain with the public key

Note that you need to specify the private key generated in step 2 when the Pulsar client connects to the [broker](reference-terminology.md#broker) (see client configuration examples for [Java](client-libraries-java.md) and [C++](client-libraries-cpp.md)).

For more specific steps involving the Athenz UI, refer to [Example Service Access Control Setup](https://github.com/AthenZ/athenz/blob/master/docs/example_service_athenz_setup.md#client-tenant-domain).

### Create the provider domain and add the tenant service to some role members

On the provider side, you need to do the following things:

1. Create a domain, such as `pulsar`
2. Create a role
3. Add the tenant service to members of the role

Note that you can specify any action and resource in step 2 since they are not used on Pulsar. In other words, Pulsar uses the Athenz role token only for authentication, *not* for authorization.

For more specific steps involving UI, refer to [Example Service Access Control Setup](https://github.com/AthenZ/athenz/blob/master/docs/example_service_athenz_setup.md#server-provider-domain).

## Enable Athenz authentication on brokers

> ### TLS encryption 
>
> Note that when you are using Athenz as an authentication provider, you had better use TLS encryption 
> as it can protect role tokens from being intercepted and reused. (for more details involving TLS encryption see [Architecture - Data Model](https://github.com/AthenZ/athenz/blob/master/docs/data_model)).

In the `conf/broker.conf` configuration file in your Pulsar installation, you need to provide the class name of the Athenz authentication provider as well as a comma-separated list of provider domain names.

```properties
# Add the Athenz auth provider
authenticationEnabled=true
authorizationEnabled=true
authenticationProviders=org.apache.pulsar.broker.authentication.AuthenticationProviderAthenz
athenzDomainNames=pulsar

# Enable TLS
tlsEnabled=true
tlsCertificateFilePath=/path/to/broker-cert.pem
tlsKeyFilePath=/path/to/broker-key.pem

# Authentication settings of the broker itself. Used when the broker connects to other brokers, either in same or other clusters
brokerClientAuthenticationPlugin=org.apache.pulsar.client.impl.auth.AuthenticationAthenz
brokerClientAuthenticationParameters={"tenantDomain":"shopping","tenantService":"some_app","providerDomain":"pulsar","privateKey":"file:///path/to/private.pem","keyId":"v1"}
```

> A full listing of parameters is available in the `conf/broker.conf` file, you can also find the default
> values for those parameters in [Broker Configuration](reference-configuration.md#broker).

## Configure Athenz authentication in Pulsar clients

To use Athenz as an authentication provider, you need to [use TLS](#tls-encryption) and provide values for four parameters in a hash:
* `tenantDomain`
* `tenantService`
* `providerDomain`
* `privateKey`

You can also set an optional `keyId`. The following is an example.

```java
Map<String, String> authParams = new HashMap();
authParams.put("tenantDomain", "shopping"); // Tenant domain name
authParams.put("tenantService", "some_app"); // Tenant service name
authParams.put("providerDomain", "pulsar"); // Provider domain name
authParams.put("privateKey", "file:///path/to/private.pem"); // Tenant private key path
authParams.put("keyId", "v1"); // Key id for the tenant private key (optional, default: "0")

Authentication athenzAuth = AuthenticationFactory
        .create(AuthenticationAthenz.class.getName(), authParams);

PulsarClient client = PulsarClient.builder()
        .serviceUrl("pulsar+ssl://my-broker.com:6651")
        .tlsTrustCertsFilePath("/path/to/cacert.pem")
        .authentication(athenzAuth)
        .build();
```

#### Supported pattern formats
The `privateKey` parameter supports the following three pattern formats:
* `file:///path/to/file`
* `file:/path/to/file`
* `data:application/x-pem-file;base64,<base64-encoded value>`

## Configure Athenz authentication in CLI tools

[Command-line tools](reference-cli-tools.md) like [`pulsar-admin`](/tools/pulsar-admin/), [`pulsar-perf`](reference-cli-tools.md), and [`pulsar-client`](reference-cli-tools.md) use the `conf/client.conf` config file in a Pulsar installation.

You need to add the following authentication parameters to the `conf/client.conf` config file to use Athenz with CLI tools of Pulsar:

```properties
# URL for the broker
serviceUrl=https://broker.example.com:8443/

# Set Athenz auth plugin and its parameters
authPlugin=org.apache.pulsar.client.impl.auth.AuthenticationAthenz
authParams={"tenantDomain":"shopping","tenantService":"some_app","providerDomain":"pulsar","privateKey":"file:///path/to/private.pem","keyId":"v1"}

# Enable TLS
useTls=true
tlsAllowInsecureConnection=false
tlsTrustCertsFilePath=/path/to/cacert.pem
```

