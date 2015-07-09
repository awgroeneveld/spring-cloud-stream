# Spring Cloud Streams: Messaging as a Microservice

This project allows a user to develop and run messaging microservices using Spring Integration and run them locally, or in the cloud, or even on Spring XD. It also allows a user to develop and run an XD module locally. Just create `MessageChannels` "input" and/or "output" and add `@EnableChannelBinding` and run your app as a Spring Boot app (single application context).  You just need to connect to the physical broker for the bus, which is automatic if the relevant bus implementation is available on the classpath. The sample uses Redis.

Here's a sample source module (output channel only):

```
@SpringBootApplication
@EnableChannelBinding
@ComponentScan(basePackageClasses=ModuleDefinition.class)
public class ModuleApplication {

  public static void main(String[] args) throws InterruptedException {
    SpringApplication.run(ModuleApplication.class, args);
  }

}

@Configuration
public class ModuleDefinition {

  @Value("${format}")
  private String format;

  @Bean
  public MessageChannel output() {
    return new DirectChannel();
  }

  @Bean
  @InboundChannelAdapter(value = "output", autoStartup = "false", poller = @Poller(fixedDelay = "${fixedDelay}", maxMessagesPerPoll = "1"))
  public MessageSource<String> timerMessageSource() {
    return () -> new GenericMessage<>(new SimpleDateFormat(format).format(new Date()));
  }

}
```

The `application.yml` has the mapping from channel names to external broker handles (queues, topics, routing keys, etc. depending on the broker), e.g.

```
---
spring:
  cloud:
    channels:
      outputChannelName: ${spring.application.name:ticker}
```

To be deployable as an XD module in a "traditional" way you need `/config/*.properties` to point to any available Java config classes (via `base_packages` or `options_class`), or else you can put traditional XML configuration in `/config/*.xml`. You don't need those things to run as a consumer or producer to an existing XD system, but you do need to replace the `outputChannelName` with `group`, `module` and `index` (the index is a sequential counter that XD uses to label the modules in a stream from left to right). There's an XML version of the same sample (a "timer" source).

## Multiple Input or Output Channels

A module can have multiple input or output channels. Instead of just one channel named "input" or "output" you can add multiple `MessageChannel` beans named `input.*` or `output.*` and the names are converted to external channel names on the broker. The external channel names are the `spring.cloud.streams.[input|output]ChannelName` plus the `MessageChannel` bean name, period separated. In addition, the bean name can be `input.[queue|topic|tap]:*` or `output.[queue|topic]:*` (i.e. with a channel type as a colon-separated prefix), and the semantics of the external bus channel changes accordingly (a tap is like a topic). For example, you can have two `MessageChannels` called "output" and "output.topic:foo" in a module with `outputChannelName=bar`, and the result is 2 external channels called "bar" and "topic:foo.bar".

## XD Module Samples

There are several samples, all running on the redis transport (so you need redis running locally to test them):

* `source` is a Java config version of the classic "timer" module from Spring XD. It has a "fixedDelay" option (in milliseconds) for the period between emitting messages.

* `sink` is a Java config version of the classic "log" module from Spring XD. It has no options (but some could easily be added), and just logs incoming messages at INFO level.

* `tap` is the same as the sink sample, except it is configured to tap the source sample output. When it is running it looks a lot like the sink, except that it only gets copies of the messages in the broker, and since it is a pub-sub subscriber, it only gets the messages sent since it started.

* `source-xml` is a copy of the classic "timer" module from Spring XD.

If you run the source and the sink and point them at the same redis instance (e.g. do nothing to get the one on localhost, or the one they are both bound to as a service on Cloud Foundry) then they will form a "stream" and start talking to each other. All the samples have friendly JMX and Actuator endpoints for inspecting what is going on in the system.

## Module or App

Code using this library can be deployed as a standalone app or as an XD module. In standalone mode you app will run happily as a service or in any PaaS (Cloud Foundry, Lattice, Heroku, Azure, etc.). Depending on whether your main aim is to develop an XD module and you just want to test it locally using the standalone mode, or if the ultimate goal is a standalone app, there are some things that you might do differently.

### Module Options

Module option (placeholders) default values can be set in `/config/*.properties` as per a normal XD module, and they can be overridden at runtime in standalone mode using standard Spring Boot configuration (e.g. `application.yml`). Because of the way XD likes to organize options, the default values can also be set as `option.*` in `bootstrap.yml` (in standalone mode) or as System properties (generally).

### Local Configuration

The `application.yml` and `bootstrap.yml` files are ignored by XD when deploying the module natively, so you can put whatever you like in there to control the app in standlone mode.

### Fat JAR

You can run in standalone mode from your IDE for testing. To run in production you can create an executable (or "fat") JAR using the standard Spring Boot tooling, but the executable JAR has a load of stuff in it that isn't needed if it's going to be deployed as an XD module. In that case you are better off with the normal JAR packaging provided by Maven or Gradle.

## Making Standalone Modules Talk to Each Other

The `[input,output]ChannelName` are used to create physical endpoints in the external broker (e.g. `queue.<channelName>` in Redis).

For an XD module the channel names are `<group>.<index>` and a source (output only) has `index=0` (the default) and downstream modules have the same group but incremented index, with a sink module (input only) having the highest index. To listen to the output from a running XD module, just use the same "group" name and an index 1 larger than the app before it in the chain.

> Note: since the same naming conventions are used in XD, you can steal messages from or send messages to an existing XD stream by copying the stream name (to `spring.cloud.streams.group`) and knowing the index of the XD module you want to interact with.

## Taps

All output channels can be also tapped so you can also attach a module to a pub-sub endpoint and listen to the tap if you know the module metadata. To tap an existing vanilla module you need to know its `outputChannelName` and the tap name is then `tap:<outputChannelName>`, so you can listen to it on an input channel named `input.topic.tap:<outputChannelName>`. The tap is only active if you explicitly ask for it: you can do that by POSTing to the HTTP endpoint `/taps/<channelName>` (where the channel name can be the internal or external name, e.g. "output" or the external name mapped to the output channel).

To tap an existing output channel in an XD module you just need to know its group, name and index, e.g.

```
spring:
  cloud:
    channels:
      group: tocktap
      name: logger
      index: 0
      tap:
        group: testtock
        name: ticker
        index: 0
```

The `spring.cloud.channels.tap` section tells the module runner which topic you want to subscribe to. It creates a new group (a tap can't be in the same group as the one it is tapping) and starts a new index count, in case anyone wants to listen downstream.

## Build Spring Cloud Streams
### Pre-requisites

 * Required :
    * Java 8
    * Maven

Currently the `receptor-client` dependency is not in a public Maven repo. To install it to your local Maven repo, execute the following commands:

```
git clone https://github.com/markfisher/receptor-client.git
cd receptor-client
./gradlew install
```
 
### Building the project

```
mvn clean install
```
