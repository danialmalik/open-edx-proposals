=============================================
OEP-41: Asynchronous Server Event Messaging
=============================================

.. list-table::
   :widths: 25 75

   * - OEP
     - :doc:`OEP-41 </oeps/oep-0041-arch-async-server-event-messaging.rst>`
   * - Title
     - Asynchronous Server Messaging
   * - Last Modified
     - 2020-02-23
   * - Authors
     - David Ormsbee <dave@edx.org>
   * - Arbiter
     - ?
   * - Status
     - Draft
   * - Type
     - Architecture (? Best Practice?)
   * - Created
     - 2020-02-23
   * - Review Period
     - ?
   * - Resolution
     - ?
   * - References
     - ?

--------
Abstract
--------

This OEP describes the conventions Open edX should use for asynchronous event
messaging across services. These events would be emitted when a service has
taken some action to data that it owns, and that other services would be
interested in. This could be a course publish, a user enrollment, updates to
catalog metadata, etc.

This OEP does not cover:

* Remote Prodecure Calls, i.e. using messaging to send commands to other
  services. Events in this document are the services broadcasting, "Here is what
  I just did!" to interested parties, not "Please do this thing for me!"
* Real-time streaming of student learning events to external platforms, as
  detailed in `OEP-26 <oep-0026-realtime-events>`_, though it is possible that
  the implementation of OEP-26 might build on the infrastructure outlined in
  this document.
* Browser level notifications and event streaming along the lines of
  Server-Sent-Events, GraphQL subscriptions, or frameworks built on top of
  WebSocket. The interactions here are strictly server to server interactions
  between services serving the same site. Again, it is entirely possible that
  an implementation of browser level notifications would be built on top of the
  infrastructure outlined here.


----------
Motivation
----------

Open edX has multiple services that need to copy data or take specific actions
when certain events occur in other services. For example:

* The start date of a course exists in both the Catalog domain represented by
  the course-discovery service, as well as in the LMS, where its value is used
  to help determine whether an enrolled student will have access to the content.
* Other services often have a user table, with data replicated from the LMS
  (the place where users register today).
* Publishing a new version of course content in Studio triggers a variety of
  LMS tasks to query and re-build optimized versions of course data for various
  systems like the Block Transformers that power the mobile application and
  course outline.
* Grading events in the LMS may trigger certificate generation if the user has
  exceeded the required threshold.

Open edX services already use `Django signals
<https://docs.djangoproject.com/en/1.11/topics/signals/>`_ for subscribing to
events within the same service, but we have no accepted recommendation for how
to do something similar across service boundaries. As the number of services in
Open edX has grown, the various ad hoc synchronization and communication methods
we've implemented over the years have caused numerous operational and
performance problems (particularly around catalog data). At the same time,
recent discussions around Domain Driven Design and data replication have given
us a better framing for the kind of relationship we want to have between various
services. There has also been a strong desire for a properly documented set of
integration APIs that would allow Open edX to better interoperate with other
applications in the ecosystem.

There is an opportunity to bring all these elements together and create a solid
foundation for simple, reliable, well documented, near real-time cross-service
message passing. But without some planning and design work, we may end up in an
even more chaotic place where each pairing of services uses different
conventions for this kind of communication.


-------------
Specification
-------------

Message Format
==============

The message format will use the `CloudEvents 1.0
<https://github.com/cloudevents/spec/blob/master/spec.md>`_ specification, with
additional usage conventions that all Open edX services should adhere to.
CloudEvents was developed by the Cloud Native Computing Foundation, with
participation from major companies like Google, Microsoft, IBM, Red Hat, Oracle,
and Huawei. While the finalized 1.0 spec is only months old and the Python
tooling is primitive (advertised to break at any time), CloudEvents is also a
pretty minimal message envelope spec, with the basic required fields being
things that we'd want to have anyway.

Event messages will be UTF-8 encoded JSON documents of 64 KB or less. An
example:

.. code-block:: javascript

  {
    "id": "b3b3981e-7544-11ea-b663-acde48001122",
    "source": "https://discovery.edx.org/events/"
    "specversion": "1.0",
    "type": "org.openedx.catalog.course.created.v1"
    "time": "2020-02-23T09:43:18Z",
    "minorversion": 0,
    "data": {
        // message specific payload here
    }
  }

