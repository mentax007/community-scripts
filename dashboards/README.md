# dashboards

There are a couple of mechanisms you can deploy that will get your Kazoo data into dashboards/dashboarding systems.  This will explain some of those options.

## Method 1: Graphite / Grafana / InfluxDB / sup / Crossbar / etc ( i.e. Graphed statistics )

This section explains how to take time-series data you can gather from your servers (i.e. Concurrent Calls, Active Registrations, etc), stick them in a time-series 
database like InfluxDB, and get them up on a display using something like Grafana to display them beautifully.

You'll need a few things to get this to work.  I may glaze over some bits, please feel free to comment on anything that needs more explanation.

I've put things in order of necessity, if you're comortable and understand what each component is for, feel free to skip parts.

### Step 1: InfluxDB 
Purpose: Influx is the data store, a fast time series database

#### Install InfluxDB 

Full instructions here: https://docs.influxdata.com/influxdb/v1.4/introduction/installation/
As of this writing, the current version is 1.4

NOTE: InfluxDB will need to listen on (by default, anyways) TCP Ports 2003 (a graphite input handler), 8083 (admin gui), 8086 (api url) 

Quick instructions:
* Configure InfluxDB Repo:
```
cat <<EOF | sudo tee /etc/yum.repos.d/influxdb.repo
[influxdb]
name = InfluxDB Repository - RHEL \$releasever
baseurl = https://repos.influxdata.com/rhel/\$releasever/\$basearch/stable
enabled = 1
gpgcheck = 1
gpgkey = https://repos.influxdata.com/influxdb.key
EOF
```

* Install InfluxDB:
```
yum install influxdb
```

#### Configure InfluxDB

It should install into /etc/influxdb, so edit /etc/influxdb/influxdb.conf.  All you really have to do is find the graphite input handler plugin and enable it, set the port, and specify the database to write to, like so:
Add by the end:

```
[[graphite]]
  # Determines whether the graphite endpoint is enabled.
  enabled = true
  database = "kazoo_stats"
  # retention-policy = ""
  bind-address = ":2003"
  # protocol = "tcp"
  # consistency-level = "one"
```

The database hasnt been created yet, but I'll make the presumption you're going to call your database __kazoo_stats__


#### Start InfluxDB

```
systemctl start influxdb; systemctl enable influxdb
```

Follow the official instructions for creating your first database here: https://docs.influxdata.com/influxdb/v1.4/introduction/getting_started/

```
influx -precision rfc3339
CREATE DATABASE kazoo_stats
```


-or-

```
As of version 1.3, the web admin interface is no longer available in InfluxDB. 
The interface does not run on port 8083 and InfluxDB ignores the [admin] section 
in the configuration file if that section is present. Chronograf replaces the web 
admin interface with improved tooling for querying data, writing data, and database management. 
See Chronograf’s transition guide for more information.
```

My Quick Setup Instructions:  Hit http://ip_or_hostname_of_influxdb_server:8083

Login with username root password root - then click cluster admins, and change your password right away.

Click disconnect in the top right, then log back in with your new username and password.

Click databases, fill in the database name __grafana-stats__ and then click Create Database.  You can leave the shard defaults as is.

Click on your newly created database name, and add a user that will be used to put data in this database only.  Write it down cause you'll need it in step 2.  You dont need to make it an admin user.

Now repeat the database + user creation process once more, this time for a database we'll call __grafana-dashboards__ where grafana will store its saved dashboards.

InfluxDB is all done!


### Step 2: Graphite-API 
Purpose: Grafana (or other tools) that are written to query Graphite can instead query a lightweight Graphite-API; and Graphite-API knows how to read natively from InfluxDB

We're using just the thinnest version of Graphite, a small python script called Graphite-API (https://github.com/brutasse/graphite-api).  It speaks the expected graphite "language" so things like grafana can speak to it, but we've replaced its data store with InfluxDB. 

