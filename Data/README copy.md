# Data Engineer Exercise

## Overview

This repo contains a sample of anonymised production data that we use to
calculate one of our core metrics. This core metric is how many devices are
online in a given period of time. Your goal is to analyse the same data and
come up with a handful of useful metrics and projections.

## Data Schema

Before diving into the tasks we need to talk about the data model. The relevant
entities are **devices**, **users**, and **servers**. Every device belongs to a
particular user and at any given time it is either disconnected or connected to
a particular server.

The basis of this exercise is a simple event stream of device connectivity
events. This means that when a device connects to one of our servers we record
an "online" event. When a device disconnects from one of our servers we record
an "offline" event. Every event contains a timestamp, the id of the device, and
whether the device connected or disconnected. The event also contains the id of
the owner of this device, and the id of the server it connected to.

**But be careful!** There are missing events from the stream. For example you
might find an online event for a device that is followed by another online
event, with no offline event in between! Or you might find that an offline
event is followed by another offline event, without an online event in between.
This can happen if, for example, a server crashes and doesn't get a chance to
report a device as offline. Or an online event could be missed during a
database outage.

To help reason about this partially corrupted event stream there is an
accompanying dataset of all the servers that have ever existed together with
the timestamp they were created and destroyed. So, for example, if you have an
online event for a particular server but you see no offline event, then you
could assume that the device was in fact online until that server got
destroyed.

The specific rules we have selected to deal with problematic sections
of a device's timeline can be summarised in the following table:

| current event | current server | next event | next server | rule        |
|---------------|----------------|------------|-------------|-------------|
| online        | X              | online     | X           | Assume device was online from current event's timestamp until next event's timestamp
| online        | X              | online     | Y           | Assume device was online from current event's timestamp until X's destruction time or next event's timestamp, whichever is smaller
| online        | X              | offline    | X           | Normal case
| online        | X              | offline    | Y           | Assume device was online from current event's timestamp until X's destruction time. Ignore next event.
| offline       | X              | online     | X           | Normal case
| offline       | X              | online     | Y           | Normal case
| offline       | X              | offline    | X           | Ignore next event
| offline       | X              | offline    | Y           | Ignore next event


### Schema for `connectivity_events.csv`

This dataset is sorted by timestamp

| field     | type     |
|-----------|----------|
| timestamp | datetime |
| device_id | integer  |
| user_id   | integer  |
| server_id | integer  |
| connected | boolean  |

### Schema for `servers.csv`

This dataset is sorted by id

| field        | type     |
|--------------|----------|
| id           | integer  |
| created_at   | datetime |
| destroyed_at | datetime |

## Task 1 - Creating a clean view

Your first task is to create a stacked chart of **unique online devices per
day**, segregated by fleet size. The fleet size is an attribute of each user
and is defined as the number of online devices that this user had at a
particular day. You can split the dataset in the following fleet sizes:

* 1-2 devices
* 3-9 devices
* 10-99 devices
* 100-999 devices

A device should be counted as online for a particular day if it was online for
any amount of time during that day. For example, a device that appear online
for only a second should still be counted for that day.

Here is a sneak peak of what this looks like:

<img src="https://i.imgur.com/DTrjRir.png" />

### Bonus

Calculate the same chart but for every day calculate the **unique online
devices in the past 28 days**, making it a running total.
