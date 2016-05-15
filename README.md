# Snap Collector Plugin — Apache Mesos

[![Build Status](https://travis-ci.com/intelsdi-x/snap-plugin-collector-mesos.svg?token=mxqCYyjxtayP5XBp4JEu&branch=master)](https://travis-ci.com/intelsdi-x/snap-plugin-collector-mesos)

This Snap plugin collects metrics from an [Apache Mesos][mesos-home] cluster.
It gathers information about cluster resource allocation and utilization, as
well as metrics about running containers.

1. [Getting Started](#getting-started)
    * [System Requirements](#system-requirements)
    * [Installation](#installation)
    * [Configuration and Usage](#configuration-and-usage)
2. [Documentation](#documentation)
    * [Collected Metrics](#collected-metrics)
    * [Examples](#examples)
    * [Known Issues and Caveats](#known-issues-and-caveats)
    * [Roadmap](#roadmap)
3. [Community Support](#community-support)
4. [Contributing](#contributing)
5. [License](#license)
6. [Acknowledgements](#acknowledgements)

## Getting Started
### System Requirements
At a minimum, you'll need:
  * [Apache Mesos][mesos-home] (currently tested against 0.26.x, 0.27.x, and 0.28.x)
  * [Golang 1.5+][golang-dl] (only needed for building the plugin)
  * [Snap][snap-github]
  * Linux (amd64)
  * Mac OS X (x86_64) (for development/testing only)

To enable metrics collection from the `perf_event` cgroups subsystem, you'll also need a kernel that supports it,
and the userland tools that enable you to run the `perf` command. On Ubuntu, this includes the following packages:
  * `linux-tools-common`
  * `linux-tools-generic`
  * `linux-tools-$(uname -r)`

*Note: see the [Caveats](#caveats) section below for known issues with Mesos and the `cgroups/perf_event` isolator.*

### Installation
Mesos installation is outside the scope of this README. There are a few resources you might want to consider taking
a look at to get started with Mesos:
  * [Apache Mesos "Getting Started" documentation][mesos-getting-started]
  * [Mesosphere Downloads][mesosphere-downloads]
  * [scripts/provision-travis.sh](scripts/provision-travis.sh)

### Configuration and Usage

## Documentation
Design documents and RFCs pertaining to this plugin are tracked via the ["RFC" label in GitHub Issues][github-rfc].

### Collected Metrics
This plugin collects hundreds of metrics from Mesos masters and agents. As such, there are too many to list them all
here, so instead we've provided a quick overview. To get a complete list of available metrics, you can run the
following commands:
```
$ snapctl plugin load snap-plugin-collector-mesos
$ snapctl metric list
```

#### Mesos master/agent metrics
This plugin returns all available metrics from the `/metrics/snapshot` API endpoint on Mesos masters and agents.
A few of the available metrics that are collected include:

Masters:
  * `master/cpus_total` and `master/cpus_used`
  * `master/disk_total` and `master/disk_used`
  * `master/mem_total` and `master/mem_used`
  * `master/slaves_active`
  * `master/tasks_running`
  * `registrar/state_store_ms/p90`, `registrar/state_store_ms/min`, `registrar/state_store_ms/max`
  * `system/load_1min`, `system/load_5min`, `system/load_15min`

Agents:
  * `slave/tasks_running` and `slave/tasks_failed`
  * `slave/cpus_total` and `slave/cpus_used`
  * `slave/disk_total` and `slave/disk_used`
  * `slave/mem_total` and `slave/mem_used`
  * `slave/executors_running`
  * `system/load_1min`, `system/load_5min`, `system/load_15min`

For a complete reference, please consult the [official Mesos documentation][mesos-monitoring].

#### Mesos monitoring statistics (executor/container metrics)
This plugin returns most of the available metrics from the `/monitor/statistics` API endpoint on the Mesos agent,
which includes metrics about running executors (containers) on a specific Mesos agent. The metrics available via this
endpoint largely depend on the features enabled in Mesos. For example: if Mesos is compiled with the
`--with-network-isolator` option, you'll be able to view per-container network statistics. Furthermore, if you have
the necessary packages installed, you can enable the `cgroups/perf_event` isolator to gather per-container perf
statistics.

### Examples
There are examples of the Snap global configuration and various tasks located in the [examples/](examples) directory.
To get started with these examples and collect Mesos metrics and publish them to a file, you'll need to perform the
following steps.

*Note: these steps will work with the Vagrant development environment included in this repo. For more info on how
to get started with Vagrant, please see [CONTRIBUTING.md](CONTRIBUTING.md).*

Start the Snap daemon in the background:

```
$ snapd --plugin-trust 0 --log-level 1 --config examples/configs/snap-config-example.json \
    > /tmp/snap.log 2>&1 &
```

Assuming you're in the working directory for this plugin, load the Mesos collector plugin:

```
$ snapctl plugin load build/rootfs/snap-plugin-collector-mesos
```

Get the available metrics for your system:

```
$ snapctl metric list
```

Load the `passthru` processor plugin, and the `file` publisher plugin:

```
$ snapctl plugin load ${SNAP_PATH}/plugin/snap-processor-passthru
$ snapctl plugin load ${SNAP_PATH}/plugin/snap-publisher-file
```

Create a new Snap task:

```
$ snapctl task create -t examples/tasks/mesos-all-file.json
```

Stop the task:

```
$ snapctl task stop <task ID>
```

### Known Issues and Caveats
  * Snap's metric catalog is populated only once, when the Mesos collector plugin is loaded. A configuration change on
  the master or agent could alter the metrics reported by Mesos. Therefore, if you modify the configuration of a Mesos
  master or agent, you should reload this Snap plugin at the same time.
  * Due to a bug in Mesos, the parsing logic for the `perf` command was incorrect on certain platforms and kernels. When
  the `cgroups/perf_event` isolator was enabled on an agent, the `perf` object would appear in the JSON returned by the
  agent's `/monitor/statistics` endpoint, but it would contain no data. This issue was resolved in Mesos 0.29.0, and was
  backported to Mesos 0.28.2, 0.27.3, and 0.26.2. For more information, see [MESOS-4705][mesos-4705-jira].
  * There is an ongoing effort to rename the Mesos "slave" service to "agent". As of Mesos 0.28.x, this work is still
  in progress. This plugin uses the newer "agent" terminology, but some metrics returned by Mesos may still use the
  older "slave" term. For more information, see [MESOS-1478][mesos-1478-jira].

### Roadmap
For version 2, we intend to support additional deployment options as documented in issue #14. Otherwise, there isn't
a formal roadmap for this plugin, but it's in active development. If you have a feature request, please
[open a new issue on GitHub][github-new-issue] or [submit a pull request][github-new-pull-request].

## Community Support
This repository is one of many plugins in Snap, a powerful telemetry framework. To reach out to other users in the
community, check out the full project at <http://github.com/intelsdi-x/snap>.

## Contributing
We love contributions!

There's more than one way to give back, from examples to blog posts to code updates. See our recommended process in
[CONTRIBUTING.md](CONTRIBUTING.md).

## License
[Snap][snap-github], along with this plugin, is open source software released
under the [Apache Software License, version 2.0](LICENSE).

## Acknowledgements
  * Authors: [Marcin Krolik][marcin-github], [Roger Ignazio][roger-github]


[github-new-issue]: https://github.com/intelsdi-x/snap-plugin-collector-mesos/issues/new
[github-new-pull-request]: https://github.com/intelsdi-x/snap-plugin-collector-mesos/pulls
[github-rfc]: https://github.com/intelsdi-x/snap-plugin-collector-mesos/issues?utf8=✓&q=is%3Aissue+is%3Aall+label%3ARFC+
[golang-dl]: https://golang.org/dl/
[marcin-github]: https://github.com/marcin-krolik
[mesos-1478-jira]: https://issues.apache.org/jira/browse/MESOS-1478
[mesos-4705-jira]: https://issues.apache.org/jira/browse/MESOS-4705
[mesos-home]: http://mesos.apache.org
[mesos-getting-started]: http://mesos.apache.org/gettingstarted/
[mesos-monitoring]: http://mesos.apache.org/documentation/latest/monitoring/
[mesosphere-downloads]: https://mesosphere.com/downloads/
[roger-github]: https://github.com/rji
[snap-github]: http://github.com/intelsdi-x/snap