Graphite-API docs are here: http://graphite-api.readthedocs.org/en/latest/

#### Install Graphite-API

First, lets install the [EPEL](https://fedoraproject.org/wiki/EPEL) repo for Centos 7. Click the link if you need the repo for another version.

```
yum install epel-release
```
Install Graphite-API:
```
yum install graphite-api

```

Create log file and adjust permissions:

```
touch /var/log/graphite-api.log
chown graphite-api:graphite-api /var/log/graphite-api.log
```

#### Configure Graphite-API

Edit the file /etc/graphite-api.yaml and fill it like so:

```
search_index: /var/lib/graphite-api/index
finders:
  - graphite_influxdb.InfluxdbFinder
influxdb:
   host: HOSTNAME_OR_IP_OF_SERVER_WITH_INFLUXDB_ON_IT
   port: 8086
   user: INFLUXDB_USER_YOU_CREATED
   pass: INFLUXDB_PASSWORD_YOU_CREATED
   db:   kazoo_stats
   cheat_times: true
cache:
    # memcache seems to have issues with storing the series. the data is too big and doesn't store it, and doesn't allow bumping size in config that much
    #CACHE_TYPE: 'memcached'
    #CACHE_KEY_PREFIX: 'graphite_api'
    CACHE_TYPE: 'filesystem'
    CACHE_DIR: '/tmp/graphite-api-cache'
logging:
  version: 1
  disable_existing_loggers: true
  handlers:
    file:
      class: logging.FileHandler
      filename: /var/log/graphite-api.log
    stdout:
      class: logging.StreamHandler
  loggers:
    graphite_api:
      handlers:
        - stdout
      propagate: true
      level: INFO
    graphite_influxdb:
      handlers:
        - stdout
      propagate: true
      level: INFO
    root:
      handlers:
        - stdout
      propagate: true
      level: INFO
```

The search_index file path needs to exist. You can place it anywhere you like, but to keep with the path in the example, you'll need to create the directory and make sure whichever user you're running gunicorn as can write to it:

```
mkdir -p /data/graphite/storage/
```

#### Start Graphite-API

It will be listening on tcp port 127.0.0.1:8888, as you can see in the command line below. If you want to change that, go ahead.  Make sure its accessible though.   

```
systemctl start graphite-api; systemctl enable graphite-api
```

### Step 3: Netcat 
The most lightweight, simple, quick way of taking a value and putting it in the database (note: this is not encrypted at all)

#### Install netcat 

```
yum install nc
```

#### Test a basic metric

The graphite metric format is:  METRICNAME METRICVALUE POSIXDATESTAMP

Here's a small script you can use to test it:

~~~
#!/bin/sh
TIMESTAMP=`date +%s`
METRIC=metric.test.thing
VALUE=10

echo $METRIC $VALUE $TIMESTAMP | nc HOST_OR_IP_OF_INFLUXDB_SERVER 2003
~~~

Now, go browse the InfluxDB web admin page (http://IP_OF_YOUR_INFLUXDB:8083), click on "Explore data" next to your database __grafana-stats__  and see if you can do "select * from metric.test.thing;" It should return one value of 10.


### Step 4: Grafana
Purpose: To actually draw the awesome dashboards! It wants to speak to a graphite server, which graphite-api is handling for us, which will then in turn query InfluxDB.

#### Install Grafana

We're going to install Grafana via grafana-authentication-proxy.  It's a small wrapper that adds proper authentication and per-user-dashboards to grafana. Its not the usual way you'd install it, but trust me, you'll want it. So, install grafana-authentication-proxy.  I put mine in /usr/local:

```
cd /usr/local
yum install -y git
git clone https://github.com/voxter/grafana-authentication-proxy.git
cd grafana-authentication-proxy/
git submodule init
git submodule update
yum install -y npm nodejs-grunt-cli
npm install
```

Update grafana to the latest version (and base it off the proper repo):

```
cd grafana && git checkout master && git pull
npm install
grunt
```

#### Configure Grafana / Grafana-authentication-proxy

Edit /usr/local/grafana-authentication-proxy/config.js

Depending on wether or not you're using elasticsarch (in this case, it would be for storing user dashboards) you can specify your ES server information here too, and it will proxy front end user auth for ES queries as well. Otherwise you can leave it alone, and we'll store our dashboards in InfluxDB

Pay attention to the following lines in the config file and update them accordingly:

```
    "graphiteUrl": "http://ip_or_host_of_your_graphite_api_server:8000", // This address must be publicly reachable

    "cookie_secret": "cookie_rhymes_with_c",
    
    "enable_basic_auth": true,
    
    "basic_auth_file": "",
        "basic_auth_users": [
            {"user": "username1", "password": "passwd1"},
            {"user": "username2", "password": "passwd2"},
        ],
```

You can choose to enable Google OAuth, or basic auth, or whatever you want - but this example we've only shown how to enable Basic Auth.

Once your config is done, start the node application with:

```
cd /usr/local/grafana-authentication-proxy
node app.js &

Server starting...
Info: HTTP Basic Authentication applied
Warning: No Google OAuth2 presented
Warning: No CAS authentication presented
Server listening on 9202
```

It will (by default) use InfluxDB to store saved dashboards.  You can also use elasticsearch if you've got that set up already, but since we didnt cover it in this tutorial, I'll skip that.


#### Deploy Grafana 

We're going to use nginx to serve grafana up over HTTP.

So first, make sure you've got nginx installed:

```
yum install -y nginx
```

Then take the grafana.conf I've included in this repo in the nginx/conf.d file, and place it in your own /etc/nginx/conf.d directory.

Make the necessary changes to the hostnames in the config file (if necessary), or simply remove /etc/nginx/conf.d/default.conf so the grafana.conf becomes the default host (if you dont have any other virtual hosts running on this server) and restart nginx.

#### Test it out!

Hit http://your_grafana_server in a web browser, it should ask you for one of the usernames you put in the grafana-authentication-proxy config.js file.  Once you log in, you should see the grafana page, and if everything is working properly, there should be a graph at the bottom of the page with simulated data on it!  Click the title of the graph, then click edit.

Click the green add query button, and then click the first 'select metric' box that has now appeared.  You should see "metric" - repeat, then you'll seee "test", once more and you'll see "thing" - this is the metric we added a moment ago when we were testing InfluxDB!

### Step 5: Collect some data!

OK, so we have a working datastore backend, a graphite Input handler, query API, and a frontend to make it all pretty.  Now, how do we get some kick ass system and kazoo data in there?

Let's start with a very simple bash script that we can cron.  I've included a script in this repo called system_stats.sh. Run it to make sure it works:

~~~
# ./system_stats.sh
kazoo.calls.inbound.net.voxter.dc1.prod.media01.total 7 1413233954
kazoo.calls.outbound.net.voxter.dc1.prod.media01.total 7 1413233954
kazoo.calls.inbound.net.voxter.dc1.prod.media02.total 7 1413233954
kazoo.calls.outbound.net.voxter.dc1.prod.media02.total 6 1413233954
~~~

This is the data that would have been sent to the graphite input handler, so you're set!  Stick that puppy in a crontab and come back in an hour.

~~~
/etc/crontab

*/10 * * * * root /usr/local/bin/graphite_log_total_calls > /dev/null 2>&1
~~~

Now go build yourself some dashboards using the data!




## Method 2: Log data from rsyslog -> logstash -> elasticsarch -> kibana

Other kinds of data you need to write queries to parse your log data (be it kamailio logs, 2600hz-platform logs, freeswitch logs, etc) and display the compiled data in a meaningful way (i.e. most popular account receiving REGISTER packets, most popular destination city for outgoing calls, etc.)

COMING SOON...
