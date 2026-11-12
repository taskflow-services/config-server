# Config Server

A [Spring Cloud Config](https://docs.spring.io/spring-cloud-config/reference/server.html) server that hands out shared configuration to the other services in the system.

- **Spring Boot** 4.1.2, **Spring Cloud** 2025.1.4, **Java** 25
- Listens on port **8888**
- Uses the `native` profile, so config is read from files on disk rather than a Git repository

> Note: the package is `com.config_server`, because `com.config-server` is not a valid Java package name.

## Shared configuration

Config files live in [src/main/resources/config/](src/main/resources/config/). Everything in `application.properties` there is served to **every** client application:

| Property | Value | Purpose |
| --- | --- | --- |
| `eureka.client.service-url.defaultZone` | `http://localhost:8761/eureka/` | Eureka discovery server location |
| `eureka.instance.hostname` | `localhost` | Hostname clients register with |
| `eureka.instance.prefer-ip-address` | `false` | Register by hostname, not IP |
| `logging.pattern.level` | `%5p [%X{correlationId:-}]` | Adds the correlation ID to log lines |
| `management.tracing.sampling.probability` | `0.5` | Trace 50% of requests |

To give one service its own settings, add a file named after its `spring.application.name`, for example `config/order-service.properties`. Profile-specific files follow the usual pattern, for example `order-service-dev.properties`.

## Running locally

```sh
./mvnw spring-boot:run
```

Check that it is serving config (replace `my-service` with any application name):

```sh
curl http://localhost:8888/my-service/default
```

The config location is set in [src/main/resources/application.properties](src/main/resources/application.properties):

```properties
spring.cloud.config.server.native.search-locations=file:///mnt/c/Users/linte/Documents/programs/config-server/src/main/resources/config/
```

This is a WSL path on one machine. On any other machine, change it, or override it when you start the server:

```sh
./mvnw spring-boot:run -Dspring-boot.run.arguments=--spring.cloud.config.server.native.search-locations=classpath:/config/
```

## Running with Docker

The [Dockerfile](Dockerfile) builds the jar in a JDK image, then runs it in a slim JRE image (with `curl` included for health checks).

```sh
docker build -t config-server .
docker run -p 8888:8888 \
  -e SPRING_CLOUD_CONFIG_SERVER_NATIVE_SEARCH_LOCATIONS=classpath:/config/ \
  config-server
```

The `/mnt/c/...` path does not exist inside the container, so the search location has to be overridden. `classpath:/config/` uses the config files baked into the jar. To edit config without rebuilding the image, mount a directory instead:

```sh
docker run -p 8888:8888 \
  -v "$(pwd)/src/main/resources/config:/config" \
  -e SPRING_CLOUD_CONFIG_SERVER_NATIVE_SEARCH_LOCATIONS=file:/config/ \
  config-server
```

## Connecting a client

In a client service, add the `spring-cloud-starter-config` dependency and point it at this server:

```properties
spring.application.name=my-service
spring.config.import=optional:configserver:http://localhost:8888
```

## Reference documentation

- [Spring Cloud Config Server](https://docs.spring.io/spring-cloud-config/reference/server.html)
- [Spring Cloud Config Client](https://docs.spring.io/spring-cloud-config/reference/client.html)
- [Spring Boot Maven Plugin](https://docs.spring.io/spring-boot/4.1.2/maven-plugin)
- [Apache Maven](https://maven.apache.org/guides/index.html)