CloudEvents Fields
==================

All top level field names are specified by the CloudEvents spec, but most fields
defined by that standard have some leeway in how they're used for any given
system. Each field below has a link to CloudEvents requirements, as well as a
description of the conventions Open edX will use for that field. All Open edX
events should have all of the fields specified below (even if some are
considered optional in the CloudEvents spec).

`id <https://github.com/cloudevents/spec/blob/master/spec.md#id>`_
------------------------------------------------------------------
Example: ``"b3b3981e-7544-11ea-b663-acde48001122"``

Message ID. CloudEvents requires this to be a string that is unique for any
given source. We will use a UUID1 here, since it is easy to generate without
worrying about collisions, and it offers slightly more useful introspection
(timestamp and machine) than a random UUID4. The UUID will be encoded as a
string using Python's default behavior: lowercase and and dash-separated.

`source <https://github.com/cloudevents/spec/blob/master/spec.md#source-1>`_
-----------------------------------------------------------------------------
Example: ``"https://discovery.edx.org/events/"``

This is an absolute URI that represents the URL of the server emitting the
event. The specific URL path may give some useful information about the events
being emitted. Groups of servers (e.g. behind a load balancer) should choose the
same URI, and not have an individual one per server. URIs should be intelligible
to humans, and not something machine generated like AWS machine names.

Message clients should *not* interpret this value and make switching logic based
on where a message is coming from. The service producing the event should be
free to change this URI at any time.


`specversion <https://github.com/cloudevents/spec/blob/master/spec.md#specversion>`_
------------------------------------------------------------------------------------
Always: ``"1.0"``

Mandatory field that refers to the version of CloudEvents. We have to use "1.0"
to be spec-compliant.

`type <https://github.com/cloudevents/spec/blob/master/spec.md#type>`_
----------------------------------------------------------------------
Example: ``"org.openedx.catalog.course.created.v1"``

This is the name of our event. CloudEvents only requires that this be a string
describing the type of event, but recommends that it be prefixed with a
reverse-DNS name for namespacing purposes. We will use a period-separated
hierarchical name comprising with the format ``{Reverse DNS}-{Architecture
Domain}-{Subject}-{Action}-{Major Version}``.

Reverse DNS
~~~~~~~~~~~
Example: ``org.openedx``

In this example, ``org.openedx`` is the generic prefix that would be used for
all Open edX installs. Events that are strictly edX-specific, like those that
might interact with internal IT or finance reporting systems, should use
``org.edx`` as the prefix instead. If in doubt, default to ``org.openedx``.

Domain (from Domain Driven Design)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Example: ``catalog``

The next part of the ``type`` hierarchy is the Domain. Examples of this are:

* catalog
* content_authoring
* credentials
* ecommerce
* learning

It is the expectation that there are relatively few domains, and that they will
roughly match deployed services. Domain names should be lower cased and use
underscores if they are more than one word.

Subject
~~~~~~~
Example: ``course``

The name of an entity that the event applies to. Examples might be ``course``,
``student``, ``enrollment``, ``order``, etc. Subjects may be namespaced, so
``special_exam.proctored.allowance`` could be a subject.

A subject should always mean the same thing within a Domain, but can mean
different things across domains. For instance, what the LMS (``learning``
domain) calls a ``course`` might map to what the ``catalog`` domain would call a
``course_run``. We should try to be consistent where possible, but each domain
ultimately gets to decide what its terms mean, and we should be careful when
translating a concept from one domain to another. For instance, the
``content_authoring`` and ``learning`` domains might both have a concept of a
"due date" for an assignment. But while the ``content_authoring`` due date is
determined only by the content author, the ``learning`` due date might take into
account a student's cohort, individual due date extensions, accessibility
allowances, and any number of other things. Both domains may call it ``due``,
but the due date information from ``content_authoring`` is just an input to the
more complex due date information in ``learning``.

Action
~~~~~~
Example: ``created``

