---
layout: post
title:  "Google Summer of Code 2021 with Ceph"
date:   2021-08-13 23:00:00 +0800
categories: re
---
During the summer, I have participated in Google Summer of Code 2021 with [Ceph](https://ceph.io/en/).
My work is focused on adding [NATS](https://nats.io/) messaging system to the current list of Ceph's
bucket notification endpoints.

# Background
----
Bucket notifications are a mechanism to send notifications when certain events are
happening on the bucket to external systems. Bucket notifications could be used in various
implementations:

[Analyzing X-rays based on Ceph and Knative](https://knative.dev/docs/) - Creating a fully-automated data
pipeline using bucket notifications from Ceph, along with Kafka and KNative Eventing and
Serving to save X-ray images after assesing the risk of pnumonia and anonymizing the
image for research usage.

The pipeline works as follows, when image is pushed to a certain bucket, Ceph will send a notification to Kafka.

Kafka will handle the notification and KNative Eventing will fetch the notification
message, pssing it to the Serving Service.

The service will launch an image-proccessing container which will assess pnumonia risk.

Eventually KNative service will anonymize the image and save it to another bucket.

{% include image name="graph.png" caption="Automated Data Pipeline" %}

## More usecase examples:
* [Data Pipelines based on diffrent storage solutions](https://www.youtube.com/watch?v=ZK510prml8o&ab_channel=CNCF%5BCloudNativeComputingFoundation%5D)

* [Pulling notifications from CEDA](https://fosdem.org/2021/schedule/event/sds_ceph_rgw_serverless/) 

# Goal
----
Before this project, Ceph supported sending notifications only via Kafka and AMQP0.9.1
message brokers and HTTP endpoint.

The goal of the project was to add NATS to our supported brokers. NATS is a highly
performant cloud native messaging system designed for [Kubernetes](https://kubernetes.io/). 
Since Ceph can run inside Kubernetes, integration with cloud native messaging systems is
very appealing.

However, unlike Kafka and AMQP, NATS is a young project, therfore we wanted to avoid adding
a NATS client library to our source code.

The approach was to integrate our [Lua Scripting feature](https://docs.ceph.com/en/latest/radosgw/lua-scripting/) together with a [NATS Lua
client](https://github.com/DawnAngel/lua-nats).

# Code Merged
----
## [Lua Package Version Support](https://github.com/ceph/ceph/pull/41927)

In order to make the integration, certain lua packages were needed. An issue found with the
Lua scripting feature was that there was no support available for package versions.

Meaning, it wasn't possible to install or remove specfic versions of a lua package.


## [Missing Notification Field](https://github.com/ceph/ceph/pull/42102)

A bucket notification includes many fields of information about the event. One of these
fields ("User Identity") wasn't available to Lua.

After exposing this field and updating the documentation, a test was added aswell to make
sure everything works correctly.

## [Full Integration](https://github.com/ceph/ceph/pull/42169)

Sending bucket notifications via NATS server is supported using the Lua script added to
the repo with a documantation explaining how to upload and use the script with Ceph.

Thanks to the Lua scripting feature and the script as an example, any message broker could
now be supported and used with Ceph without changing the source code of Ceph.(!)

## [Fixing Installing Function](https://github.com/ceph/ceph/pull/42739)

In one of the commits aiming to reduce compilation time a bug was found.

An incorrect return value checked caused undesirable behaviour.

#  Code In-Progress
----
## [Global Data Table](https://github.com/ceph/ceph/pull/42779)


Allowing the user to collect data in a Lua script and use it in other executaions opens
a door for various usecases. This can be achieved using Lua's compatibility with c++,
  A lua table will be saved in the RGW as a map. This map will be translated to a Lua
  table before executing each script.

# To-Do
----
## Caching The Connection

Each execution of a Lua script creats a new connection to the NATS server in order to send
the notification. By saving the connection beyond the lifespan of the script we can avoid unnecessary connection creations and closures.

There were several approaches to save the connection beyond the lifespan of the script.
The first approach was to create the connection in a global lua state and each request will have access to this state (and to the connection
as well). 

Another soloution was to use the open connection socket file descriptor. By saving the file descriptor in a variable outside the script(in the cpp part),
and setting it as zero inside the NATS connection object.
This way we could avoid prevent the garbage collector to close the socket. Finally, setting the file
descriptor of the new socket the the old one (where the connection was established).

Currently, The Lua-Nats client doesn't support this solution. 

----
## References

- [^1]: [Lua Package Version Support](https://github.com/ceph/ceph/pull/41927)

- [^2]:[Missing Notification Field](https://github.com/ceph/ceph/pull/42102)

- [^3]: [Full Integration](https://github.com/ceph/ceph/pull/42169)

- [^4]: [Fixing Installing Function](https://github.com/ceph/ceph/pull/42739)

- [^5]: [Global Data Table]((https://github.com/ceph/ceph/pull/42779)

- [GSoC Project Link](https://summerofcode.withgoogle.com/projects/#4847082420043776)
