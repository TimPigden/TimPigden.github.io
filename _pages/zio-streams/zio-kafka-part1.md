---
layout: default
title: Streaming Data from Kafka - Part1
description: Illustrates use of Kafka from zio-kafka
---
# Streaming with ZIO and Kafka

# Setup

IMPORTANT: This is the merely the procedure I followed on Ubuntu 19.10. YMMV. I did not have Kafka
in my environment. Throughout, I'm using resources from Confluent

## Initial Setup

1. Install Docker (as of date of this blog, (4th November 2019) it's not yet
properly configured for Ubuntu 19.10, so you may need to fiddle around a
bit to get it working.

2. Follow the instructions for [Confluent Platform Quick Start (Docker)](https://docs.confluent.io/current/quickstart/ce-docker-quickstart.html) and test through to (but not including "Stop Docker").

3. Install the [Confluent CLI](https://docs.confluent.io/current/cli/installing.html)
4. Add CONFLUENT_HOME environment variable.

## Creating Simple Data
I used [Easy Ways to Generate Test Data in Kafka](https://www.confluent.io/blog/easy-ways-generate-test-data-kafka)

First we create topic2 using datagen. I used the sample datagen-users.json file.
```
{
  "name": "datagen-users",
  "config": {
    "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
    "kafka.topic": "topic2",
    "quickstart": "users",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",
    "max.interval": 1000,
    "iterations": 10000000,
    "tasks.max": "1"
  }
}
```
which I put in the streams/assets folder of the source for the project.


```
confluent local config datagen-users -- -d ./datagen-users.json
```