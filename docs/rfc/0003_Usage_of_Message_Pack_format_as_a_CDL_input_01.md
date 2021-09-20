# Front Matter

```
    Title           : Usage of MessagePack format as a CDL input
    Author(s)       : Mateusz 'esavier' Matejuk
    Team            : CommonDataLayer
    Reviewer        : CommonDataLayer
    Created         : 2021-02-17
    Last updated    : 2021-02-17
    Category        : Feature
    CDL Feature ID  : CDLF-0000E-00
```

#### Abstract

```
     The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL
     NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and
     "OPTIONAL" in this document are to be interpreted as described in
     RFC 2119.
```
[RFC 2119 source][rfc2119]

## Glossary

##### Terminology
* CDL - [Common Data Layer Project][cdl-project]
* DR - Data Router, an CDL component responsible for ingesting and routing initial messages.
* SR - Schema Registry, an CDL component responsible for keeping information about the type of the object conveyed inside the message.
* User - user of the CDL. In this case, the user knows how the CDL works, and is assumed to have access to the API and code unless stated otherwise.
* Message - (abstract) message sent to CDL for processing
* Breaking Change - change in behavior, or API of the CDL that may result with system breakage on release update.
* MD - man-day - amount of work completed by one developer in one work day.

##### Formats
v1 Input Format - messages of format v1 are not related to Message Batching, i.e. batch can consist of a list of messages, of version v1, but array itself is not a message format. Input Message v1 is formatted as follows:
```
	{
	    object_id: UUID,
	    schema_id: UUID,
	    data: { any valid json },
	}
```
v1 Batch - it is a batch format introduced to alleviate some problems with transports. It conforms only to the v1 input format, and its treatment is described in RFC for batching
```
[
	{  v1 message }
	{  v1 message }
	{  v1 message }
	...
	=> n
]
 ```
## Introduction

#### Background
In present design, DR can only ingest JSON input format, which is both inefficient, and proves to be troublesome with some features. One example of which is being binary format ingestion. It was proposed to introduce MessagePack as an alternative. MessagePack is as flexible as JSON, meaning there is no need to recompile with each alteration in the protocol, and there is no need to utilize the message schema before initiating the communication. In addition, MessagePack allows the program to maintain binary data as is, without extra escaping or encoding. The Scope of this document include only information about the CDL input transport, which is, by extension, communication between user and DR.

#### Assumptions
We have to assume that some users may want to retain the ability to communicate over plain JSON, and/or choose a specific format for a specific job, meaning that two formats may have to be employed at all times.

#### Preliminary Testing

##### Performance
Simple preliminary testing using rust, C, and zig (while zig implementation being quite humble) displays some improvements to using message pack format over pure JSON string.
* testing was performed over 1 000 000 loops, and averaged per one encoding.
* results shown are generated by code in C, but those were similar in scale for the two other languages

| #                            | JSON        | message pack |
|------------------------------|-------------|--------------|
| avg encoding of one message  | 3.11 × 10-4 | 7.850 × 10-5 |
| avg decoding of one message  | 7.91 × 10-4 | 1.81 × 10-4  |

Similar results can be depicted in readily available publications:
[MessagePack vs Json benchmark][msgpack-benchmark]

#####  Length of the Payload
Example serialization shows that the same message that was serialized in MessagePack was smaller than in JSON.
| # | JSON    | message pack | difference | compression ratio
|---|---------|--------------|------------|------------------|
| # | 121 615 | 101 411      | 20 204      | 0.8338

##### Limitations
* integer values  are limited to 64 bytes
* maximum length of binary object is (2^32) -1 bytes (4294967295 bytes or around 3.9 GiB)
* maximum length of string object is (2^32) -1
* String may be malformed, or be a non-valid UTF-8 sequence
* maximum array length is (2^32) -1
* maximum key-value map length is (2^32) -1
[Source: MessagePack Specification][msgpack-spec]
* During research, it was apparent that only a few rust libraries support full MessagePack features, for example extensions.

## Solutions

##### Current or Existing Solution
DR can only ingest properly formatted JSON messages, consisting of either a single v1 object or an array of separate v1 objects, each potentially unrelated to each other. This means that all special characters and payloads have to be escaped or encoded in some way. Until now, it was proposed to use base64 encoding, however, for each 3 bytes encoded, base64 encoded equivalent requires 4 bytes to be transmitted, meaning 33% increase in payload  (not necessary stored), potentially also inside the CDL network.

