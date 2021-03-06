---
title: Provisioner Agent
markdown2extras: tables, code-friendly
apisections:
---
<!--
    This Source Code Form is subject to the terms of the Mozilla Public
    License, v. 2.0. If a copy of the MPL was not distributed with this
    file, You can obtain one at http://mozilla.org/MPL/2.0/.
-->

<!--
    Copyright (c) 2014, Joyent, Inc.
-->

# Provisioner Agent

The provisioner agent runs on all SDC nodes (headnode and compute nodes)
and is primarly the agent by which CNAPI operates on those nodes via
a number of tasks (documented below). Provisioner chiefly operates over AMQP,
but some effort has been put into making tasks as transport agnostic as
possible.

When the main process starts up, it connects to AMQP. It will then create
queues on which it will receive messages. Each queue can handle one or more
tasks, where type may be something like `machine_create`, `machine_reboot`,
etc. Each queue is given a upper limit of tasks to be concurrently
executed.


# Provisioner Messages

*Note*: ${var} denotes you should substitue that value with something
meaningful.


## Incoming messages

To start a new task send a message to this routing key:

    ${agent}.${node_uuid}.task.${task}

Payload:

    {
        task_id: 'my_unique_task_id',
        client_id:  'my_unique_client_id'
    }

Keys:

`task_id`:

> A unique id to relate this request to any tasks, events and steps.

`client_id`:

> A unique id that will identify the initiator of the task. This is
> used in outgoing messages we wish the sender to get, so they can bind a
> routing key to a queue ahead of time.


## Outgoing

### Steps

Step messages indicate the entry/exit of a task step. If an event's name is
prefixed with start: or end: it means it was a step and the event name after
the colon (:) was the name of the step.

    provisioner.${node_uuid}.event.start:prec_check.${client_id}.${task_id}
    provisioner.${node_uuid}.event.end:prec_check.${client_id}.${task_id}
    provisioner.${node_uuid}.event.start:ensure_dataset_present.${client_id}.${task_id}
    provisioner.${node_uuid}.event.end:ensure_dataset_present.${client_id}.${task_id}


### Events

Events messages indicate may milestone in a task, or that something has
happened. This might be that a certain % progress has been reached, that we
have started or finished a step, or something that doesn't necessarily correlate to the entry or exit of a step.

    provisioner.${node_uuid}.event.screenshot.${client_id}.${task_id}


### Progress

Indicates from 0-100 how far along this task is, with 0 being just started and
100 being finished.

    provisioner.${node_uuid}.event.progress.${client_id}.${task_id}

## Sample Interaction

This is what a request to provision a VM on a compute node might look like.

Tasks begin with a request originating from a "client". Tasks end when the
agent sends a "finish" event.

E.g. for the `machine_create` task, it might look like this:

    --> provisiner.564dba97-54f6-4d3d-50d4-fe51cb228cc8.task.machine_create

    {
        client_id:  '5699633f',
        task_id: '11999575',

        <vm parameters>
    }

Provisioner indicates it has started the task:

    <-- provisioner.564dba97-54f6-4d3d-50d4-fe51cb228cc8.event.start.5699633f.11999575
    {}

Provisioner begins to execute steps and emit progress events:

    <-- provisioner.564dba97-54f6-4d3d-50d4-fe51cb228cc8.event.progress.5699633f.11999575
    { value: 0 }

    <-- provisioner.564dba97-54f6-4d3d-50d4-fe51cb228cc8.event.start:pre_check.5699633f.11999575
    <-- provisioner.564dba97-54f6-4d3d-50d4-fe51cb228cc8.event.end:pre_check.5699633f.11999575

    <-- provisioner.564dba97-54f6-4d3d-50d4-fe51cb228cc8.event.progress.5699633f.11999575
    { value: 20 }

    <-- provisioner.564dba97-54f6-4d3d-50d4-fe51cb228cc8.event.start:ensure_dataset_present.5699633f.11999575
    <-- provisioner.564dba97-54f6-4d3d-50d4-fe51cb228cc8.event.end:ensure_dataset_present.5699633f.11999575

    <-- provisioner.564dba97-54f6-4d3d-50d4-fe51cb228cc8.event.progress.5699633f.11999575
    { value: 30 }

    <-- provisioner.564dba97-54f6-4d3d-50d4-fe51cb228cc8.event.start:fetch_dataset.5699633f.11999575
    <-- provisioner.564dba97-54f6-4d3d-50d4-fe51cb228cc8.event.end:fetch_dataset.5699633f.11999575

    <-- provisioner.564dba97-54f6-4d3d-50d4-fe51cb228cc8.event.progress.5699633f.11999575
    { value: 50 }

    <-- provisioner.564dba97-54f6-4d3d-50d4-fe51cb228cc8.event.start:create_machine.5699633f.11999575
    <-- provisioner.564dba97-54f6-4d3d-50d4-fe51cb228cc8.event.end:create_machine.5699633f.11999575

    <-- provisioner.564dba97-54f6-4d3d-50d4-fe51cb228cc8.event.progress.5699633f.11999575
    { value: 100 }

Finally the `finish` event message is sent.

    <-- provisioner.564dba97-54f6-4d3d-50d4-fe51cb228cc8.event.finish.5699633f.11999575
    {
        <result parameters>
    }

# Tasks

# Machine Tasks

## machine_create
## machine_destroy
## machine_boot
## machine_shutdown
## machine_reboot

## machine_create_image

Called by CNAPI's
[VmImagesCreate](https://mo.joyent.com/docs/cnapi/master/#VmImagesCreate)
to create a new image from a prepared VM and publish it to the local DC's
IMGAPI.

### Inputs

| Field            | Type    | Required? | Description                                                                                                                                                                                                                                                                                                                              |
| ---------------- | ------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| compression      | String  | required  | The "compression" field as required by `imgadm create`. One of "none", "gzip" or "bzip2".                                                                                                                                                                                                                                                |
| uuid             | UUID    | required  | UUID of a prepared and stopped VM from which the image will be created.                                                                                                                                                                                                                                                                  |
| incremental      | Boolean | optional  | Whether to create an incremental image. Default is false.                                                                                                                                                                                                                                                                                |
| manifest         | Object  | required  | Manifest details for the image to be created. Those fields that are required are mentioned in this table. See [the image manifest docs](https://mo.joyent.com/docs/imgapi/master/#image-manifests) for full details. Some fields -- e.g. 'type', 'os' -- are inherited from the origin image (the image used to create the prepared VM). |
| manifest.uuid    | UUID    | required  | A newly generated UUID to be used for the created image.                                                                                                                                                                                                                                                                                 |
| manifest.owner   | UUID    | required  | The UUID of an existing user who will own the image.                                                                                                                                                                                                                                                                                     |
| manifest.name    | String  | required  | The name for the image to be created.                                                                                                                                                                                                                                                                                                    |
| manifest.version | String  | required  | The version for the image to be created.                                                                                                                                                                                                                                                                                                 |
| imgapi_url       | URL     | required  | The URL of the IMGAPI to which the image will be published. Typically this the DC's local IMGAPI at "http://imgapi.$domain"                                                                                                                                                                                                              |



# ZFS Tasks

Tasks for interacting directly with `zfs`.

## zfs_clone_dataset
## zfs_create_dataset
## zfs_destroy_dataset
## zfs_get_properties
## zfs_set_properties
## zfs_list_datasets
## zfs_rename_dataset
## zfs_rollback_dataset
## zfs_snapshot_dataset
## zfs_list_pools


# Metering Tasks

TODO


# Operator Guide

TODO: examples using taskadm, logs location (separated out task logs), etc

