# Events

Zalando’s architecture centers around decoupled microservices and in that context we favour asynchronous event driven approaches. The guidelines in this section focus on how to design and publish events intended to be shared for others to consume. 

**Events, Event Types and Categories.**

Events are defined using an item called an _Event Type_. The Event Type allows
events to have their structure declared by producers and understood by
consumers. It declares standard information, such as a name, an owning
application (and by implication, an owning team), a schema defining the event's
custom data, and a compatibility mode declaring how the schema will be
evolved. Event Types also allow the declaration of validation and enrichment
strategies for events, along with supplemental information such as how events
can be partitioned in an event stream.

Event Types also belong to a well known _Event Category_ (such as a data change
category), which provides extra information that is common to that kind of
event.

Event Types can be published and made available as API resources for teams to
use, typically in an _Event Type Registry_. Each event published can then be
validated against the overall structure of its event type and the schema for its
custom data.

The basic model described above was originally developed in the
[Nakadi project](https://github.com/zalando/nakadi), which acts as a reference
implementation of the event type registry, and as a validating publish/subscribe
broker for event producers and consumers.

## {{ book.must }} Treat Events as part of the service interface

Events are part of a service’s interface to the outside world equivalent in standing to a service’s REST API. Services publishing data for integration must treat their events as a first class design concern, just as they would an API. For example this means approaching events with the "API first" principle in mind [as described in the Introduction](../Introduction.md).

## {{ book.must }} Make Events available for review

Services publishing event data for use by others must make the event schema available for review. 

## {{ book.must }} Ensure that Events are registered as Event Types

In Zalando's architecture, events are registered using a structure called an
_Event Type_. The Event Type declares standard information as follows:

- A well known event category, such as a general or data change category.
- The name of the event type.
- An owning application, and by implication, an owning team.
- A schema defining the event payload. 
- The compatability mode for the type.

Event Types allow easier discovery of event information and ensure that
information is well-structured, consistent, and can be validated.

Event type owners must pay attention to the choice of compatability mode.  The
mode provides a means to evolve thee schema. The range of modes are designed to
be flexible enough so that producers can evolve schemas while not inadvertently
breaking existing consumers:

- 'compatible': Consumers can reliably process events produced under different
  versions, such that every event published since the first version is remains
  valid with respect to the newest schema. When in compatible mode, it's allowed
  to add new optional properties and definitions to an existing schema, but no
  other changes are allowed. Under this mode, the following json-schema
  attributes are not supported: `not`, `patternProperties`,
  additionalProperties` and `additionalItems`.

- 'forward': Compatible schema changes are allowed and it's possible to use the
  full json schema specification for defining schemas. Consumers of forward
  compatible event types can safely read events tagged with the latest schema
  version as long as they follow the robustness principle described in the
  guideline's [API Design Principles](../DesignPrinciples.md).

- 'none': Any schema modification is accepted, even if it might break existing
  producers or consumers. When validating events, no additional properties are
  accepted unless explicitly stated in the schema.

The core Event Type structure is shown below as an Open API object definition:

```yaml
EventType:
    description: | 
      An event type defines the schema and its runtime properties. The required
      fields are the minimum set the creator of an event type is expected to
      supply.
    required:
      - name
      - category
      - owning_application
      - schema    
    properties:
      name:
        description: |
          Name of this EventType.  Note: the name can encode the
          owner/responsible for this EventType and ideally should follow a
          common pattern that makes it easy to read and understand.
        type: string
        pattern: '[a-zA-Z][-0-9a-zA-Z_]*(\.[a-zA-Z][-0-9a-zA-Z_]*)*'
        example: order.order_cancelled, acme-platform.users
      owning_application:
        description: Indicator of the application owning this `EventType`.
        type: string
        example: price-service
      category:
        description: Defines the category of this EventType. 
        type: string
        x-extensible-enum:
          - data
          - business
      compatibility_mode:
        description: |
          The compatibility modes are:

          - 'compatible': This means consumers can reliably process events
            produced under different versions, such that every event published
            since the first version is remains valid with respect to the newest
            schema. When in compatible mode, it's allowed to add new optional
            properties and definitions to an existing schema, but no other
            changes are allowed. Under this mode, the following json-schema
            attributes are not supported: `not`, `patternProperties`,
            additionalProperties` and `additionalItems`.
          
          - 'forward': Compatible schema changes are allowed. It's possible to
          use the full json schema specification for defining schemas. Consumers
          of forward compatible event types can safely read events tagged with
          the latest schema version.

          - 'none': Any schema modification is accepted, even if it might break
          existing producers or consumers. When validating events, no additional
          properties are accepted unless explicitly stated in the schema.
          
        type: string
        default: forward
      schema:
        description: The most recent payload schema for this EventType. 
        type: object
        properties:
          version:
            description: | 
               Values are based on semantic versioning. Changes to `title` or
              `description` are considered PATCH level changes. Adding new
              optional fields is considered a MINOR level change. All other
              changes are considered
              MAJOR level.                        
            type: string
            default: '1.0.0'
          created_at:
            description: Creation timestamp of the schema. 
            type: string
            readOnly: true
            format: date-time
            example: '1996-12-19T16:39:57-08:00'
          type:
            description: |            
               The schema language of schema definition. Currently only
               json_schema (JSON Schema v04) is defined, but in the future there
               could be others.            
            type: string
            x-extensible-enum:
              - json_schema
          schema:
            description: The schema as string in the syntax defined in the field type. 
            type: string
        required:
          - type
          - schema
      created_at:
        description: When this event type was created.      
        type: string
        pattern: date-time
      updated_at:
        description: When this event type was last updated.      
        type: string
        pattern: date-time
```

APIs such as registries supporting event types, may extend the model, including
the set of supported categories and schema formats. For example the Nakadi API's
event category registration also allows the declaration of validation and
enrichment strategies for events, along with supplemental information, such as
how events are partitioned in the stream.

## {{ book.must }} Ensure Events conform to a well-known Event Category

An _event category_ describes a generic class of event. The guidelines define
two such categories:

- Business Event: a category used for business process events.

- Data Change Event: a category used for describing changes to data entities.

The set of categories is expected to evolve in the future.

A category describes a predefined structure that event publishers must conform
to along with standard information about that kind of event (such as the
operation for a data change event).

**The Business Category.**

The structure of the _Business Category_ is shown below as an Open API
Schema Object definition:

```yaml
  BusinessEvent:
    description: |
      A general kind of event. Event kinds based on this event define their
      custom schema payload as the top level of the document, with the
      "metadata" field being required and reserved for standard metadata. An
      instance of an event based on the event type thus conforms to both the
      EventMetadata definition and the custom schema definition. Previously 
      this category was called the Business Category with the event 
      type registration name of "business" instead of "general".
    required:
      - metadata  
    properties:
      metadata:
          $ref: '#/definitions/EventMetadata'

```

Event types based on the Business Event define their custom schema payload at the
top-level of the document, with the `metadata` field being reserved for standard
information (the contents of `metadata` are described further down in this
section).

In the example fragment below, the reserved metadata field is shown with fields
"a" and "b" being defined as part of the custom schema:

<pre style="font-size: .85em">
  {
    <span style="font-weight: bold;">"metadata" : {...}</span>
    <span style="color: blue;">"a": "a1",</span>
    <span style="color: blue;">"b": "b1"</span>
  }
</pre>

**The Data Change Category.**

The _Data Change Category_ structure is shown below as an Open API Schema Object:

```yaml
  DataChangeEvent:
     description: |
        Represents a change to an entity. The required fields are those expected
        to be sent by the producer, other fields may be added by intermediaries
        such as a publish/subcribe broker. An instance of an event based on the
        event type conforms to both the DataChangeEvent's definition and the
        custom schema definition.      
    required:
      - metadata
      - data_op
      - data_type
      - data    
    properties:
      metadata:
        description: The metadata for this event.
        $ref: '#/definitions/EventMetadata'
      data:
        description: |    
          Contains custom payload for the event type. The payload must conform
          to a schema associated with the event type declared in the metadata
          object's `event_type` field.                
        type: object
      data_type:     
        type: string
        example: 'pennybags:order'
      data_op:
        type: string
        enum: ['C', 'U', 'D', 'S']
        description: |
          The type of operation executed on the entity:

          - C: Creation of an entity
          - U: An update to an entity.
          - D: Deletion of an entity.
          - S: A snapshot of an entity at a point in time.
```

The Data Change Category is structurally different to the Business Category. It defines
a field called `"data"` for placing the custom payload information, as well as
specific information related to data changes in the `data_type`. In the example
fragment below, the fields `"a"` and `"b"` are part of the custom payload housed
inside the `"data"` field:

<pre style="font-size: .85em">
  {
    <span style="font-weight: bold;">"metadata": {...}</span>
    <span style="font-weight: bold">"data_op": "C"</span>
    <span style="font-weight: bold">"data_type": "example.order"</span>
    <span style="font-weight: bold">"data": {</span>
      <span style="color: blue;">"a": "a1",</span>
      <span style="color: blue;">"b": "b1"</span>
    <span style="font-weight: bold">}</span>
  }
</pre>

**Event Metadata.**

The Business and Data Change categories share a common structure for
_metadata_. The metadata structure is shown below as an Open API Schema Object:

```yaml
  EventMetadata:
    type: object
    description: | 
      Carries metadata for an Event along with common fields. The required
      fields are those expected to be sent by the producer, other fields may be
      added by intermediaries such as publish/subscribe broker.      
    required:
      - eid
      - occurred_at      
    properties:
      eid:
        description: Identifier of this event.
        type: string
        format: uuid
        example: '105a76d8-db49-4144-ace7-e683e8f4ba46'
      event_type:
        description: The name of the EventType of this Event. 
        type: string
        example: 'example.important-business-event'
      occurred_at:
        description: When the event was created according to the producer.
        type: string
        format: date-time
        example: '1996-12-19T16:39:57-08:00'
      received_at:      
        description: |           
          When the event was seen by an intermediary such as a broker.
        type: string
        readOnly: true
        format: date-time
        example: '1996-12-19T16:39:57-08:00'
      version:
        description: |    
          Version of the schema used for validating this event. This may be
          enriched upon reception by intermediaries. This string uses semantic
          versioning.          
        type: string
        readOnly: true
      parent_eids:
        description: |
          Event identifiers of the Event that caused the generation of 
          this Event. Set by the producer.      
        type: array
        items:
          type: string
          format: uuid
        example: '105a76d8-db49-4144-ace7-e683e8f4ba46'
      flow_id:
        description: | 
          A flow-id for this event (corresponds to the X-Flow-Id HTTP header).          
        type: string
        example: 'JAh6xH4OQhCJ9PutIV_RYw'
      partition:
        description: |
          Indicates the partition assigned to this Event. Used for systems where
          an event type's events can be sub-divided into partitions.          
        type: string
        example: '0'
```

## {{ book.must }} Ensure that Events define useful business resources
 
 Events are intended to be used by other services including business process/data analytics and monitoring. They should be based around the resources and business processes you have defined for your service domain and adhere to its natural lifecycle (see also "Should: Define useful resources" in the [General Guidelines](../general-guidelines/GeneralGuidelines.md)).
 
As there is a cost in creating an explosion of event types and topics, prefer to define event types that are abstract/generic enough to be valuable for multiple use cases, and avoid publishing event types without a clear need.

## {{ book.must }} Events must not provide sensitive customer personal data.

Similar to API permission scopes, there will be Event Type permissions passed via an OAuth token supported in near future. In the meantime, teams are asked to note the following:

 - Sensitive data, such as (e-mail addresses, phone numbers, etc) are subject to strict access and data protection controls. 
 
 - Event type owners **must not** publish sensitive information unless it's mandatory or neccessary to do so. For example, events sometimes need to provide personal data, such as delivery addresses in shipment orders  (as do other APIs), and this is fine.

## {{ book.must }} Use the Business Events Category to signal steps and arrival points in business processes

When publishing events that represent steps in a business process, event types
must be based on the Business Category.

All your events of a single business process will conform to the following rules:

- Business events must contain a specific identifier field (a business process id or "bp-id") similar to flow-id to allow for efficient aggregation of all events in a business process execution.

- Business events must contain a means to correctly order events in a business process execution. In distributed settings where monotically increasing values (such as a high precision timestamp that is assured to move forwards) cannot be obtained, the `parent_eids` data structure allows causal relationships to be declared between events.

- Business events should only contain information that is new to the business process execution at the specific step/arrival point.

- Each business process sequence should be started by a business event containing all relevant context information.

- Business events must be published reliably by the service.

At the moment we cannot state whether it's best practice to publish all the events for a business process using a single event type and represent the specific steps with a state field, or whether to use multiple event types to represent each step. For now we suggest assessing each option and sticking to one for a given business process.

## {{ book.must }} Use Data Change Events to signal mutations

When publishing events that represents created, updated, or deleted data, change
event types must be based on the Data Change Category.

- Change events must identify the changed entity to allow aggregation of all related events for the entity.
- Change events should [contain a means of ordering](#should-provide-a-means-of-event-ordering) events for a given entity.
- Change events must be published reliably by the service.

## {{ book.should }} Provide a means for explicit event ordering

Some common error cases may require event consumers to reconstruct event streams or replay events from a position within the stream. Events *should* therefore contain a way to restore their partial order of occurence.

This can be done - among other ways -  by adding
- a strictly monotonically increasing entity version (e.g. as created by a database) to allow for partial ordering of all events for an entity
- a strictly monotonically increasing message counter

System timestamps are not necessarily a good choice, since exact synchronization of clocks in distributed systems is difficult, two events may occur in the same microsecond and system clocks may jump backward or forward to compensate drifts or leap-seconds. If you use system timestamps to indicate event ordering, you must carefully ensure that your designated event order is not messed up by these effects. 

**Note** that basing events on data structures that can be converged upon in a distributed setting (such as [CRDTs](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type), [logical clocks](https://en.wikipedia.org/wiki/Logical_clock) and [vector clocks](https://en.wikipedia.org/wiki/Vector_clock)) is outside the scope of this guidance.

## {{ book.should }} Use the hash partition strategy for Data Change Events

The `hash` partition strategy allows a producer to define which fields in an event are used as input to compute a logical partition the event should be added to. Partitions are useful as they allow supporting systems to scale their throughput while provide local ordering for event entities.

The `hash` option is particulary useful for data changes as it allows all related events for an entity to be consistently assigned to a partition, providing a relative ordered stream of events for that entity. This is because while each partition has a total ordering, ordering across partitions is not assured by a supporting system, thus it is possible for events sent across partitions to appear in a different order to consumers that the order they arrived at the server.

When using the `hash` strategy the partition key in almost all cases should represent the entity being changed and not a per event or change identifier such as the `eid` field or a timestamp. This ensures data changes arrive at the same partition for a given entity and can be consumed effectively by clients.

There may be exceptional cases where data change events could have their partition strategy set to be the producer defined or random options, but generally `hash` is the right option - that is while the guidelines here are a "should", they can be read as "must, unless you have a very good reason". 

## {{ book.should }} Ensure that Data Change Events match API representations

A data change event's representation of an entity should correspond to the REST API representation. 

There's value in having the fewest number of published structures for a service. Consumers of the service will be working with fewer representations, and the service owners will have less API surface to maintain. In particular, you should only publish events that are interesting in the domain and abstract away from implementation or local details - there's no need to reflect every change that happens within your system.

There are cases where it could make sense to define data change events that don't directly correspond to your API resource representations. Some examples are -
 
 - Where the API resource representations are very different from the datastore representation, but the physical data are easier to reliably process for data integration.

 - Publishing aggregated data. For example a data change to an individual entity might cause an event to be published that contains a coarser representation than that defined for an API

 - Events that are the result of a computation, such as a matching algorithm, or the generation of enriched data, and which might not be stored as entity by the service. 


## {{book.must}} Permissions on events must correspond to API permissions
If a resource can be read synchronously via a REST API and read asynchronously via an event, the same read-permission must apply: We want to protect access to data, not the way data is accessed.


## {{ book.must }} Indicate ownership of Event Types

Event definitions must have clear ownership - this can be indicated via the `owning_application` field of the EventType. 

Typically there is one producer application, which owns the EventType and is responsible for its definition, akin to how RESTful API definitions are managed. However, the owner may also be a particular service from a set of multiple services that are producing the same kind of event.

## {{ book.must }} Define Event Payloads in accordance with the overall Guidelines

Events must be consistent with other API data and the API Guidelines in general. 

Everything expressed in the [Introduction to these Guidelines](../Introduction.md) is applicable to event data interchange between services. This is because our events, just like our APIs, represent a commitment to express what our systems do and designing high-quality, useful events allows us to develop new and interesting products and services. 

What distinguishes events from other kinds of data is the delivery style used, asynchronous publish-subscribe messaging. But there is no reason why they could not be made available using a REST API, for example via a search request or as a paginated feed, and it will be common to base events on the models created for the service’s REST API. 

The following existing guideline sections are applicable to events -

- [General Guidelines](../general-guidelines/GeneralGuidelines.md) 

- [Naming](../naming/Naming.md)

- [Data Formats](../data-formats/DataFormats.md)

- [Common Data Objects](../common-data-objects/CommonDataObjects.md)

- [Hypermedia](../hyper-media/Hypermedia.md) - 

## {{ book.must }} Maintain backwards compatibility for Events

Changes to events must be based around making additive and backward compatible changes. This follows the guideline, "Must: Don’t Break Backward Compatibility" from the [Compatibility guidelines](../compatibility/Compatibility.md). 

In the context of events, compatibility issues are complicated by the fact that producers and consumers of events are highly asynchronous and can’t use content-negotiation techniques that are available to REST style clients and servers. This places a higher bar on producers to maintain compatibility as they will not be in a position to serve versioned media types on demand. 

For event schema, these are considered backward compatible changes, as seen by consumers  -

- Adding new optional fields to JSON objects.
- Changing the order of fields (field order in objects is arbitrary).
- Changing the order of values with same type in an array.
- Removing optional fields.
- Removing an individual value from an enumeration.

These are considered backwards-incompatible changes, as seen by consumers  -

- Removing required fields from JSON objects.
- Changing the default value of a field.
- Changing the type of a field, object, enum or array.
- Changing the order of values with different type in an array (also known as a tuple).
- Adding a new optional field to redefine the meaning of an existing field (also known as a co-occurrence constraint).
- Adding a value to an enumeration (note that [`x-extensible-enum`](../compatibility/Compatibility.md#should-used-openended-list-of-values-xextensibleenum-instead-of-enumerations) is not available in JSON Schema).

## {{ book.should }} Avoid `additionalProperties` in event type definitions

Event type schema should avoid using `additionalProperties` declarations, in 
order to support schema evolution.

Events are often intermediated by publish/subscribe systems and are 
commonly captured in logs or long term storage to be read later. In 
particular, the schemas used by publishers and consumers can  
drift over time. As a result, compatibility and extensibility issues 
that happen less frequently with client-server style APIs become important 
and regular considerations for event design. The guidelines recommend the 
following to enable event schema evolution:

- Publishers who intend to provide compatibility and allow their schemas 
  to evolve safely over time must define new optional fields 
  (as additive changes) and update their schemas in advance of publishing 
  those fields. In doing so, they **must not** declare an `additionalProperties` 
  field as a wildcard extension point. 

- Consumers must ignore fields they cannot process and not raise 
  errors, just as they would as an API client. This can happen if they are 
  processing events with an older copy of the event schema than the one
  containing the new definitions specified by the publishers. 

Requiring event publishers to define their fields ahead of publishing avoids 
the problem of _field redefinition_. This is when a publisher defines a 
field to be of a different type that was already being emitted, or, is 
changing the type of an undefined field. Both of these are prevented by 
not using `additionalProperties`. 

See also "Treat Open API Definitions As Open For Extension By Default"  
in the [Compatibility](../compatibility/Compatibility.md#must-treat-open-api-definitions-as-open-for-extension-by-default) 
section for further guidelines on the use of `additionalProperties`. 

## {{ book.must }} Use unique Event identifiers

The `eid` (event identifier) value of an event must be unique.

The `eid` property is part of the standard metadata for an event and gives the event an identifier. Producing clients must generate this value when sending an event and it must be guaranteed to be unique from the perspective of the owning application. In particular events within a given event type's stream must have unique identifiers. This allows consumers to process the `eid` to assert the event is unique and use it as an idempotency check. 

Note that uniqueness checking of the `eid` might be not enforced by systems consuming events and it is the responsibility of the producer to ensure event identifiers do in fact distinctly identify events. A straightforward way to create a unique identifier for an event is to generate a UUID value.

## {{ book.should }} Design for idempotent out-of-order processing

Events that are designed for [idempotent](#must-use-unique-event-identifiers) out-of-order processing allow for extremely resilient systems: If processing an event fails, consumers and producers can skip/delay/retry it without stopping the world or corrupting the processing result. 

To enable this freedom of processing, you must explicitly design for idempotent out-of-order processing: Either your events must contain enough information to infer their original order during consumption or your domain must be designed in a way that order becomes irrelevant.

As common example similar to data change events, idempotent out-of-order processing can be supported by sending the following information:

- the process/resource/entity identifier,
- a [monotonically increasing ordering key](#should-provide-a-means-of-event-ordering) and
- the process/resource state after the change.

A receiver that is interested in the current state can then ignore events that are older than the last processed event of each resource. A receiver interested in the history of a resource can use the ordering key to recreate a (partially) ordered sequence of events.

## {{ book.must }} Follow conventions for Event Type names

Event types can follow these naming conventions (each convention has its own should, must or could conformance level) -
 
 - Event type names must be url-safe. This is because the event type names may appear in URLs published by other systems and APIs.

 - Event type names should be lowercase words and numbers, using hyphens, underscores or periods as separators.

## {{ book.must }} Prepare for duplicate Events

Event consumers must be able to process duplicate events.

Most message brokers and data streaming systems offer “at-least-once” delivery. That is, one particular event is delivered to the consumers one or more times. Other circumstances can also cause duplicate events.

For example, these situations occur if the publisher sends an event and doesn't receive the acknowledgment (e.g. due to a network issue). In this case, the publisher will try to send the same event again. This leads to two identical events in the event bus which have to be processed by the consumers. Similar conditions can appear on consumer side: an event has been processed successfully, but the consumer fails to confirm the processing.