This is the action that occurred for the event. Some of most common ones will be
``created``, ``updated``, and ``deleted``, but many applications will want more
specific actions like ``declined``, ``started``, ``verified``, etc. Applications
should prefer these more specific actions when possible, since they are usually
clearer for listeners and don't requiring knowing implementation details. For
instance, it is preferable to have a ``registration.completed`` event than to
have a ``registration.updated`` event and leave it to clients to check the
resulting message's payload for ``status: "complete"``.

Actions should be past tense, to better align with our existing conventions
around Django signals and learning analytics events (we're not completely
consistent, but we tend towards past tense).

Major Version
~~~~~~~~~~~~~
Example: ``v1``

The last portion is the major version of the message, starting with ``v1``. All
messages will have a major and minor version, with the minor version encoded in
the ``data`` attribute (todo: link to where we discuss that). New fields may be
added to a message without incrementing the major version, but all additions
*must* be backwards compatible. (Todo: link to more info on backwards
compatibility.)

`datacontenttype <https://github.com/cloudevents/spec/blob/master/spec.md#datacontenttype>`_
--------------------------------------------------------------------------------------------
Example: ``"application/json"``

`RFC 2046 <https://tools.ietf.org/html/rfc2046>`_ string value describing the
format of ``data`` attribute. It should be ``application/json`` the vast
majority of the time, but some events might have ``text/xml``.

`time <https://github.com/cloudevents/spec/blob/master/spec.md#time>`_
----------------------------------------------------------------------
Example: ``"2020-02-23T09:00:00Z"``

Timestamp that the event occurred, in UTC using `RFC 3339
<https://tools.ietf.org/html/rfc3339>`_. If this event was sent because we
created a new row in the database, we should pull this ``time`` directly from
the ``created_at`` field in that model so that the output matches exactly. Do
*not* call ``datetime.now()`` in these situations because we will get times
that are just a few milliseconds offset from the database record of these
actions. Do call ``datetime.now()`` if the event happens and has no
corresponding database changes. If you are sending out multiple event messages
describing the same occurance (e.g. a version 1 and version 2 of an event), they
should have the *exact* same timestamp.

Extension Fields
================

CloudEvents allows for extension fields to be declared, so long as they do not
conflict with existing fields. We are adding a couple that will be required
across all Open edX generated events

pii
---
Example: ``false``

Boolean field to indicate whether this message contains any personally
identifiable information. It is the responsiblity of the receiver to examine
this field and any documentation to understand what specific fields are PII and
how they need to be stored and tracked for possible later deletion.

minorversion
------------
Example: ``2``

This is a minor version in semver reckoning, meaning that it increments when
we have made backwards compatible additions to the message payload in the
``data`` attribute. Values can only be integers. The initial value for
``minorversion`` should be ``0``. There is no corresponding ``majorversion``
because that information is encoded into the message type.


Message Content Data Guidelines
===============================

These are general guidelines to consider when creating your messages. There can
always be exceptional circumstances and use cases that require going against one
of these guidelines, but try to default to these guidelines unless you're really
sure about what you're doing.

One Producer Service Per Event Type
-----------------------------------

Each event type should be emitted by one, and only one, service. That service is
the source of truth for whatever entity the event describes. If course-discovery
emits an event describing when a course starts (e.g.
``org.openedx.catalog.course.start_date.changed``), it is *not* appropriate for
the LMS to send potentially conflicting information using that same event type.

Two services may have similar sounding events. The course-discovery service
(``catalog`` domain) might emit a ``org.openedx.catalog.course.created.v1``
event when a catalog entry for a course is created, while Studio
(``content_authoring`` domain) might emit a
``org.openedx.content_authoring.course.created.v1`` event when course content is
first authored there. These are similar, related events, but they are not the
same event type.

Events are Created by the Owning Domain
---------------------------------------

Teams at edX are broadly aligned to domains and roughly mapped to services.
Services should not emit events for other domains. For instance, the ecommerce
service is its own domain and should not be emitting ``catalog`` or ``learning``
events.

Encoding Opaque Keys
--------------------

The opaque-keys library is used by many parts of Open edX to identify pieces of
content. Whenever these things are passed in messages, use the convention of
ending the variable name with "_key", so ``course_key`` instead of ``course_id``
and ``usage_key`` instead of ``usage_id``.

Keep Messages Self-Contained and Avoid Callbacks
------------------------------------------------

