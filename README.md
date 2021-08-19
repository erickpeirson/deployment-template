# Flibble service <!-- Name of the application -->

<!-- A brief description (1-2 sentences) that describes what the application does. -->
Rates cats that come across the cat-veyor queue, and publishes an `enraged` event to a topic when a cat makes it very cross.

<!-- A ToC is not strictly necessary, but it does help your reader find information rapidly. Most IDEs have markdown support that includes ToC generation. -->
  - [Deployment model](#deployment-model)
  - [System context & critical workflows](#system-context--critical-workflows)
  - [Care & feeding](#care--feeding)
    - [Signs of happiness](#signs-of-happiness)
    - [Signs of sadness](#signs-of-sadness)
    - [Monitoring & SLOs](#monitoring--slos)
      - [Latency](#latency)
      - [Throughput](#throughput)
      - [Error rate](#error-rate)

## Deployment model

<!-- Provide an overview of runtime modality of the application, including anything that might impact scaling/replication. -->
Stateless application. Runs a single process that may spawn a configurable number of worker threads to perform cat rating. May be scaled horizontally up to the lesser of either the queue max connections or the output topic broker max connections divided by the number of threads per replica of the application. 

## System context & critical workflows

<!-- Summarize how the application interacts with other services in the system. A diagram is a nice way to convey high-level relationships. The image below is dynamically rendered PlantUML. Check out https://plantuml.com/text-encoding for more information.-->
[![](https://www.plantuml.com/plantuml/png/FSf13i8W44RXlQUO2t21nZIn2pTkh9vWA7-1G8O4XlRwniQwVU-REphCUC_HsepX3N6Dc0GxBQoN-Rklnfp_jYGfUuRp-1if2ghH1wMoqYbVh6Ya0OVvLJC-U4qyrP9GXsUtERP0aCeUZh11z0C0)](http://www.plantuml.com/plantuml/uml/FSf13i8W44RXlQUO2t21nZIn2pTkh9vWA7-1G8O4XlRwniQwVU-REphCUC_HsepX3N6Dc0GxBQoN-Rklnfp_jYGfUuRp-1if2ghH1wMoqYbVh6Ya0OVvLJC-U4qyrP9GXsUtERP0aCeUZh11z0C0)

<!-- Briefly describe each relationship and, to the extent known/understood, the service level expectations that the application has for its interactant. Summarize how the application behaves when the interaction is degraded and/or fails. -->
- **Queue** - Consumes cats that have been placed on the queue by the ratings API as a result of a client request.
  - Expects round-trip timing to obtain and acknowledge a new cat of 99.99% @ < 100ms
  - If the queue goes away, will periodically attempt to reconnect. Won't rate any cats until a connection is restored.
- **Kafka broker** - Publishes `enraged` events to a topic via a Kafka broker, so that other components in the system know to "watch out!". 
  - Expects publish time of 99.99% @ < 50ms. 
  - If the broker goes away, will periodically attempt to reconnect. Will continue to rate cats, and `enraged` events will be buffered by the Kafka client until a connection to the broker is restored.

## Care & feeding

### Resource envelope

<!-- Summarize the resources required to operate the application. These may be estimates; just be sure to note the assumptions used to arrive at these estimates. -->

Benchmarked in a test cluster on AWS (us-east-1).

|            | Memory (Mi) | CPU (ms) | Network (kbps)
| ----       | ----------- | -------- | --------------
| Minimum    | 50          | 50       | --
| Nominal[1] | 200         | 200      | 50[3]
| Maximum[2] | 600         | 1000     | 250[3]

[1]: Assumes typical workload of 10 cats per second
[2]: Assumes peak workload of 50 cats per second
[3]: Assumes average cat size is 5kb

### Signs of happiness

<!-- How do you know that the application is working? Describe one or several observable signs that the application is doing its job correctly. -->

- For about every 1 in 10 cats that enter the queue, an `enraged` event will come across the configured topic;
- The `flibble_cats_rated_count` metric should increase monotonically with the number of cats that enter the queue;

### Signs of sadness

<!-- What kinds of things happen when the application is not working correctly? Describe any known failure modes, and how those modes manifest at the application or system level. As this section grows, it may be desirable to break out incident runbooks into separate documents. -->

- When the Flibble service has trouble accessing cats on the queue, it may get bored and start eating up memory just for giggles;
- There is a corner case in which the Flibble service encounters 5 cats that make it very cross, causing the main process to lock up permanently;

### Monitoring & SLOs

<!-- What should we measure to assess application function and anticipate/detect system faults? 

This section is based on the idea (due to Google) of the "four golden signals," with the slight adjustment that throughput and saturation are not explicitly distinguished. 

This example assumes that we are using a pull-based telemetry system like Prometheus to monitor applications. -->

Exposes metrics at `/metrics` via TCP:9000. All metrics are prefixed with `flibble_`.

#### Latency

<!-- Latency isn't just for describing how we handle client requests! Think of this as "how long does it take to do a unit of work?". -->

**SLO**: 99.99% of cats rated in < 500ms total.

| Name                       | Type    | Description
|-----                       |-----    | -----------
| `rating_time_total_ms`     | gauge   | The total amount of time taken to rate a cat, in milliseconds. 
| `rating_time_queue_ms`     | gauge   | The amount of time that a cat spent on the queue before being rated, in milliseconds.
| `rating_time_retrieve_ms`  | gauge   | The amount of time taken to retrieve a cat from the queue, in milliseconds.
| `rating_time_proc_ms`      | gauge   | The amount of processing time taken to rate a cat, in milliseconds.

#### Throughput

<!-- How many units of work are we doing per unit time? How fast ought we be able to go without falling over? -->

**SLO**: up to 50 cats per second (CPS) handled successfully without latency degradation below SLO.

| Name                       | Type    | Description
|-----                       |-----    | -----------
| `cats_rated_count`         | counter | Number of cats that have been successfully rated by the Flibble service.
| `enraged_count`            | counter | Number of times that the Flibble service has become enraged.
| `thread_count`             | counter | Number of worker threads performing cat rating.

#### Error rate

<!-- The application will fail some percentage of the time. We need a clear-eyed view of what level of failure is acceptable.

Not all errors are necessarily a sign of a system fault: there may be classes of failure that can be handled gracefully by the application, and may trigger contingency processes in other parts of the system. This example focuses specifically on errors that are not explicitly handled by the application. -->

**SLO**: 99.99% of cats are rated successfuly (without an unhandled exception).

| Name                       | Type    | Description
|-----                       |-----    | -----------
| `error_count`              | counter | Number of errors encountered, by code.

Error codes:
- `0` - No error!
- `1` - Cat cannot be rated; outside training support
- `2` - Cat is a dog
- `3` - Unhandled error