#### Proposed Solutions - Preamble
There are few different ways of controlling the message recognition, and each one have its drawbacks. The program can either be configured prior to the runtime to expect different messages on different communication mediums, be it kafka topic, or GRPC endpoint, or the code can be placed to expect different markers describing the message type and format and act on those.

Proposed below are detailed designs on how this issue can be handled:

##### Solution I - Separate Endpoints
Each format will have a corresponding abstract transport endpoint. In this case, for each available ingestion method we have to specify different handling methods, due to the nature of each communication medium:
*  Kafka would receive a separate topic to use with MessagePack, or use one topic, however in that case, each partition would have different type of messages, and format mixing would have to be avoided
 * RabbitMQ would receive separate queue to use with MessagePack
 * GRPC would have separate endpoint

###### Notes
* No testing nor PoC was created for this solution, please treat it as being theoretical.
* Having separate endpoints will result in CDL not being able to keep the ordering as described in the ordering rfc. It has to be noted that this should not be a problem, as it would be highly unordinarily to interact with a system that, while being focused on message ordering, is using multiple message formats at once. Nevertheless, this limitation have to be considered and, if this solution will be chosen, additional documentation have to be prepared, and this limitation MUST be highlighted.

###### Major Concerns
* This design will result in multiple configurations that have to be tested separately of each other and in tandem.
* Some amount of additional configuration have to be added.
* Separate endpoints can be misused, resulting in a cascade of errors. Additional error handling has to be introduced to discern the type of issue, and whatever it was related to format errors or not.
* Parallel usage of two or more endpoints may end up in thread starvation on constrained machines. In such example, by having one thread available for the workload, the focus would be shifted to one endpoint over another. This is an infrastructure related issue, but it has to be considered in this context.

###### Performance
* From initial evaluation it seems that this solution should not have any considerable performance drawbacks.
* Parallel usage can result in performance degradation, however this is only a theoretical issue, and was not proven by testing yet due to pre-PoC state of the feature and no observed evidences of this behavior in other parts of  the CDL.

##### Solution II - single endpoint with metadata carrier
Both formats can use this same, abstract, transport endpoint. All the information required to control DR behavior and informing about the state of the message, will be provided in the transport's metadata. This means that we can retain the simplicity of having one, non-specific endpoint, where both message formats will be received.

 After preliminary research, it seems that all the protocols can, in theory, pass metadata context alongside the payload.

##### Notes
* For GRPC transport, information about the format would have to be contained in the header, or separate GRPC call
* For Kafka and RabbitMQ, metadata can be stored in headers.
* Detached tests were executed, in which CDL context was not used, mostly focused on the availability and usability of the libraries, and it’s support. Additionally, PoC was not created for this solution, please treat it as being theoretical.

##### Major Concerns
* All the future transports will have to support sending some kind of metadata alongside the payload itself. This is quite possible but not guaranteed for all the users in all the use cases. In case new transport will be proposed, and it will not support it, it will have to be discarded or result in reopening this feature while choosing another solution.
* Each transport have to get additional code, handling different ways of getting, and possibly parsing, the metadata provided with the message.
* It was also stated on the internal meetings that clients and/or libraries in different languages may have problems with this functionality, as it is not widely used. As per project directives, CDL have to be compatible with different languages, and this may be a potential issue.

##### Performance
* Assuming the difference in metadata formats and our ability to both parse and serialize it, it can be safely assumed that performance will not suffer, however it depends on the way each library will get and use the metadata provided with the message. Comparing to the usual `O(m*m℘)` where `m` is the message size and `m℘` is the parsing cost, this solution will cost roughly `O((m*m℘) + (n*n℘))` where `n` and `n℘` is metadata length and its parsing cost respectively.

##### Solution III - single Endpoint With Format Recognition
In this solution, both formats can be sent to one transport endpoint, similarly to the design proposed in Solution II, the message will not contain metadata in itself, but parsing will be performed for each available serialization method. In case all the methods fail, the message will be considered malformed and handled the usual way, which means reporting the error and continuing to work on the next queued message.


