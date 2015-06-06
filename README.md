# Installing InfluxDB and Grafana

Install InfluxDB and Grafana, you can find the URL for the latest RPMs from
the InfluxDB and Grafana websites:

```sh
sudo yum -y install $url_to_influxdb.rpm
sudo yum -y install $url_to_grafana.rpm
```

Check that your max file descriptors is sensible:

```sh
sysctl fs.file-max
```

Should be > 10k. If not increase it by editing `/etc/sysctl.conf`.

```sh
sudo systemctl enable influxdb
sudo systemctl start influxdb
```

Check that InfluxDB is running by browsing to port 8083.

```sh
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

Check that Grafana is running by browsing to port 3000.

Set up a database in InfluxDB and a data source in Grafana to point to that
database. 

# Connecting up CollectD

First install CollectD:

```sh
sudo yum -y install collectd
```

Then edit the configuration for both InfluxDB and CollectD. For the InfluxDB
configuration we want to update the CollectD input plugin, enabling it and
specifying the port to listen on, the database to send to and the types:

`/opt/influxdb/share/config.toml`:

```ini
# Configure the collectd api
[input_plugins.collectd]
enabled = true
# address = "0.0.0.0" # If not set, is actually set to bind-address.
port = 25826
database = "metrics"
# types.db can be found in a collectd installation or on github:
# https://github.com/collectd/collectd/blob/master/src/types.db
typesdb = "/usr/share/collectd/types.db" # The path to the collectd types.db file
```

Then configure CollectD.

```sh
sudo vim /etc/collectd.conf
```

Uncomment the network plugin and update the network plugins settings to
point to the InfluxDB CollectD API.

`/etc/collectd.conf`:

```aconf
<Plugin network>
#       # client setup:
        Server "127.0.0.1" "25826"
#       <Server "239.192.74.66" "25826">
#               SecurityLevel Encrypt
#               Username "user"
#               Password "secret"
#               Interface "eth0"
#       </Server>
#       TimeToLive 128
#
#       # server setup:
#       Listen "ff18::efc0:4a42" "25826"
#       <Listen "239.192.74.66" "25826">
#               SecurityLevel Sign
#               AuthFile "/etc/collectd/passwd"
#               Interface "eth0"
#       </Listen>
#       MaxPacketSize 1452
#
#       # proxy setup (client and server as above):
#       Forward true
#
#       # statistics about the network plugin itself
#       ReportStats false
#
#       # "garbage collection"
#       CacheFlush 1800
</Plugin>
```

```sh
sudo systemctl enable collectd
sudo systemctl start collectd
```

Check that CollectD is sending data using `tcpdump`:

```sh
sudo yum -y install tcpdump
sudo tcpdump -i lo -p -n -s 1500 udp
```

You should now be able to see new series in your InfluxDB admin. You
can set up dashboards through Grafana. CPU is a count, if you want to
see the values in percent you should use the aggregation function
`derivative`.

# Connecting up StatsD

Install the EPEL release RPMS, install NodeJS, NPM and then StatsD and
the InfluxDB backend for StatsD:

```sh
sudo yum -y install epel-release
sudo yum -y update
sudo yum -y install nodejs npm
sudo npm install -g statsd statsd-influxdb-backend
```

Create the StatsD configuration file:

```sh
vi statd.js
```

`statd.js`:

```js
{
  backends: [ "statsd-influxdb-backend" ],
  influxdb: {
    host: "127.0.0.1",
    port: 8086,
    database: "metrics",
    username: "statsd",
    password: "statsd"
  }
}
```

Then start up StatsD:

```sh
sudo statsd statd.js 2&>1 >>statsd.log &
```

You can use netcat to write some test data to StatsD:

```sh
sudo yum -y install nc
echo "user_registration:1|c" | nc -u -w10 127.0.0.1 8125
```

You should be able to see this new series in InfluxDB and set up a
new graph to plot these values.
