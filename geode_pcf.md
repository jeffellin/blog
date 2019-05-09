# Connecting to an External Gemfire / Geode from Cloud Foundry

The new Spring Boot Data Geode supports autoconfiguration of the Client Cache when deploying an app to Pivotal Cloud Foundry (PCF).  When using Pivotal Cloud Cache (PCC), this makes deployment very easy.  The locator addresses and the credentials are automatically retrieved from the service binding.


Not every app uses PCC when wanting to connect to a cache. When developing locally, I generally use the following Spring Boot properties to set the connection information.

```
spring.data.gemfire.locators=somelocator[10334]
```

While this generally works fine but there may be problems when pushing this app to PCF.

One way around this is to create the ClientCache manually.

```java
@Bean
public ClientCache cache(){

    ClientCache cache = new ClientCacheFactory().addPoolLocator("somelocator", 10334).
    .setPdxReadSerialized(true)
    .create();
    
    return cache;

}
```

As a Spring Boot loyalist, I don't like this.  I would much rather use configuration. 

I should be declaratively set up my cache in `application.properties` as follows.

```
spring.data.gemfire.locators=somelocator[10334]
spring.data.gemfire.pdx.read-serialized=true

```

This works great until I deploy my app to PCF. When starting the app, I get the following error message.

```
No service with tags [cloudcache, gemfire] was found
```

This occurs because of the way autoconfiguration works in SBDG.

## SBDG Autoconfiguration

Auto Configuration of the Gemfire client is as follows.


1. ClientSecurityAutoConfiguration
1. ClientCacheAutoConfiguration

The specific problem that occurs in the above scenario is that we would like to use `ClientCacheAutoConfiguration,` but `ClientSecurityAutoConfiguration` fails because there are no security settings present. 


The specific problem is that when pushed to PCF, SBDG attempts to configure security.  It does this by trying to look for a bound service with the tag `cloudcache` or `gemfire` 

This happens even if you specify:
`security.username` and `security.password`

### ClientSecurityAutoConfiguration

In an attempt to debug this issue, I tried running my application locally using the `cloud` profile.  With the `cloud` profile enabled the app still started fine.  It turns out the AutoConfiguration class doesn't specifically use the `cloud` profile.  It checks to see if the app is running in `CloudFoundry.`  It does this by checking for the presence of a `VCAP_SERVICES` environment variable in `isCloudFoundryEnvironment().` 



```java
Optional.of(environment)
.filter(this::isEnabled)
.filter(this::isCloudFoundryEnvironment)
.ifPresent(this::configureSecurityContext);
```

Since the app has determined it is running in cloud foundry, it attempts to retrieve the credentials from `VCAP_SERVICES.`

```java
CloudCacheService cloudCacheService = vcapPropertySource.findFirstCloudCacheService();

```

### Disabling Security AutoConfiguration.

The easiest way to deal with this is to disable Security AutoConfiguration.

```
spring.boot.data.gemfire.security.auth.environment.post-processor.enabled
```

pushing the app at this point should successfully allow it to connect to the cluster.


