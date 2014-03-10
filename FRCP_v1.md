# <a name="frcp"></a>Federated Resource Control Protocol (FRCP)

This document describes the *Federated Resource Control Protocol (FRCP)*, which 
may be used by any software entity for controlling and orchestrating distributed
resources, such as testbed devices, sensor nodes or measurement software. This 
protocol is currently used by 
[OMF6](https://github.com/mytestbed/omf)
entities, and may be implemented by any other software project to interact with 
other FRCP-enabled components.

This document describes the current FRCP version 2 protocol.

This document is organised as follows:

[Protocol and Interactions](#proto_inter)  
[Messaging System, Naming and Addressing](#msg_system)  
[Generic Message Syntax](#syntax_msg) - [XML](#syntax_xml) and [JSON](#syntax_json) format  
[Inform Syntax](#syntax_inform)  
[Configure Syntax](#syntax_configure)  
[Request Syntax](#syntax_request)  
[Create Syntax](#syntax_create)  
[Release Syntax](#syntax_release)  
[Optional Guard Syntax](#syntax_guard)  
[Core Properties for All Resources](#core_properties)  

----

## <a name="proto_inter"></a>Protocol and Interactions

The basic protocol consists of a **message** being sent by a **requester** to a 
**component** (or **resource**). The component may accept the message and 
perform the requested associated actions. This may lead to further messages 
being sent to an **observer**. We introduced a separate observer to allow for 
different messaging framework. For instance, in an RTP like framework the 
observer is usually the requester itself, while in a publish-subscribe 
framework the observer is a stand-in for a publication.

![https://raw.github.com/mytestbed/specification/master/fig_overview.svg](https://rawgithub.com/mytestbed/specification/master/fig_overview.svg)  

The protocol consists of five messages:
[inform](#syntax_inform),
[configure](#syntax_configure),
[request](#syntax_request),
[create](#syntax_create),
[release](#syntax_release).
A basic interaction pattern is shown in the figure below.

![https://raw.github.com/mytestbed/specification/master/fig_interaction.svg](https://rawgithub.com/mytestbed/specification/master/fig_interaction.svg)  

For many implementation it will be impractical to create a directly addressable 
message endpoint for every component and we therefore introduce an optional 
**service address** (e.g. **aggregate manager**) for each component. The 
**service address** identifies a proxy which normally serves many components, 
requiring the message to contain the component id as well.

[(jump to top)](#frcp)

----

## <a name="msg_system"></a>Messaging System, Naming and Addressing

As a generic protocol FRCP does not require a specific messaging system, or
naming or addressing conventions. However _without loss of generality_, the 
remaining of this document will assume the following context, to allow for a
easier description and illustrations of the protocol.

**Messaging System**

This document assumes that a publish-and-subscriber (PubSub) messaging system
is used between entities exchanging FRCP messages. In such a system, entities
may **subscribe** to **topics**, they will then receive any messages which have
been **published** for that topic by any other entities.

**Naming and Addressing Convention**

This document assumes that a FRCP entity has a globally unique ID, which is
selected by its creator (e.g. another FRCP entity). Interoperable deployments of
FRCP entities must agree on a convention to map this unique ID to a unique
address in the underlying messaging system. Thus while FRCP can be used with
many naming and addressing convention, FRCP deployements which want to
interoperate must use the same convention.

This document (and [the reference FRCP implementation](https://github.com/mytestbed/omf)) assumes the following convention:

* assuming that
    * an entity has the ID: `res123`
    * a PubSub messaging system is being used
    * the server providing `res123` access to that PubSub system is named `foo.com`
* then the address of the topic associated to that entity is
    * if using a XMPP-based PubSub system: `xmpp://res123@foo.com`
    * if using a AMPQ-based PubSub system: `amqp://foo.com/res123`


[(jump to top)](#frcp)

----

## <a name="syntax_msg"></a>Generic Message Syntax

FRCP messages can be described using either the XML or the JSON format. The remaining of this document will provide examples in both formats. The current schema definitions 
for FRCP messages FRCP are available here: 
[http://schema.mytestbed.net/omf/](http://schema.mytestbed.net/omf/)

###  <a name="syntax_xml"></a>Generic XML Format

The generic XML format of a FRCP message is:

        <MTYPE xmlns="http://schema.mytestbed.net/omf/X.Y/protocol" mid=ID>
          <src>RID</src>
          <ts>TIMESTAMP</ts>
          <rp>TOPIC</rp>
          ....     
        </MTYPE>

* X.Y = the version of the protocol (currently 6.0)
* MTYPE = the type of message, either `create`, `configure`, `request`, 
`inform`, or `release`
* ID = a globally unique ID for this message
* RID = a globally unique ID for the resource that published this message. Note 
that every entities are resources. Thus a software acting on behalf of a user, 
such as an experiment controller, will also have an unique resource ID.
In OMF6, this ID is in the form of a URI.
* TIMESTAMP = the Posix/Unix time in second when this message was generated by 
its publisher. This `<ts>` and the `<mid>` child elements are used to prevent 
message replays.
* TOPIC = optional. If the publisher of this message would like any replying 
messages to be published to a specific topic address, then it must set TOPIC to 
that address in the `<rp>` (i.e. "reply to") element.

This element may then have child elements, which further describe various 
information specific to the message type. The following sections of this 
document describe each message type and provide further details on the child 
elements specific to each type.

In addition to these message type specific child elements, FRCP also defines 
two other child elements which are used in various message types. These are:

* the `<guard>` element, which carries constraint information. When this child 
element is present, an entity will only act on a received message if it 
satisfies the constraints described within this element. This `<guard>` element 
is described in more detail at the end of this page.

* the `<props>` element which declare various properties related to a message 
and which is described next.

FRCP provides an optional mechanism to verify that a given message was 
published by a given entity. This optional mechanism is described in the 
[FRCP Authentication and Authorisation document]().

#### The `<props>` Properties Element

Some FRCP message may include a `<props>` child element, which declares various 
properties related to that message. This is for example the case of the 
**configure** and **request** types of message as described below.

The properties and values within a `<prop>` child element may follow any of the 
three alternative syntax (i.e. key 1 to 3) below:

        <props xmlns:foo="http://foo.com/goo">

          <foo:key1 type="...">some_other_values</foo:key1>
          
          <foo:key2>
            <val>some_value</val>
          </foo:key2>

          <foo:key3>
            <val>1024</val>
            <unit>bps</unit>
          </foo:key3>

        </props>

* foo = the namespace (e.g. a given resource type context) for the properties 
defined within this `<props>` element.
* keyi =  the name of the i-th property defined within this `<props>` element
* The latter descriptive syntax for foo:key3 uses a list of child elements to 
describe the property in more details. Basic child elements are `<val>` 
and `<unit>`, but may also include `<origin>`, `<resolution>`, or 
`<min-value>`, `<max-value>` if the property is used as a constraint in a 
**request** message.

FRCP supports default namespace setting, thus the following message is also 
valid:

        <props xmlns="http://foo.com/goo">
          <key1 type="...">some_other_values</foo:key1>
        </props>


A `key` element may have an optional `type` attribut, which (if present) gives 
the data type of the enclosed value. Valid data types are: 
`string`, `fixnum`, `boolean`, `hash`, and `array`. The syntax for each of 
these types are as follows:

* **string**

        <foo:key1 type=string>my_string_value</foo:key1>

* **fixnum**

        <foo:key2 type=fixnum>5.6</foo:key2>

* **boolean**

        <foo:key3 type=boolean>true</foo:key3>  

* **hash**

        <foo:key4 type='hash'>  
          <keyA>value_for_key4_keyA</keyA>  
          <keyB type='boolean'>true</keyB>  
          <keyC type='int'>123</keyC>  
        </foo:key4>

* **array**

        <foo:key5 type='array'>
          <it type='fixnum'>1</it>
          <it type='fixnum'>2</it>
          <it type='fixnum'>3</it>
        </foo:key5> 

The behaviours of a FRCP-enabled entity which receives a **configure** message 
with `array` or `hash` property is described in the 
[Configure Syntax section](#syntax_configure).

[(jump to top)](#frcp)

###  <a name="syntax_json"></a>Generic JSON Format

The generic JSON format of a FRCP message is:

        {
          "op": MTYPE,
          "mid": ID,
          "src": RID,
          "ts": TIMESTAMP,
          "rp": TOPIC,
          ....
        }
* The meaning of each fields is similar to the previous 
[XML syntax](#syntax_xml) above, i.e.
    * MTYPE = the type of message, either `create`, `configure`, `request`, 
`inform`, or `release`
    * ID = a globally unique ID for this message
    * RID = a globally unique ID for the resource that published this message
    * TIMESTAMP = the Posix/Unix time in second when this message was generated 
    by its publisher
    * TOPIC = optional, if the publisher of this message would like any 
    replying messages to be published to a specific TOPIC address

Similar to the above [XML syntax](#syntax_xml), the remainder of that 
JSON-formatted message may then have further objects, which further describe 
various information specific to the message type. The following sections of 
this document describe each message type and provide further details on the 
object specific to each type.

#### The `"props"` Properties Object

Similar to the above [XML syntax](#syntax_xml), some JSON FRCP message may 
include a `"props"` object, which declares various properties related to that 
message. This is for example the case of the **configure** and **request** 
types of message as described below.

The properties and values within a `"prop"` object may follow any of the 
three alternative syntax (i.e. key 1 to 3) below:

        "props": {
          "@context": "http://foo.com/goo",
          "key1": "some_other_values",        
          "key2": {
            "val": "some_value"
          },
          "key3": {
            "val": 1024,
            "unit": "bps"
          }
        }

* @context = the resource type context for the properties defined within this 
`"props"` object. Within FRCP this `@context` key is used to refer to 
the namespace within which the property keys are defined. 
    * In the current FRCP version the value for this object will always be a 
    string URI, i.e. current FRCP does not support or defined inline context 
    definition as specified in JSON-LD
    * Thus in the current FRCP version, properties within a "props" object are
    all defined under the same unique namespace, i.e. in the above example 
    `key1` and `key2` are from the same namespace.
* keyi =  the name of the i-th property defined within this `"props"` object
* The latter descriptive syntax for "key3" uses another object to 
describe the property in more details, such as `"val"`, or `"unit"`, etc...

The type of the value assigned to a `key` element may be any of the types 
supported in JSON, i.e.
`string`, `number`, `boolean`, `hash`, and `array`.

* **string**  
    "key1": "some_string_value",

* **number**  
    "key2": 12.345,

* **boolean**  
    "key3": true,

* **array**  
    "key5": [1,2,3,4,5,6]

* **hash**

        "key4": {  
          "keyA": "string_value_for_key4_keyA",  
          "keyB": true,  
          "keyC": 123  
        }



The behaviours of a FRCP-enabled entity which receives a **configure** message 
with `array` or `hash` property is described in the 
[Configure Syntax section](#syntax_configure).

[(jump to top)](#frcp)

----

## <a name="syntax_inform"></a>Inform Syntax

### Overview

A FRCP entity publishes an **inform** message to provide some information on 
some of its properties, which may have changed as a result of another message 
that it has previously received (e.g. a **configure** or a **request** 
message). In addition, a FRCP entity may also publish spontaneous or 
unsolicited **inform** messages to provide ongoing information on some of its 
properties. 

An **inform** message has the following XML syntax:

        <inform xmlns="http://schema.mytestbed.net/omf/X.Y/protocol" mid=ID>
          <src>RID</src>
          <ts>TIMESTAMP</ts>
          <it>TYPE</it>
          <cid>CID</cid>
          ....     
        </inform> 

Or alternatively the following JSON syntax:

        {
          "op": "inform",
          "mid": "ID",
          "src": "RID",
          "ts": "TIMESTAMP",
          "it":"TYPE",
          "cid": "CID",
          ....
        }

* `TYPE` =  the type of this inform message. All resources must support the 
following basic types: CREATION.OK, CREATION.FAILED, STATUS, RELEASED, ERROR, 
and WARN. In addition, any resource may define its own particular types of 
inform message, following the convention BASIC_TYPE.SPECIFIC_TYPE. For example, 
if you have a custom resource which wants to return an inform message as a 
reply to a configure message that contained a syntax error, then that custom 
resource may define and use the custom inform type ERROR.SYNTAX.

* `CID` = this is the Context ID for this message. This element or object must 
be present only if this inform message is a reply to another message, in which 
case CID must be the ID of the original message. This element is not present in 
a spontaneous inform message.

An **inform** message may further have child elements (XML) or objects (JSON) 
specific to its type. For example, a STATUS **inform** message will contain a 
`props` element/object describing the status of the resource's properties. Such 
`props` element/object was described previously in the 
[Generic Syntax section above](#syntax_msg).
The next sections will provide more details on the different types of **inform**
messages that must issued by a resource as a reply to specific received FRCP 
messages.

### Usage

An FRCP-enabled entity:

1. may publish an **inform** message spontaneously to provide unsolicited 
report about some of its properties
2. should publish an **inform** message as a reply to previously received 
**create**, **configure**, **request**, or **release** messages
3. should publish its inform message on its main topic address (e.g. this could 
be mapped from its globally unique ID as is often the case in OMF6 resources).
However, if this **inform** message is a reply to a previously received message
which had a `rp` (i.e. "reply to") element/object set, then a copy of this 
**inform** message must also be published to the topic address specified by 
that `rp`.

### Examples

An spontaneous **inform** message issued by a resource with the ID 'node123' 
and the type 'vm' (i.e. virtual machine) defined at 
'http://foo.com/virtual_machine'. This example also shows the three alternative 
syntaxes to describe property values 
(see [Generic Syntax section above](#syntax_msg)).

* XML

          <inform xmlns="http://schema.mytestbed.net/omf/6.0/protocol" mid="48fa94">
            <src>xmpp://node123@domainA.com</src>
            <ts>1360889609</ts>
            <it>STATUS</it>
            <props xmlns:vm="http://foo.com/virtual_machine">
              <vm:os>
                <value>ubuntu</value>
              </vm:os>
              <vm:osversion>12.04</vm:osversion>
              <vm:ram>
                <value>8</value>
                <unit>GB</unit>
              </vm:ram>
            </props>
          </inform>
* JSON

          {
            "op": "inform",
            "mid": "48fa94s",
            "src": "amqp://domainA.com/node123",
            "ts": "1360889609",
            "it":"STATUS",
            "props": {
              "@context": "http://foo.com/virtual_machine",
              "os": "ubuntu",        
              "osversion": {
                "val": 12.04
              },
              "ram": {
                "val": 8,
                "unit": "GB"
              }
            }          
          }


[(jump to top)](#frcp)

----

## <a name="syntax_configure"></a>Configure Syntax

### Overview

A configure message is published to a topic address to ask its subscriber(s) to 
set some of its (their) properties to some given values. 

It has the following XML or JSON syntax:

      <configure xmlns="http://schema.mytestbed.net/omf/X.Y/protocol" mid=ID>
        <src>RID</src>
        <ts>TIMESTAMP</ts>
        <props xmlns:foo="http://foo.com/goo">
           <foo:key1>...</foo:key1>
            ....     
        </props>
      </configure>


      {
        "op": "configure",
        "mid": "ID",
        "src": "RID",
        "ts": "TIMESTAMP",
        "props": {
          "@context": "http://foo.com/goo",
          "key1": ... ,
          ...
        }
      }

As discussed above in the [Generic Syntax section](#syntax_msg), the `props`
element/object may include several `key` elements/objects. Each `key` refers to
a single property of the resource, and holds the value to be set to that 
property.

### Usage

When a FRCP-enabled entity receives a **configure** message:

1. it should try to set its properties as described in the `props` part of that
message. More precisely, if `props` contains the key `foo` with the value `bar`:
    * if the receiving entity does not have a property named `foo`, then it 
    should publish back an **inform** message of type ERROR
    * if it does have a property named `foo`, then it should try to assign the 
    value `bar` to that property 'foo'

    * **About the Assignment of a Value to Propery**: 
        * when not advertised otherwise by the resource itself, *assign* in the
        general case has the meaning: "replacing the previous value X1 of type 
        Y1 held by property `foo` with the new value X2 of type Y2 provided by 
        this configure message"    
        * For example:
            * if `foo` previously held a value 8 of type number, and 
            the key/value `"foo": 23` is received, then the resource should try
            to set `foo` to the value `23` of type number
            * if `foo` previously held a value 8 of type number, and the 
            key/value `"foo": "Alice"` is received, then the resource should try
            to set `foo` to the value `Alice` of type string
            * if `foo` previously held a value [1,2,3] of type array, and the
            key/value `"foo": [9]` is received, then the resource should try to
            set `foo` to the value `[9]` of type array
        * Thus, FRCP does not mandate any type checking for properties. However,
        a specific resources is free to implement its own type checking or 
        casting as it sees fit, and return an **inform** message of type ERROR
        as a reply to a **configure** message with assignments that fail its 
        type checking
        * Furthermore, FRCP also allows a given resource to implement and
        advertise any custom assignment behaviour as it sees fit. For example, 
        when explicitly mentioned in the resource's documentation, it could 
        decide to:
            * set the value of `foo` to SHA1("Alice"), when it receives the 
            key/value `"foo": "Alice"`
            * set the value of `foo` to the array [1,2,3,9], when it receives 
            the key/value `"foo": [9]` and `foo` previously held the value 
            [1,2,3] of type array.

2. it must publish an **inform** message to its main topic address (e.g. 
typcially in OMF6 entities implementing FRCP, this would the topic derived from 
the resource's unique ID) to inform on the outcome of that configuration. This 
inform message must:
    * have its `cid` element/object set with the ID of the received 
    **configure** message (i.e. the value of `mid` element/object of the
    **configure** message).
    * provide in its `props` element/object the list of properties that have
    been modified as a result of the received **configure** message. This list
    of key/value properties should follow the `props` format described in the
    above [Generic Syntax section](#syntax_msg).

For some property the update from a previous value to a new one may take some 
time (e.g. changing the `filesystem` property of a disk resource from `ext3` to
`ext4` might require formatting, which would not be immediate). During that
updating process, FRCP defines an extended optional syntax for the `props`
element/object, which allows ongoing consecutive **inform** message to provide
progress updates about the configuration. This extended optional syntax is as 
follows for XML or JSON

        <props xmlns:foo="http://foo.com/goo">
          <foo:key1>   
            <current>V1</current>
            <target>V2</target>
            <progress>V3</progress>
            <msg>MSG</progress>
          </foo:key1>
        </props>


        "props": {
          "@context": "http://foo.com/goo",
          "key1": {
            "current": V1,
            "target": V2,
            "progress": V3,
            "msg": MSG,
          }
        }

  * `V1` = the current value of the property `key1`.
  * `V2` = the target value for `key1` as requested by the last **configure** 
  message. The configuration of `key` is assumed to be complete if and only if
   `V1` equals `V2`.
  * `V3` = optional, when present it is a progress indicator interger from 0 to 
  100, with 100 meaning that the configuration is complete (i.e. `V1` equals 
  `V2`)
  * `MSG` = optional, a text message with more information on the property 
  setting process


### Examples

These are some examples of **configure** messages and their corresponding 
**inform** reply messages, both in XML and JSON.

**example 1 (XML)**

      <configure xmlns="http://schema.mytestbed.net/omf/6.0/protocol" mid="83ty28">
        <src>xmpp://node123@domainA.com</src>
        <ts>1360895974</ts>
        <props xmlns:iperf="http://foo.com/iperf">
           <iperf:target type='hash'>
             <ip type='string'>192.168.1.2</ip>
             <port type='fixnum'>5001</port>
           </iperf:target>
        </props>
      </configure> 

      <inform xmlns="http://omf.mytestbed.net/omf/6.0/protocol" mid="d934l8">
        <src>xmpp://iperf456@domainB.com</src>
        <ts>1360895982</ts>
        <cid>83ty28</cid>
        <it>STATUS</it>
        <props xmlns:iperf="http://foo.com/iperf">
           <iperf:target type='hash'>
             <ip type='string'>192.168.1.2</ip>
             <port type='fixnum'>5001</port>
           </iperf:target>
        </props>
      </inform>

**example 1 (JSON)**

      {
        "op": "configure",
        "mid": "83ty28",
        "src": "amqp://domainA.com/node123",
        "ts": "1360895974",
        "props": {
          "@context": "http://foo.com/iperf",
          "ip": "192.168.1.2" ,
          "port": 5001
        }
      }

      {
        "op": "inform",
        "mid": "d934l8",
        "src": "amqp://domainA.com/node123",
        "ts": "1360895982",
        "cid": "83ty28",
        "it": "STATUS",
        "props": {
          "@context": "http://foo.com/iperf",
          "ip": "192.168.1.2" ,
          "port": 5001
        }
      }

**example 2 (XML)**

      <configure xmlns="http://schema.mytestbed.net/omf/6.0/protocol" mid="4gt521">
        <src>xmpp://node123@domainA.com</src>
        <ts>1360895906</ts>
        <props xmlns:iperf="http://foo.com/iperf">
           <iperf:transport>UDP</iperf:transport>
           <iperf:bitrate>
             <unit>kBps</unit>
             <value>1024</value>
           </iperf:bitrate>
        </props>
      </configure> 

      <inform xmlns="http://omf.mytestbed.net/omf/6.0/protocol" mid="1d3a9x">
        <src>xmpp://iperf456@domainB.com</src>
        <ts>1360895966</ts>
        <cid>4gt521</cid>
        <it>STATUS</it>
        <props xmlns:iperf="http://foo.com/iperf">
           <iperf:bitrate>
             <unit>kBps</unit>
             <current>512</current>
             <target>1024</target>
             <progress>50</progress>
           </iperf:bitrate>
           <iperf:transport>UDP</iperf:transport>
        </props>
      </inform>

      <inform xmlns="http://omf.mytestbed.net/omf/6.0/protocol" mid="63ha70">
        <src>xmpp://iperf456@domainB.com</src>
        <ts>1360895976</ts>
        <cid>4gt521</cid>
        <it>STATUS</it>
        <props xmlns:iperf="http://foo.com/iperf">
           <iperf:bitrate>
             <unit>kBps</unit>
             <current>1024</current>
             <target>1024</target>
             <progress>100</progress>
           </iperf:bitrate>
           <iperf:transport>UDP</iperf:transport>
        </props>
      </inform>

**example 2 (JSON)**

      {
        "op": "configure",
        "mid": "4gt521",
        "src": "amqp://domainA.com/node123",
        "ts": "1360895906",
        "props": {
          "@context": "http://foo.com/iperf",
          "transport": "UDP" ,
          "bitrate": 1024
        }
      }

      {
        "op": "inform",
        "mid": "1d3a9x",
        "src": "amqp://domainA.com/node123",
        "ts": "1360895966",
        "cid": "4gt521",
        "it": "STATUS",
        "props": {
          "@context": "http://foo.com/iperf",
          "transport": "UDP",
          "bitrate": {
            "unit": "kBps",
            "current": 512,
            "target": 1025,
            "progress": 50
          }
        }
      }

      {
        "op": "inform",
        "mid": "63ha70",
        "src": "amqp://domainA.com/node123",
        "ts": "1360895976",
        "cid": "4gt521",
        "it": "STATUS",
        "props": {
          "@vocab": "http://foo.com/iperf",
          "transport": "UDP",
          "bitrate": {
            "unit": "kBps",
            "current": 1024,
            "target": 1025,
            "progress": 100
          }
        }
      }

[(jump to top)](#frcp)

----

## <a name="syntax_request"></a>Request Syntax

### Overview

A **request** message is published to a topic address to ask its subscriber(s) 
to publish information about some properties. This message has the following
XML and JSON syntax:

      <request xmlns="http://schema.mytestbed.net/omf/X.Y/protocol" mid=ID>
        <src>RID</src>
        <ts>TIMESTAMP</ts>
        <props xmlns:foo="http://foo.com/goo">
           <foo:key1 />
           <foo:key2 />
            ....     
        </props>
      </request> 


      {
        "op": "request",
        "mid": "ID",
        "src": "RID",
        "ts": "TIMESTAMP",
        "props": {
          "@context": "http://foo.com/goo",
          "key1": "",
          "key2": "",
          ....
          }
        }
      }

In the above message, the `props` element/object only contains the list of
properties (e.g. `key1`, `key2`) for which the current values are requested.

### Usage

When a FRCP-enabled entity receives a **request** message:

1. it should publish an **inform** message, which provides the current values of
the properties listed within the `props` section of the **request** message.
This **inform** message is similar to the one sent as a reply to a 
**configure** message ([Configure Syntax section](#syntax_configure))
2. it should publish this **inform** message to its main topic address (e.g. in 
the OMF6 implementation, this topic derives from the entity's unique resource 
ID)
3. if the **request** message contains a `rp` element/object, then a copy 
of this **inform** message should also be published on the topic address given 
in that `rp` element/object

### Examples

These are some examples of **request** messages and their corresponding 
**inform** reply messages, both in XML and JSON.

**example 1 (XML)**

      <request xmlns="http://schema.mytestbed.net/omf/6.0/protocol" mid="ea4afb">
        <src>xmpp://node123@domainA.com</src>
        <ts>1360897589</ts>
        <props xmlns:iperf="http://foo.com/iperf">
          <iperf:protocol />
          <iperf:bitrate />
          <iperf:packet_size />
        </props>
      </request>

      <inform xmlns="http://omf.mytestbed.net/omf/6.0/protocol" mid="86df5f">
        <src>xmpp://iperf456@domainB.com</src>
        <ts>1360897595</ts>
        <cid>ea4afb</cid>
        <it>STATUS</it>
        <props xmlns:iperf="http://foo.com/iperf">
          <iperf:protocol>UDP</iperf:protocol>
          <iperf:bitrate>
             <unit>kBps</unit>
             <value>1024</value>
          </iperf:bitrate> 
          <iperf:packet_size">
             <unit>bytes</unit>
             <value>512</value>
        </props>
      </inform>

**example 1 (JSON)**

      {
        "op": "request",
        "mid": "ea4afb",
        "src": "amqp://domainA.com/node123",
        "ts": "1360897589",
        "props": {
          "@context": "http://foo.com/iperf",
          "protocol": "",
          "bitrate": "",
          "packet_size": ""
        }
      }

      {
        "op": "inform",
        "mid": "86df5f",
        "src": "amqp://domainA.com/node123",
        "ts": "1360897595",
        "cid": "ea4afb",
        "it": "STATUS",
        "props": {
          "@context": "http://foo.com/iperf",
          "protocol": "UDP",
          "bitrate": { "unit": "kBps", "value": 1024},
          "packet_size": { "unit": "byte", "value": 512}
        }
      }

**example 2 (XML)**

      <request xmlns="http://schema.mytestbed.net/omf/6.0/protocol" mid="g345h7">
        <src>xmpp://node123@domainA.com</src>
        <ts>1360897611</ts>
        <props xmlns:wirelessnode="http://foo.com/wirelessnode">
          <wirelessnode:child_resource_types />
        </props>
      </request>

      <inform xmlns="http://omf.mytestbed.net/omf/6.0/protocol" mid="d812h0">
        <src>xmpp://node456@domainB.com</src>
        <ts>1360897619</ts>
        <cid>g345h7</cid>
        <it>STATUS</it>
        <props xmlns:wirelessnode="http://foo.com/wirelessnode">
          <wirelessnode:child_resource_types type='Array'>
            <item type='string'>Application</item>
            <item type='string'>WiFiAtherosInterface</item>
            <item type='string'>WiFiIntelInterface</item>
            <item type='string'>WiMaxInterface</item>
          <wirelessnode:child_resource_types>
        </props>
      </inform>

**example 2 (JSON)**

      {
        "op": "request",
        "mid": "g345h7",
        "src": "amqp://domainA.com/node123",
        "ts": "1360897611",
        "props": {
          "@context": "http://foo.com/wirelessnode",
          "child_resource_types": ""
        }
      }

      {
        "op": "inform",
        "mid": "86df5f",
        "src": "amqp://domainA.com/node123",
        "ts": "1360897619",
        "cid": "d812h0",
        "it": "STATUS",
        "props": {
          "@context": "http://foo.com/wirelessnode",
          "child_resource_types": [
            "Application",
            "WiFiAtherosInterface",
            "WiFiIntelInterface",
            "WiMaxInterface"
          ]
        }
      }

[(jump to top)](#frcp)

----

## <a name="syntax_create"></a>Create Syntax

### Overview

A ***create*** message is published to a topic to ask its subscriber(s) to 
create another resource. The creator is referred to as the parent resource (P) 
and the newly created resource as the child (C). This message has the following 
XML and JSON syntax:

      <create xmlns="http://schema.mytestbed.net/omf/X.Y/protocol" mid=ID>
        <src>RID</src>
        <ts>TIMESTAMP</ts>
        <props>
          <type>TYPE</type>
          ...
        </props>
      </create> 


      {
        "op": "create",
        "mid": "ID",
        "src": "RID",
        "ts": "TIMESTAMP",
        "props": {
          "type": "TYPE",
          ...
          }
        }
      }

where TYPE is the type of child resource to create. The list of valid resource 
types can be queried from the parent resource (see the [Request Message examples]
(#syntax_request), or discovered out-of-band.

In addition to the `type` property, a ***create*** message may also carry
additional optional properties in its `props` element/object. These additional
properties should follow the syntax defined in the [General message](#syntax_msg)
and [Configure message](#syntax_configure) sections.

### Usage

When a FRCP entity P receives a **create** message:

1. P must decide if it will create the requested child resource C. This 
decision's process and policies is specific to each resource and thus outside
the scope of FRCP 
2. if P agrees to create C, then P must select and create a globally unique ID
for C and a corresponding unique topic address in the messaging system
3. then P must provision the new resource C. Depending on C's type this could
involve tasks such as starting up a new VM, starting up a new application, or
activating a device. P may either perform these tasks itself or ask
some other entities to do them. This provisioning is outside the scope of FRCP
4. if some additional properties with values are set as part of the ***create***
message, then the newly provisioned resource C must set its properties
accordingly as if it had received these values via a ***configure*** message
5. P must publish an ***inform*** message to its own topic to report on the
outcome of creation process. This message may be published as soon as the topic
address for the child resource has been created. This ***inform*** message has
the following XML and JSON syntax:

          <inform xmlns="http://schema.mytestbed.net/omf/6.0/protocol" mid=ID>
            <src>RID</src>
            <ts>TIMESTAMP</ts>
            <cid>CID</cid>
            <it>INFOTYPE</it>
            <reason>MOREINFO</reason>
            <props>
              <res_id>CHILDID</res_id>
              <type>TYPE</type>
              ...
            </props>
          </inform>

          {
            "op": "inform",
            "mid": "ID",
            "src": "RID",
            "ts": "TIMESTAMP",
            "cid": "CID",
            "it": "CREATION.OK",
            "reason": "MOREINFO",
            "props": {
              "res_id": "CHILDID",
              "type": "TYPE",
              ...
            }
          }

    Where `INFOTYPE` is equal to either `CREATION.OK` (i.e. the child resource
    was created successfully) or `CREATION.FAILED` (i.e. the child resource
    cannot be created), and `CHILDID` is the unique ID of the newly created
    resource, and `MOREINFO` is an optional message providing additional
    information on the creation process (e.g. why did it fail?)
    
6. if some some additional properties and values were given as part of the 
***create*** message, then the corresponding ***inform*** message must have in
its `props` element/object the configured properties and values (i.e. as if
they were originally sent in a [Configure message](#syntax_configure))

### Examples

These are some examples of **create** messages and their corresponding 
**inform** reply messages, both in XML and JSON.

**example 1 (XML)**

      <create xmlns="http://schema.mytestbed.net/omf/6.0/protocol" mid="1ab3f0">
        <src>xmpp://node123@domainA.com</src>
        <ts>1360889715</ts>
        <props xmlns:vm="http://foo.com/virtual_machine">
          <type>virtual_machine</type>
          <vm:os>ubuntu</vm:os>
          <vm:osversion>12.04</vm:osversion>
          <vm:ram>
            <value>8</value>
            <unit>GB</unit>
          </vm:ram>
        </props>
      </create>

      <inform xmlns="http://omf.mytestbed.net/omf/6.0/protocol" mid="27ab23">
        <src>xmpp://iperf456@domainB.com</src>
        <ts>1360889720</ts>
        <cid>1ab3f0</cid>
        <it>CREATION.OK</it>
        <props xmlns:vm="http://foo.com/virtual_machine">
          <res_id>xmpp://vm456@domainB.com</res_id>
          <type>virtual_machine</type>
          <vm:os>ubuntu</vm:os>
          <vm:osversion>12.04</vm:osversion>
          <vm:ram>
            <value>8</value>
            <unit>GB</unit>
          </vm:ram>
        </props>
      </inform>

**example 1 (JSON)**

      {
        "op": "request",
        "mid": "1ab3f0",
        "src": "amqp://domainA.com/node123",
        "ts": "1360889715",
        "props": {
          "@context": "http://foo.com/virtual_machine",
          "type": "virtual_machine",
          "os": "ubuntu",
          "osversion": "12.04",
          "ram": { "unit": "gigabyte", "value": 8}
        }
      }

      {
        "op": "inform",
        "mid": "27ab23",
        "src": "amqp://domainA.com/node123",
        "ts": "1360889720",
        "cid": "1ab3f0",
        "it": "CREATION.OK",
        "props": {
          "@context": "http://foo.com/virtual_machine",
          "res_id": "xmpp://vm456@domainB.com",
          "type": "virtual_machine",
          "os": "ubuntu",
          "osversion": "12.04",
          "ram": { "unit": "gigabyte", "value": 8}
        }
      }

**example 2 (XML)**

      <create xmlns="http://schema.mytestbed.net/omf/6.0/protocol" mid="962ac5">
        <src>xmpp://node123@domainA.com</src>
        <ts>1360889765</ts>
        <props xmlns:vm="http://foo.com/wifi_interface">
          <type>WiFiAtherosInterface</type>
          <wifi_interface:std>g</wifi_interface:std>
          <wifi_interface:channel>6</wifi_interface:channel>
          <wifi_interface:mode>adhoc</wifi_interface:mode>
        </props>
      </create>

      <inform xmlns="http://omf.mytestbed.net/omf/6.0/protocol" mid="xd389g">
        <src>xmpp://iperf456@domainB.com</src>
        <ts>1360889795</ts>
        <cid>962ac5</cid>
        <it>CREATION.FAILED</it>
        <reason>Unknown or unsupported resource type</reason>
        <props xmlns:vm="http://foo.com/wifi_interface">
          <type>WiFiAtherosInterface</type>
        </props>
      </inform>

**example 2 (JSON)**

      {
        "op": "create",
        "mid": "962ac5",
        "src": "amqp://domainA.com/node123",
        "ts": "1360889765",
        "props": {
          "@context": "http://foo.com/wifi_interface",
          "type": "WiFiAtherosInterface",
          "std": "g",
          "channel": 6,
          "mode": "adhoc"
        }
      }

      {
        "op": "inform",
        "mid": "xd389g",
        "src": "amqp://domainA.com/node123",
        "ts": "1360889795",
        "cid": "962ac5",
        "it": "CREATION.FAILED",
        "reason": "Unknown or unsupported resource type"
        "props": {
          "@context": "http://foo.com/wifi_interface",
          "type": "WiFiAtherosInterface",
        }
      }

[(jump to top)](#frcp)

----

## <a name="syntax_release"></a>Release Syntax

### Overview

A ***release*** message is published to a topic to ask its subscriber(s) to 
release (i.e. terminate) a given resource child. This message has the following 
XML and JSON syntax:

      <release xmlns="http://schema.mytestbed.net/omf/X.Y/protocol" mid=ID>
        <src>RID</src>
        <ts>TIMESTAMP</ts>
        <props>
          <res_id>CHILDID</res_id>
        </props>
      </release> 


      {
        "op": "release",
        "mid": "ID",
        "src": "RID",
        "ts": "TIMESTAMP",
        "props": {
          "res_id": "CHILDID"
        }
      }

where `CHILDID` is the globally unique ID of the child resource to release.

### Usage

When a FRCP entity P receives a **release** message for one of its child
resource C:

1. P should inform C that it will be imminently released. This gives the
opportunity for C to perform any cleanup tasks if required. Then P should
initiate the termination of C. The mechanism for P to inform C of its release
and for P to terminate C are outside the scope of FRCP
2. once C is terminated, P should remove the C's topic address from the
messaging system. Then P must publish an **inform** message to its own topic
address to convey that information, using the following XML or JSON syntax:

          <inform xmlns="http://schema.mytestbed.net/omf/6.0/protocol" mid=ID>
            <src>RID</src>
            <ts>TIMESTAMP</ts>
            <cid>CID</cid>
            <it>INFOTYPE</it>
            <reason>MOREINFO</reason>
            <props>
              <res_id>CHILDID</res_id>
            </props>
          </inform>

          {
            "op": "inform",
            "mid": "ID",
            "src": "RID",
            "ts": "TIMESTAMP",
            "cid": "CID",
            "it": "CREATION.OK",
            "reason": "MOREINFO",
            "props": {
              "res_id": "CHILDID"
            }
          }

    Where `INFOTYPE` is equal to either `RELEASE.OK` (i.e. the child resource
    was released successfully) or `RELEASE.FAILED` (i.e. the child resource
    cannot be released), and `MOREINFO` is an optional message providing 
    additional information on the release process (e.g. why did it fail?)

3. The release process is recursive! Any child resource of a resource currently
being released must first be all released. Thus when receiving a release
instruction from its parent, a resource must release all of its own child
resources

### Examples

These are some examples of **release** messages and corresponding 
**inform** reply messages, both in XML and JSON.

**XML**

      <release xmlns="http://schema.mytestbed.net/omf/6.0/protocol" mid="fgh432">
        <src>xmpp://node123@domainA.com</src>
        <ts>1360889900</ts>
        <props>
          <res_id>interface789</res_id>
        </props>
      </release>

      <inform xmlns="http://omf.mytestbed.net/omf/6.0/protocol" mid="8714g0">
        <src>xmpp://iperf456@domainB.com</src>
        <ts>1360889908</ts>
        <cid>fgh432</cid>
        <it>RELEASE.OK</it>
        <props>
          <res_id>interface789</res_id>
        </props>
      </inform>

**JSON**

      {
        "op": "release",
        "mid": "fgh432",
        "src": "amqp://domainA.com/node123",
        "ts": "1360889900",
        "props": {
          "res_id": "interface789"
        }
      }

      {
        "op": "inform",
        "mid": "8714g0",
        "src": "amqp://domainA.com/node123",
        "ts": "1360889908",
        "cid": "fgh432",
        "it": "RELEASE.OK",
        "props": {
          "res_id": "interface789"
        }
      }


[(jump to top)](#frcp)

----

## <a name="syntax_guard"></a>Optional Guard Element or Object

### Usage

All the messages described above may also include an optional `guard`
element/object. When present, it should contain a list of key/values similar to
a `props` element/object. When an FRCP resource receives a message with such a
`guard` element/object, it must act on the message **ONLY IF** all of its
properties which are mentioned in the `guard` element/object have the exact
corresponding values. Such an exact match must happen on all properties of type
string, fixnum, boolean, hash, or array. Indeed, if the property 'foo' is of a
type 'array', then all the items in the entity's `foo` property have to match
all the items described in the `foo` key within the guard element/object.
In the future, FRCP may support additional 'mode' attributes to request the 
type of match to perform.

Therefore if many entities are subscribing to a same topic address on the
messaging system, then using a `guard` element/object allow the publisher of a
message to target a specific set of these subscribers.

For example, if a message has a `guard` element containing two properties `p1`
and `p2` with corresponding values `v1` and `v2`, then this message must only be
processed by the receiving subscribers which have their a `p1` and `p2`
properties set to the values `v1` and `v2`, respectively.

### Examples

Here is are XML and JSON configure messages with a `guard` element/object

      <configure xmlns="http://schema.mytestbed.net/omf/6.0/protocol" mid="dj15fa">
        <src>xmpp://node123@domainB.com</src>
        <ts>1360896006</ts>
        <props xmlns:iperf="http://foo.com/iperf">
           <iperf:bitrate>
             <unit>kBps</unit>
             <value>1024</value>
           </iperf:bitrate>
        </properties>
        <guard xmlns:iperf="http://foo.com/iperf">
            <iperf:transport>UDP</iperf:transport>
            <iperf:target>192.168.1.10</iperf:target>
        </guard>
      </configure>

      {
        "op": "configure",
        "mid": "dj15fa",
        "src": "amqp://domainA.com/node123",
        "ts": "1360896006",
        "props": {
          "@context": "http://foo.com/iperf",
          "bitrate": { "unit": "kBps", "value": 1024}
        }
        "guard": {
          "@context": "http://foo.com/iperf",
          "transport": "UDP",
          "target": "192.168.1.10"
        }
      }

  Only the Iperf resources, which have their `transport` and `target` properties
  set to `UDP` and `192.168.1.10`, respectively, will act on this configure
  message, i.e. will configure their bitrate to 1024 kBps. 

[(jump to top)](#frcp)

----

## <a name="core_properties"></a>Core Properties for All Resources

FRCP defines the following small set of core properties that must be
implemented by all resources.

### uid

Its value is the resource's globally unique ID

### name and hrn

Its value is a human friendly/readable name for this resource

### supported_children_type

It is an array of all the type of child resources that this resource can create

### child_resources

It is an array of the IDs of all the child resources which were created by this
resources

### membership

In addition to subscribing to the topic address corresponding to its ID in the
messaging system, a resource may also subscribe to any other topic addresses. 

This allows the efficient delivery of FRCP to a group of resources. Assuming we
have two distinct resource R1 and R2, both are subscribed to their individual
topic addresses T1 and T2. If they are also both subscribed to a third topic G.
Then sending a single message to topic G will allow us to reach both resource
R1 and R2. Thus this could be viewed as R1 and R2 both belonging to a group 
named G.

In FRCP, the **membership** property of a resource is an array that contains
the ID of the topics to which that resource has subscribed to in addition to its
own topic address. Thus in the above example, the **membership** property of R1 would be equal to ["G"].

To tell a resource to subscribe to new group, one should send a **configure**
message to that resource with `membership` property set to the group's topic.
The resource receiving that **configure** message must add the new topic to its
**membership** array property, then it must subscribe to that new topic, and it
must finally return an **inform** message with a `membership` element/object
with list of all of the topics it is subscribed to.

**XML example**

      <configure xmlns="http://schema.mytestbed.net/omf/6.0/protocol" mid="aed351">
        <src>xmpp://node123@domainA.com</src>
        <ts>1394429797</ts>
        <props>
          <membership type="string">xmpp://blue_group@domainC.com</membership>
        </props>
      </configure>

      <inform xmlns="http://schema.mytestbed.net/omf/6.0/protocol" mid="b86017">
        <src>xmpp://node456@domainB.com</src>
        <ts>1394429797</ts>
        <cid>aed351</cid>
        <it>STATUS</it>
        <props>
          <membership type="array">
            <it type="string">xmpp://square_group@domainA.com</it>
            <it type="string">xmpp://heavy_group@domainB.com</it>
            <it type="string">xmpp://blue_group@domainC.com</it>
          </membership>
        </props>
      </inform>

**JSON example**

      {
        "op": "configure",
        "mid": "aed351",
        "src": "amqp://domainA.com/node123",
        "ts": "1394429797",
        "props": {
          "membership": "amqp://domainC.com/blue_group"
        }
      }

      {
        "op": "inform",
        "mid": "b86017",
        "src": "amqp://domainB.com/node456",
        "ts": "1394429797",
        "cid": "aed351",
        "it": "STATUS",
        "props": {
          "membership": [
            "amqp://domainA.com/square_group",
            "amqp://domainB.com/heavy_group",
            "amqp://domainC.com/blue_group"
          ]
        }
      }




