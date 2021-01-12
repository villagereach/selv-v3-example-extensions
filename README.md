# OpenLMIS Example Extensions
This example is a Docker image containing extensions of OpenLMIS services. It is meant to demonstrate how extensions are added to openlmis-ref-distro.


## Quick start
1. Fork/clone this repository from GitHub.
 ```shell
 git clone https://github.com/villagereach/selv-v3-extensions-config.git
 ```
2. Add an environment file called `.env` to the root folder of the project, with the required 
project settings and credentials. For a starter environment file, you can use [this 
one](https://raw.githubusercontent.com/OpenLMIS/openlmis-ref-distro/master/settings-sample.env). eg:
 ```shell
 curl -o .env -L https://raw.githubusercontent.com/OpenLMIS/openlmis-ref-distro/master/settings-sample.env
 ```

3. Start up the application.
 ```shell
 docker-compose -f ref-distro-example-docker-compose.yml up
 ```
4. Check if the application behavior has changed according to the implemented extension point. 

5. Bean of implemented extension point should be also visible in docker-compose logs.
   To see extended logs add this loggers to the env file of ref-distro.
```
 logging.level.org.springframework.beans.factory=DEBUG
 logging.level.org.springframework.core.io.support=DEBUG
 logging.level.org.springframework.context.annotation=DEBU
```


## Integration with selv-v3-ref-distro
1. Fork/clone `selv-v3-ref-distro` repository from GitHub.
 ```shell
 git clone https://github.com/villagereach/selv-v3-distro.git
 ```
2. Start up selv-v3-ref-distro.
 ```shell
    docker-compose -f docker-compose.selv-v3-fulfillment-extension.yml up
 ```
 
## <a name="extensionpoints">Adding extension points</a>
1. Add extension to the "dependencies" configuration in build.gradle:
```
    extension "org.openlmis:selv-v3-fulfillment-extension:0.0.1-SNAPSHOT"
```
2. Modify extensions.properties with name of the extended component.
```
    AdjustmentReasonValidator=NoneValidator
    FreeTextValidator=ReasonFreeTextValidator
    UnpackKitValidator=NoKitsValidator
```


## <a name="configuringrefdistro">Configuring selv-v3-ref-distro</a>
The Reference Distribution is configured to use extension modules by defining a named volume that is common to the service and partner image. 
```
volumes:
  extensions-config:
    external: false
```
The shared volume contains extension jars and extension point configuration. The role of this image is to copy them at start-up, so they may be read by the service.

An example configuration can be found in selv-v3-ref-distro as `docker-compose.selv-v3-fulfillment-extension.yml`.

## Implementing new Extension Point in OpenLMIS
#### Prepare core service to enable extensions
1. Add [ExtensionManager](https://github.com/OpenLMIS/openlmis-fulfillment/blob/master/src/main/java/org/openlmis/fulfillment/extension/ExtensionManager.java) 
and [ExtensionException](https://github.com/OpenLMIS/openlmis-fulfillment/blob/master/src/main/java/org/openlmis/fulfillment/extension/ExtensionException.java) 
to src/extension in the repository where extension point should be included [(See commit here)](https://github.com/OpenLMIS/openlmis-stockmanagement/commit/610845042a33ae6391e79b8492ab4be9ed2f4478).
2. Add extension point interface in src/extension/point, for example: [AdjustmentReasonValidator](https://github.com/OpenLMIS/openlmis-fulfillment/blob/master/src/main/java/org/openlmis/fulfillment/extension/point/OrderNumberGenerator.java).
3. Add final ExtensionPointId class with the extension point ids defined like [here](https://github.com/OpenLMIS/openlmis-fulfillment/blob/master/src/main/java/org/openlmis/fulfillment/extension/point/ExtensionPointId.java#L20).
4. Add Default class which implements the extension point interface. See an [example](https://github.com/OpenLMIS/openlmis-fulfillment/blob/master/src/main/java/org/openlmis/fulfillment/domain/Base36EncodedOrderNumberGenerator.java).
Default class needs to have Component annotation with the value of its id:
```
@Component(value = "DefaultAdjustmentReasonValidator")
```
5. Update the usage of the extension point. Now it should use ExtensionManager to find proper implementation. See example [here](https://github.com/OpenLMIS/openlmis-fulfillment/blob/413a2991f5c89815cc76084ca5edeaac1a4097a6/src/main/java/org/openlmis/fulfillment/service/OrderService.java#L125). 
6. Add runtime dependency to build.gradle file in the repository like [here](https://github.com/OpenLMIS/openlmis-stockmanagement/blob/8e9ccf50a7b9e141bb7d4fae225fead9514b1b8f/build.gradle#L73).
7. In build.gradle add tasks that sign archives and publishes repository itself to Maven (check details in this [ticket](https://openlmis.atlassian.net/browse/OLMIS-6954)).
8. Run the CI build job to publish the repository to Maven.
#### Extension point implementation and usage
1. Create a new extension module, which contains code that overrides extension point, for example: [selv-v3-fulfillment-extension](https://github.com/villagereach/selv-v3-fulfillment-extension).
2. Annotate your implementation of the extension point with @Component annotation with the value of its id like [here](https://github.com/OpenLMIS/openlmis-fulfillment/blob/master/src/main/java/org/openlmis/fulfillment/domain/Base36EncodedOrderNumberGenerator.java#L26).
3. Create an appropriate CI job ([example](http://build.openlmis.org/job/OpenLMIS-stockmanagement-validator-extension/)). Build the job to publish the repository to Maven.
4. Create a new extensions module which collects extension points for all services. The [selv-v3-extensions-config](https://github.com/villagereach/selv-v3-extensions-config) is an example of such a module.
5. Modify [extensions.properties](https://github.com/villagereach/selv-v3-extensions-config/blob/master/extensions.properties#L2) with the name of the extended component 
6. Add the extension to the "dependencies" configuration in [build.gradle](https://github.com/villagereach/selv-v3-extensions-config/blob/master/build.gradle#L14).
7. Create a dedicated docker-compose.yml file with the extensions-config service. See the example: [docker-compose.selv-v3-fulfillment-extension.yml](https://github.com/villagereach/selv-v3-ref-distro/blob/master/docker-compose.selv-v3-fulfillment-extension.yml).
7. Add the extensions module as [volume](https://github.com/villagereach/selv-v3-ref-distro/blob/master/docker-compose.selv-v3-fulfillment-extension.yml) to the extended service in the docker-compose.yml file.
