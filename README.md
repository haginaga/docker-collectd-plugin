# docker-collectd-plugin

A [Docker](http://docker.io) plugin for [collectd](http://collectd.org)
using [docker-py](https://github.com/docker/docker-py) and collectd's
[Python plugin](http://collectd.org/documentation/manpages/collectd-python.5.shtml).

This uses the new stats API [introduced by
\#9984](https://github.com/docker/docker/pull/9984) in Docker 1.5.

The following container stats are gathered for each container, with
examples of some of the metric names reported:

* Network bandwidth:
 * `network.usage.rx_bytes`
 * `network.usage.tx_bytes`
 * ...
* Memory usage:
 * `memory.usage.total`
 * `memory.usage.limit`
 * ...
* CPU usage:
 * `cpu.usage.total`
 * `cpu.usage.system`
 * ...
* Block IO:
 * `blockio.io_service_bytes_recursive-<major>-<minor>.read`
 * `blockio.io_service_bytes_recursive-<major>-<minor>.write`
 * ...

All metrics are reported with a `plugin:docker` dimension, and the name
of the container is used for the `plugin_instance` dimension.

## Install

1. Checkout this repository somewhere on your system accessible by
   collectd; for example as
   `/usr/share/collectd/docker-collectd-plugin`.
1. Install the Python requirements with `sudo pip install -r
   requirements.txt`.
1. Configure the plugin (see below).
1. Restart collectd.

## Configuration

Add the following to your collectd config:

```apache
TypesDB "/usr/share/collectd/docker-collectd-plugin/dockerplugin.db"
LoadPlugin python

<Plugin python>
  ModulePath "/usr/share/collectd/docker-collectd-plugin"
  Import "dockerplugin"

  <Module dockerplugin>
    BaseURL "unix://var/run/docker.sock"
    Timeout 3
  </Module>
</Plugin>
```
### Optional configuration flags

The boolean configuration options CpuQuotaPercent and CpuSharesPercent turn on metrics for CPU quota and CPU shares. Both options are set to False by default.

The boolean option `CollectNetworkStats` controls whether container network
stats are collected -- it defaults to true.  Some networking backends don't
report network statistics (e.g. when running on Kubernetes) so it can be useful
to disable to avoid error messages.

```apache
TypesDB "/usr/share/collectd/docker-collectd-plugin/dockerplugin.db"
LoadPlugin python

<Plugin python>
  ModulePath "/usr/share/collectd/docker-collectd-plugin"
  Import "dockerplugin"

  <Module dockerplugin>
    BaseURL "unix://var/run/docker.sock"
    Timeout 3
    CpuQuotaPercent true
    CpuSharesPercent true
  </Module>
</Plugin>
```

### Extracting additional dimensions

The plugin has the ability to extract additional information about or
from the monitored container and report them as extra dimensions on your
metrics. The general syntax is via the `Dimension` directive in the
`Module` section:

```apache
<Module dockerplugin>
  ...
  Dimension "<name>" "<spec>"
</Module>
```

Where `<name>` is the name of the dimension you want to add, and
`<spec>` describes how the value for this dimension should be extracted
by the plugin. You can extract:

- manually specified dimension values, with `raw:foobar`;
- container environment variable values, with `env:ENV_VAR_NAME`;
- any container detail value from Docker, with `inspect:Path.In.Json`.

Here are some examples:

```apache
Dimension "mesos_task_id" "env:MESOS_TASK_ID"
Dimension "image" "inspect:Config.Image"
Dimension "foo" "raw:bar"
```

For `inspect`, the parameter is expected to be a JSONPath matcher giving
the path to the value you're interested in within the JSON output of
`docker inspect` (or `/containers/<container>/json` via the remote API).
See https://github.com/kennknowles/python-jsonpath-rw for more details
about the JSONPath syntax.

### Important note about additional dimensions

The additional dimensions extracted by this `docker-collectd-plugin` are
packed into the `plugin_instance` dimension using the following syntax:

```
plugin_instance:value[dim1=value1,dim2=value2,...]
```

Because CollectD limits the size of the reported fields to 64 total
characters, trying to pack too many additional dimensions will result in
the whole constructed string from being truncated, making parsing those
additional dimensions correctly on the target metrics system impossible.
You should make sure that the total size of your `plugin_instance`
value, including all additional dimensions names and values, does not
exceed 64 characters.

## How to send only basic metrics

A lot of the metrics reported by this plugin are advanced statistics you
may not necessarily need. CollectD makes it easy to filter metrics you
don't want to record or send. The first step is to define a `PostCache`
chain with a rule to pass the Docker CollectD plugin's metrics through a
custom filtering chain.

```apache
LoadPlugin match_regex

<Chain "PostCache">
  <Rule>
    <Match "regex">
      Plugin "^docker$"
    </Match>
    <Target "jump">
      Chain "FilterOutDetailedDockerStats"
    </Target>
  </Rule>

  Target "write"
</Chain>
```

*Note:* make sure you have exactly _one_ `<Chain "PostCache">` section;
CollectD will consider the first one it sees, and ignore any other. If
you already have a section, simply add the `<Rule>` block that jumps to
the `FilterOutDetailedDockerStats` sub-chain to your existing
`PostCache` chain.

You can then use the following filtering chain to only allow the "basic"
CPU, memory, network and block I/O metrics to go through, and drop
everything else from the plugin:

```apache
<Chain "FilterOutDetailedDockerStats">
  <Rule "CpuUsage">
    <Match "regex">
      Type "^cpu.usage$"
    </Match>
    Target "return"
  </Rule>
  <Rule "CpuPercent">
    <Match "regex">
      Type "^cpu.percent$"
    </Match>
    Target "return"
  </Rule>
  <Rule "MemoryUsage">
    <Match "regex">
      Type "^memory.usage$"
    </Match>
    Target "return"
  </Rule>
  <Rule "MemoryPercent">
    <Match "regex">
      Type "^memory.percent$"
    </Match>
    Target "return"
  </Rule>
  <Rule "NetworkUsage">
    <Match "regex">
      Type "^network.usage$"
    </Match>
    Target "return"
  </Rule>
  <Rule "BlockIO">
    <Match "regex">
      Type "^blkio$"
      TypeInstance "^io_service_bytes_recursive*"
    </Match>
    Target "return"
  </Rule>

  Target "stop"
</Chain>
```

For convenience, you'll find those configuration blocks in the example configuration file [`10-docker.conf`](https://github.com/signalfx/integrations/blob/master/collectd-docker/10-docker.conf).

## How to send metrics about resource allocation

If a filter has been configured, then additional resource allocation metrics can be gathered by adding the following snippet to the plugin's filter configuration.

```apache
<Rule "CpuShares">
  <Match "regex">
    Type "^cpu.shares$"
  </Match>
  Target "return"
</Rule>
<Rule "CpuQuota">
<Match "regex">
  Type "^cpu.quota$"
</Match>
Target "return"
</Rule>
<Rule "CpuThrottlingData">
  <Match "regex">
    Type "^cpu.throttling_data$"
  </Match>
  Target "return"
</Rule>
<Rule "MemoryStats">
  <Match "regex">
    Type "^memory.stats$"
    TypeInstance "^swap"
  </Match>
  Target "return"
</Rule>
```

## How to filter by Names, Labels, and Image Names
It is now possible to filter by container names, image names, and labels.
Each filter supports python regular expression strings.

### Names
Containers with names matching the specified regular expression patterns
Because the Docker api reports container names with preceding '/' an appropriate regular expression pattern
should be `"/name"` or a substring match `".name"`

```apache
<Module dockerplugin>
  ...
  ExcludeName "/random_name" # This pattern matches for /random_name
  ExcludeName ".*random.*" # This pattern matches for the substring random_name
</Module>
```

### Labels
Labels are key/value pairs.  You may specify just a label key pattern to exclude all containers with a label whose key mattches the pattern regardless of it's value.  You may also specify a label key pattern and a pattern for the value.  When both key and value patterns are provided, they must both match for the container to be excluded.

```apache
<Module dockerplugin>
  ...
  ExcludeLabel ".hello." ".world." # This pattern exludes labels with the substrings "hello" and values with substring "world"
  ExcludeLabel ".hello." ".*" # This pattern excludes labels with the substring "hello" and all values
</Module>
```

### Image Names
```apache
<Module dockerplugin>
  ...
  ExcludeImage ".*elasticsearch.*" # This pattern matches for the substring elasticsearch
  ExcludeImage "^(?=elasticsearch)(?=.*:latest)" # This pattern looks for the word elasticsearch at the beginning of the name and tag latest
  ExcludeImage "^(?=.*search)(?=.*:latest)" # This pattern looks for the substring search in the name and tag latest
</Module>
```

## Requirements

* Docker 1.9+
* `docker`
* `jsonpath_rw`
* `python-dateutil`
* Python 3

## Amazon CloudWatch Agent
If you want to output to the Amazon Cloudwatch Agent, add the TypesDB of Docker plugin to Amazon Cloudwatch Agent configuration.

```json
"metrics_collected": {
  "collectd": {
    "metrics_aggregation_interval": 60,
    "collectd_typesdb": [
      "/usr/share/collectd/types.db",
      "/usr/share/docker-collectd-plugin/dockerplugin.db"
    ],
    "collectd_security_level": "none"
},
```

Add the Network plugin to collectd configuration.

```apache
LoadPlugin network
<Plugin "network">
  Server "127.0.0.1" "25826"
  ReportStats false
</Plugin>

TypesDB "/usr/share/collectd/types.db"
TypesDB "/usr/share/docker-collectd-plugin/dockerplugin.db"
LoadPlugin python
<Plugin python>
  ModulePath "/usr/share/docker-collectd-plugin"
  Import "dockerplugin"

  <Module dockerplugin>
    BaseURL "unix://var/run/docker.sock"
  </Module>
</Plugin>
```

Security level should be set to "encrypt" on the production environment:)