##### Notes
* The Nature of this solution and its performance review, introduces the term "Cost of Failure" which is a measure `𝜅` that can be described as `0<𝜅<=m`, where m is the length of the message.
	- It describes the ability of the underlying library to recognize the format or fail in the case error.  The earlier the library can recognize parsing error, the lower is the `𝜅`, and the earlier code can react to the error.
	 - Moreover, `𝜅` is also related to the attached function, in that case`n*𝜅` means:
	 `for x in n, return 𝜅(x)`
	 -  It has to be, by definition fluid, and if not argued otherwise, taken at worst-case scenario rate.
	- In the layman's terms, lower `𝜅` is better, and until stated otherwise, defaults to `𝜅==m`
* Detached tests were executed, in which CDL context was not used, additionally PoC was not created for this solution, please treat it as being theoretical.

##### Major Concerns
* Cost of Failure will scale alongside the number of formats. Currently, this is limited to two which this document describes, although this is only true for the current state, and it may or may not change in the future.
* Different libraries providing support for either JSON and MessagePack can behave differently. Cost of Failure, in those, is not documented or at least not easily available. Due to that, it would be wise that to assume the worst scenario, which is `𝜅==m` for each deserialization method.

##### Performance
* Comparing the performance to the other designs mentioned before, clearly shows that it is potentially more costly, ranging anywhere from `0` to `n*𝜅` where `n` is the number of formats CDL supports (currently this document describes the second format) assuming the worst scenario where `𝜅=m`

##### Potential Improvements
* assuming that one of the deserialization methods will fail, it is possible to parallelize those and to try to get at least one result out of several methods. This may improve performance to up to a single `𝜅`, while degrading memory usage (due to the possibility that deserialization methods will be destructive in the context of the used language) and requiring n threads to start working on the same message at the same time. This may prove helpful in cases when the user will want to use different formats and mix messages. Withal, it will be more troublesome to use for users that are using either one or the other.

* Format recognition can be adjusted per queue. Assuming that the user will either send one format or the other, for each queue, it is possible for DR to keep information in its cache or configuration, informing code what to expect. The proposed solution would be to add a counter for each transport, that would track the ratio of formats that were successfully recognized and in which format they appear to be sent. This would help "guess" which of the deserialization methods have the best chances of success on a given queue. This will improve performance of this design in both edge cases (one type of message arriving via transport) and in case of mixed messages, without introducing severe performance issues.

##### Solution IV - Single Endpoint With Deserialization Marker
This design uses some kind of marker that allows to easily discover what type of message arrived. There is potentially a few different ways to solve it each with its issue, all the same it is more of an expansion. Solutions like this are very low-level and usually are, but not always, tailored for specific usage and not generic, which is the opposite of what CDL tries to be.

1. First byte:
	If we can assume that the JSON formatted message will start  with either byte 0x5B "`[`" or 0x7B "`{`".
	In this case, we can check the first character that arrives, and decide the message format on that information. This also may be implemented as an improvement for Solution III. This method assumes that the message will not start with a non-whitespace character, and MessagePack will not use those bytes for its own serialization methods (which according to its specification will be the case). Please note that this option may result in unnecessary and heavy CPU branching if done incorrectly, while not providing many tools and methods to prevent it.

2. Extension (header) check:
	This is the same method as the one adverted above, nonetheless in this case, correct behavior can be ensured using MessagePack's extension types. Unfortunately, it is unsure by reading the specification whenever the header or appendix is created to accommodate extension data. In this case, Empiric testing was proven successful in determining that for a specific implementation, `messagepack-rs` it is indeed a header consisting of bytes `0xD6` which, according to specification, represent the `ext` family of formats. However, one issue is apparent, which is a severely lacking support for user extensions in rust libraries, of which at least one out of 5 supported it. Rust support were found in the "First that works" approach, and it is unclear what support is available for the other languages, nevertheless support is apparent and well-defined in MessagePack specification.
[MessagePack Specification][msgpack-spec]

3. Marker injection
	Last and "dirtiest" option is to inject the required specific byte before or after the payload. This option is guaranteed to break support on the client side, and simply judging from the sheer volume of changes on both sides of the CDL system, is heavily discouraged.

##### Notes
Suggested changes require alteration in v1 specification to force users to trim the messages from white spaces before committing them to the transport.
The Cost of Failure for this solution is 0(1), presumably also confined to one branch operation.

