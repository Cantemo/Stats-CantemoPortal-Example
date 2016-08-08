# StatsD Cantemo Portal Example

This is an example on how to set up monitoring using statsd with
Cantemo Portal. This setup uses a docker image to setup StatsD,
Graphite and Grafana in order to gather periodic statistics and
display them in a web GUI.

## Important notes!

- This is not officially supported by Cantemo AB and intended as a
  resource to help setting up statistics gathering. We welcome pull
  requests from the community to help improve this setup.

# Prerequisites

- Installation of Cantemo Portal (tested with 2.4.x) setup and running.
- Root access to server that Cantemo Portal is installed on.
- A second machine with root access or access to deploy docker images

## Install docker

Start by installing docker and docker compose on your intended
server. Docker is available for Linux, Windows and Mac, but is best
supported on Linux so our recommendation is to run it on Linux. For
instructions on how to install it, see
https://docs.docker.com/engine/installation/ and
https://docs.docker.com/compose/install/ for instructions on how to
install.


## Deploy the docker image

To start up, run the following command

```
$ docker-compose -p portalstats up -d
```

You should now be able to access grafana at http://hostname

The default credentials are admin/admin

![Grafana default screen](images/home-screen.png?raw=true)



## Setting up datasource

The image does not come with a datasource configured by default. You
can set one up by clicking the grafana icon in the upper-left hand
corner of the page, then selecting "Data Sources" in the left-side
menu and finally "Add new" in the breadcrumb at the top of the page.

![Screenshot of Data Source setup](images/graphite-datasource-setup.png?raw=true)

## Uploading a dashboard

Graphite supports having multiple dashboards installed. This can be
used to gather statistics from many different kinds of system. In this
repo we have included an example dashboard with measurements of
various statistics gathered from Portal, Vidispine and the underlying
operating system.

To upload it, click on the dashboard dropdown at the top of the page,
which probably says Home at this point, and then click Import at the
bottom of the popup.

![The dashboard view](images/dashboard-list.png?raw=true)

Now click the Choose File and upload the file "Portal Metrics.json"
from this repository.

Make sure you save the newly imported dashboard, since it will be
stored in persistant storage otherwise.

There will not be any data displayed in the graphs, since we haven't
configured the system send any stats to it yet. If you see data, it is
most likely because you failed to set up a datasource earlier. Grafana
will then use a fake data source which just shows random data.

![Empty dashboard view](images/empty-metrics.png?raw=true)

# Setting up portal

To configure portal to send data points to StatsD, you need to add the following to portal.conf:

```
[statsd]
STATSD_HOST = statsd-host.example.com
STATSD_PREFIX =	portal-hostname
```

After restarting everything with:

```
$ supervisorctl restart all
```

you should start seeing data come into the Portal section on the dashboard.

# Setting up vidispine

Full documentation on setting up statsd with vidspine is available at
http://apidoc.vidispine.com/latest/system/monitoring.html#statsd

You need to PUT an xml document like:

```xml
<MetricsConfigurationDocument xmlns="http://xml.vidispine.com/schema/vidispine">
  <statsd>
    <host>statsd-host.example.com</host>
    <port>8125</port>
    <prefix>portal-hostname</prefix>
  </statsd>
</MetricsConfigurationDocument>
```

to http://hostname:8080/API/configuration/metrics. An example on how do to this is:

```
$ echo "<MetricsConfigurationDocument xmlns=\"http://xml.vidispine.com/schema/vidispine\"><statsd><host>statsd-host.example.com</host><port>8125</port><prefix>portal-host</prefix></statsd></MetricsConfigurationDocument>" | \
  curl -v --data-binary @- -H 'Content-Type: application/xml' -X PUT -u admin:admin http://portal-host:8080/API/configuration/metrics
```
This setting takes effect directly and you should start seeing data pop up on your dashboard.

![Vidispine metrics](images/vidispine-metrics.png?raw=true)

# Setting up Diamond

Diamond is a daemon which periodically gathers statistics from various
operating system components and sends them to StatsD. To install
diamond on a portal server, you first have to install some
prerequisite software:

```
$ yum install gcc python-setuptools python-devel postgresql-devel

$ easy_install pip
```

After this, you can install Diamond using

```
$ pip install diamond==4.0.41 statsd pyrabbit psycopg2
```


By default, all diamond metrics will be added using the system's hostname. If this is not the same as the portal-hostname you have used above you can set the following in /etc/diamond/diamond.conf:

```
hostname = portal-hostname
```

Enable statsd in diamond:

```
$ cat /etc/diamond/diamond.conf.example | sed -e 's,diamond\.handler.*,diamond.handler.stats_d.StatsdHandler,' > /etc/diamond/diamond.conf

$ sed -i -e 's,127.0.0.1,statsd-host.example.com,' /etc/diamond/handlers/StatsdHandler.conf
```

Enable some of the stats collectors in Diamond using the following commands:

```
$ sed -i -e 's,False,True,' \
  /etc/diamond/collectors/DiskUsageCollector.conf \
  /etc/diamond/collectors/DiskSpaceCollector.conf \
  /etc/diamond/collectors/ElasticSearchCollector.conf \
  /etc/diamond/collectors/PostgresqlCollector.conf \
  /etc/diamond/collectors/VMStatCollector.conf
```

This will enable the above metrics and will not require any configuration on a default portal install.

To enable RabbtiMQ stats, you will have to install the management plugin into rabbitmq using the command:

```
$ rabbitmq-plugins enable rabbitmq_management

$ service rabbitmq-server restart
```

Add the following to /etc/diamond/collectors/RabbitMQCollector.conf and restart diamond

```
enabled = True
host = localhost:15672
```

After this you should start seeing messages and memory usage appearing.

![OS Metrics](images/os-metrics.png?raw=true)

## Trouble-shooting

Here are some common errors you may encounter:

### Address in use

```
ERROR: for stats Cannot start service stats: driver failed
programming external connectivity on endpoint
statscantemoportalexample_stats_1
(f823b6bde76db6d5c3147c0b9aacc622c576da8572ed53f3d48c6bd4c99f180a):
Error starting userland proxy: Bind for 0.0.0.0:80: unexpected error
(Failure EADDRINUSE)
```

This happens because some other process is already listening to port
80. You can either stop that service or change the mapped port number
in docker-compose.yml to something different. This port is only used
by the web browser to communicate with Grafana and can be changed.



## References

- Docker - https://docs.docker.com/
- Docker Compose - https://docs.docker.com/compose/
- StatsD - https://github.com/etsy/statsd
- Graphite - https://graphiteapp.org/
- Grafana - http://grafana.org/
- Diamond - https://github.com/python-diamond/Diamond