Whenever possible, messages should be self-contained and not force the receiver
to either keep a full history of the entity or to hit a callback for more
information. For instance, if the event relates to an update to the record, it
is reasonable to include the entire record, and not just the one field that
changed. This has to be weighed against size restrictions of the message.

A callback is when you create a message containing an ID or URL that you expect
the consumer to make a synchronous call to when it receives the message. This is
commonly used when an event represents some change that is too large to
practically fit into the 64K message. For instance, we currently emit a generic
``course_published`` Django signal whenever data is published in Studio, leading
to a cascade of calls from various apps to the ModuleStore in order to extract
the content data that they need.

Callbacks threaten performance and stability because they reduce a service's
ability to control its own load. For instance, a sudden increase in Courseware
traffic might generate a burst of student analytics events. If this stream of
events overwhelms your service's ability to consume them, the queue may start to
back up with unread events. Yet this shouldn't cause your service to fail, since
it still gets to control how quicky it consumes events off of that queue. It has
the freedom to either slowly catch up (if the burst was a momentary spike), or
to scale up additional workers to handle the higher throughput. Your service's
decision to scale up or down does not directly impact other services.

Things change when we introduce a callback to this same scenario. Say the
analytics events now include a callback URL to get basic user information. In
this scenario, doubling the consumers that your service has now also doubles the
load that your service is placing on the REST endpoint serving this user
information. There's no way for you as a service owner to know for sure if
that's safe, and doing so may lead to cascading failures in other systems.

One thing to consider is whether we can emit multiple events that better target
specific consumer use cases. Let's take the ``course_published`` event as an
example. Some listeners only care about schedule changes, because they have to
account for when a course starts and ends. Search indexing really only wants to
know the exact bit of content that was modified so that it can update that text.
There is no rule that says a single user action has to translate into a single
event. Be mindful of what your consumers actually care about and the broad use
cases you're trying to serve.

If a callback is still necessary, try to make sure that it points to an
especially robust and performant endpoint. For instance, an event that is fired
when a user changes their profile information might include a URL to the S3
location of their new profile picture. Just keep in mind that messages may be
read long after they're generated, and any presigned S3 URLs you generate might
be expired by the time a consumer gets them.

Transport Layer
===============

Events will be sent and received using Kombu, the messaging library that celery
uses. We will use Redis as the backing server. This allow us to implement this
messaging scheme without adding any new server dependencies.

Concept Mapping
---------------

Todo: Need to actually code up a proof-of-concept (base off the experimental
course-discovery PR?) to test that this works properly...

Every domain will map to a Kombu Topic Exchange, and services will be Producers
that send messages to those Exchanges. The Exchange name and message Routing Key
will be derived from the event ``type``. The parts of the ``type`` up to and
including the Domain will be the name of the exchange. The rest of the ``type``,
starting from the Subject will be the Routing Key. So the
``org.openedx.catalog.course.created.v1`` event would go to the
``org.openedx.catalog`` Exchange, and use ``course.created.v1`` for the Routing
Key.

It is then possible for consumers to declare a Queue with multiple bindings that
subscribe to various families of events coming from potentially multiple
exchanges::

    from kombu import Exchange, Queue, binding

    catalog_exchange = Exchange('org.openedx.catalog', type='topic')
    ecommerce_exchange = Exchange('org.openedx.ecommerce', type='topic')

    queue = Queue(
        bindings=[
            binding(catalog_exchange, routing_key='program.*'),
            binding(catalog_exchange, routing_key='course.*'),
            binding(ecommerce_exchange, routing_key='*.purchased.*'),
        ]
    )

Todo: Contrast with Kafka.

.. warning::
    Everything below this area are still in scratch-notes form. So please don't
    review.

Code Practices
==============

Defining Message Classes
------------------------

When to Emit Messages
---------------------
(Transactions, on_commit, etc.)

Handling Versions
-----------------

Bootstrapping
-------------
Equivalent of simulate_publish?

Testing
-------

Documentation
-------------
AsyncAPI?

Management/Introspection
------------------------

Logging
-------

Architectural Implications
==========================

Improve Data Replication and Resiliency
---------------------------------------

Reduce the need for Plugins
---------------------------

Simplify Integration with Other Systems
---------------------------------------
(Richie, third party analytics, etc.)