##### Major Concerns
* Changes to existing v1 spec that are not clear or apparent. It means that v1 format will not change, but the way of delivery will. It is unclear right now how to announce those changes and if those should result in v2 format specification or not, also it is not known if the trimming should be used on the side of the receiver or the sender.
* Depending on specification and partial loss of flexibility - DR will now employ another set of restrictions, however minor, to the message format to be able to correctly and concisely discern the incoming content format.

##### Solution V - Single Endpoint With Specific Deserialization Focus
The Method presented here will be the simplest, albeit with some drawbacks. Taking into consideration the scalability of the DR, we can provide multiple DR instances with configurations that are in counterbalance to each other. Providing this is the second format that is intended to be supported, two instances should be perfectly able to cover all the cases, in which the first instance will look for JSON formatted messages, and the second will respond on MessagePack payload.
This method will work with different configurations, especially in the case in which the user will use explicitly one format over another and use CDL system with that knowledge in hand. Configuration MUST be provided during the application startup.

###### Major Concerns
* This design will have to be extended with better error handling that the one that is present currently. In case there will be mixed-message scenario, where multiple formats are used in tandem, there is a guarantee that either instance will throw a deserialization error, as it will be configured to handle a different format.

###### Notes
No testing nor PoC was created for this solution, please treat it as being theoretical.

## Further Considerations

#### Impact on Other Teams
Depending on the chosen solution, we may have to notify all users about the changes. It may also be necessary to introduce those as a "breaking change".

#### Scalability
Using Solution I may result in additional endpoints that have to be either provided or created. Apart from that, from initial research, there appear to be no impact on scalability whatsoever.

#### Availability Problems
Some of the solutions depends on the specific usage and/or implementation of the specific libraries and languge support. It is encouraged to put additional effort in weighting the specific solution against the other.

#### Testing
This feature MUST undergo thorough testing before being accepted. Test cases MUST include:
* Failure scenarios:
	- Edge scenarios where messages are unreadable or follow worst-case scenario path.
* Malformed messages:
	- reaction of the software to malformed or misaligned messages.
* Happy test:
	- Proper behavior with proper values.
* Format testing for each, readily available repository type:
	- Document storage.
	- Time series storage.
	- Binary storage.
* Performance testing with all the before mentioned cases:

#### Workload Estimation
Success rate should be acceptable, considering there is not a lot of solution-sepcific logic to introduce. Depending on which solution is chosen. It is initially roughly estimated to 20MD

## Deliberation

#### Out of Scope
* Any other, not mentioned features, were not taken into consideration in this scope. Especially versioning (CDLF-00010-00)
* Communication within CDL system itself, and an output format are out of the scope of this document.

#### Open Questions
* Usage and behavior in Service mesh can not be checked at this point. This RFC should be revisited when the feature is complete and testing can be performed.
* CDLF-0000D-00 - Service Meshing - format handling changes should not affect Service Meshing itself, however at this point in time it is hard to guarantee that.

#### Notes
* CDLF-00009-00 - Message Batching - have to be taken into consideration while writing this feature
* CDLF-0000A-00 - Message Ordering - guaranteeing order in case of mixed formats will not be possible, albeit while using a specific format or different endpoint, order can be enforced.

### End Matter
This is the version of the RFC that was deliberately cut down to ingestible size and format. The number of cases and solutions were condensed to those that were considered by the researcher the most valuable, leaving out minor variation that can be deduced from already provided solutions.

#####  References
MessagePack related materials:
[MessagePack Specification](https://github.com/msgpack/msgpack/blob/3f0a9aae716596a86878c0d68dc0bd4256673202/spec.md)
[MessagePack Benchmark](https://thephp.website/en/issue/messagepack-vs-json-benchmark/)
[MessagePack Project](https://msgpack.org/)

CDL :
[CDL project](https://github.com/epiphany-platform/CommonDataLayer)

References to notable rust libraries used in research:
[messagepack-rs (library with user extensions for Rust)](https://lib.rs/crates/messagepack-rs)

[msgpack-spec]: https://github.com/msgpack/msgpack/blob/3f0a9aae716596a86878c0d68dc0bd4256673202/spec.md
[msgpack-benchmark]: https://thephp.website/en/issue/messagepack-vs-json-benchmark/
[rfc2119]:https://www.ietf.org/rfc/rfc2119.txt
[cdl-project]:https://github.com/epiphany-platform/CommonDataLayer