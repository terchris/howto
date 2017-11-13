# Monitoring what is going on inside your CKAN installation

![Picture](http://urbalurba.no/dataset/46568ec0-d676-4a2a-a039-4479abf96fba/resource/07f26a58-fc7d-4532-9865-8c11d1ba4f3f/download/dockermoving.png) 

Maybe you, like me, have the unpleasant experience of been hacked. 
Seeing traffic going in and out from your site should be a happy feeling and not a reason to worry about if you are hacked. The cloud providers default tools just tells you that there is bytes going in and out,  that CPU is low or high, that disk write and read is going on.

But what is really happening. Is it all good or all bad ?


CKAN.org relies on several systems. These systems are:

- Postgresql - The database where all data is stored
- Redis - 
- Solr -
- Apache -
- Nginx - 
- Ubuntu - The operating system

This howto describes how you set up monitoring for these systems. If you, like me, don’t care what these systems do and just want them do what they are supposed to. Then this howto is for you.

<hr>


To know something about this you need to look inside the box. And you need to have some historic metering that can tell you if this is normal or not.

If you, like me, also want all this for free. Well, then this howto is definitely for you. 


## Datadog

Datadog is a monitoring & analytics tool that lives in the cloud. You install an agent on your server and the agent reports what is going on to https://www.datadoghq.com/

**screenshot

The screenshot shows the different systems that is used by CKAN. 

**more about what you can do - how to use it and so on

Datadog has a free tier. It allows you to  monitor up to 5 machines and it just have 1 day history on the metrics monitored in your server. But you get a good view of what is going on inside the box.


## Installing datadog

Installing the datadog agent
The agent is the component that collects information about the systems on your server and sends information to datadoghq.
In my case the server is a Ubuntu server. So I go to:
https://app.datadoghq.com/account/settings#agent/ubuntu

I just copy the line there and paste it to the terminal 

The line looks something like this:
```
DD_API_KEY=my-secret-key-of-letters bash -c "$(curl -L https://raw.githubusercontent.com/DataDog/dd-agent/master/packaging/datadog-agent/source/install_agent.sh)"
```

**do we need to run as sudo ?

When the package is installed you will see:
**what

Testing if the agent is running
```
sudo /etc/init.d/datadog-agent info
```
In order to have the agent collect information from a system you need to restart it
```
sudo /etc/init.d/datadog-agent restart
```

More commands here 
https://docs.datadoghq.com/guides/basic_agent_usage/ubuntu/

Looking at the metrics in datadoghq

**how to display CPU, disk, network 


## Creating the CKAN dashboard
Install all the monitoring components as described. Then come back here to create the dashboard that tells you what is going on inside your CKAN system.


**screenshots  


## Installing monitoring for the systems CKAN use
On a plane install all of these are running on the same host. 
The ckan config file (/etc/ckan/default/production.ini) has the following two lines
```
solr_url = http://127.0.0.1:8983/solr
ckan.redis.url = redis://localhost:6379/0
```

### Installing datadog for redis
create the redis config file /etc/dd-agent/conf.d/redisdb.yaml and put the following lines
```
init_config:

instances:
   -   host: localhost
       port: 6379
```
Save the file and change owner
***missing

Then restart the agent
```
sudo /etc/init.d/datadog-agent restart
```
and then
```
sudo /etc/init.d/datadog-agent info
```
You should see this:
```
  redisdb (5.18.1)
    ----------------
      - instance #0 [OK]
      - Collected 25 metrics, 0 events & 1 service check
```
This indicates that redis is now installed properly on your server

Now go to datadoghq and add the integration
Open https://app.datadoghq.com/account/settings

Search for redis and click install. You will see a tab. Click on the “Configuration” tab and then you see a button at the bottom named “Install integration”

The redis tile will move up from the Available integrations to the Installed integrations


### Installing datadog for solr

Go to the datadog directory
```
cd /etc/dd-agent/conf.d
```
Copy the example file 
```
sudo cp solr.yaml.example solr.yaml
```
and edit it. changing the following lines:
```
instances:
  - host: localhost
    port: 8983
```
Save the file and change owner
***missing

Then restart the agent to reload config
```
sudo /etc/init.d/datadog-agent restart
```
and then
```
sudo /etc/init.d/datadog-agent info
```

ERR : (can you people from datadog pls help) 
```
  solr (5.18.1)
    -------------
      - instance #solr-localhost-8983 [ERROR]: 'Cannot connect to instance localhost:8983. java.io.IOException: Failed to retrieve RMIServer stub: javax.naming.CommunicationException [Root exception is java.rmi.ConnectIOException: non-JRMP server at remote endpoint]' collected 0 metrics
      - Collected 0 metrics, 0 events & 0 service checks

(The datadog doc says something about installing Make sure that JMX Remote is enabled on your Solr server.
The install doc here seems to require me to run a java UI - this is not posible on a ssh terminal.
)
```

Now go to datadoghq and add the integration
Open https://app.datadoghq.com/account/settings

Search for solr and click install. You will see a tab. Click on the “Configuration” tab and then you see a button at the bottom named “Install integration”

The solr tile will move up from the Available integrations to the Installed integrations


### Installing datadog for postgres

Open the integration page in datadoghq
Open https://app.datadoghq.com/account/settings

Search for PostgreSQL and click Install.
On the Configuration tab there is a password generator. Click on it to generate a password.
Then on a terminal window on your server do
```
 sudo -u postgres psql
```
I get this output
```
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
	LANGUAGE = (unset),
	LC_ALL = (unset),
	LC_CTYPE = "UTF-8",
	LANG = "en_US.UTF-8"
    are supported and installed on your system.
perl: warning: Falling back to the standard locale ("C").
psql (9.3.19)
Type "help" for help.

postgres=# 
```
Copy the lines on the prompt. In my case it looks like this:
 
```
postgres=#  create user datadog with password my-secret-password-here;
CREATE ROLE
postgres=# grant SELECT ON pg_stat_database to datadog;
GRANT
```

To exit postgres 
```
postgres=# \q 
```
Then do 
```
sudo psql -h localhost -U datadog postgres -c "select * from pg_stat_database LIMIT(1);"  
 && echo -e "\e[0;32mPostgres connection - OK\e[0m" || \ ||  
echo -e "\e[0;31mCannot connect to Postgres\e[0m"
```

paste the password when you are prompted


You get a screen and it seems that you dont get the prompt back. So in order to exit paste 
```
 \q 
```

Create the file /etc/dd-agent/conf.d/postgres.yaml
```
sudo vi /etc/dd-agent/conf.d/postgres.yaml
```
Put the following lines in the file:

```
init_config:

instances:
   -   host: localhost
       port: 5432
       username: datadog
       password: my-secret-password-here
```
Change owner on the config file
```
 sudo chown dd-agent:dd-agent /etc/dd-agent/conf.d/postgres.yaml
```
Restart the datadog agent
```
sudo /etc/init.d/datadog-agent restart
```
and then
```
sudo /etc/init.d/datadog-agent info
```
You should see this:
```
   postgres (5.18.1)
    -----------------
      - instance #0 [OK]
      - Collected 7 metrics, 0 events & 1 service check
```

Now go to datadoghq and add the integration
Open https://app.datadoghq.com/account/settings

Search for PostgreSQL and click install. You will see a tab. Click on the “Configuration” tab and then you see a button at the bottom named “Install integration”

The PostgreSQL tile will move up from the Available integrations to the Installed integrations




### Installing datadog for apache

Check the version of apache 
```
apache2 -v
```
You should see this:
```
Server version: Apache/2.4.7 (Ubuntu)
Server built:   Sep 18 2017 16:37:54
```
Check if mod_status is installed:
```
apache2ctl -M | grep status
```
If you get this it is already installed and you can continue
```
 status_module (shared)
```
 You can test if it is running by using lynx
 ```
 lynx http://localhost/server-status?auto
 ```
(lynx is a command line web browser. Nice for testing that stuff is working without worrying about firewalls and other stuff interfering. Install it sudo apt-get install lynx )

 Copy the example config file to production
 ```
 sudo cp /etc/dd-agent/conf.d/apache.yaml.example /etc/dd-agent/conf.d/apache.yaml
```
Change owner on the config file
```
 sudo chown dd-agent:dd-agent /etc/dd-agent/conf.d/apache.yaml
```
Restart the datadog agent
```
sudo /etc/init.d/datadog-agent restart
```
and then
```
sudo /etc/init.d/datadog-agent info
```
You should see this:
```
    apache (5.18.1)
    ---------------
      - instance #0 [OK]
      - Collected 10 metrics, 0 events & 1 service check
```


Now go to datadoghq and add the integration
Open https://app.datadoghq.com/account/settings

Search for Apache and click install. You will see a tab. Click on the “Configuration” tab and then you see a button at the bottom named “Install integration”

The Apache tile will move up from the Available integrations to the Installed integrations


### Installing datadog for nginx

 Check your version 
 ```
 nginx -v
```
 You should see this:
```
 nginx version: nginx/1.4.6 (Ubuntu)
```
 Check if you have the  http_stub_status_module installed type:
 ```
nginx -V 2>&1 | grep -o with-http_stub_status_module 
```
If it is installed then the output is:
```
with-http_stub_status_module
```
Copy the config example to production
```
sudo cp /etc/dd-agent/conf.d/nginx.yaml.example /etc/dd-agent/conf.d/nginx.yaml
```
Change owner on the config file
```
 sudo chown dd-agent:dd-agent /etc/dd-agent/conf.d/nginx.yaml
```
 Restart the datadog agent
 ```
sudo /etc/init.d/datadog-agent restart
```
and then
```
sudo /etc/init.d/datadog-agent info
```
ERR:
```
 nginx (5.18.1)
    --------------
      - instance #0 [ERROR]: '404 Client Error: Not Found for url: http://localhost/nginx_status'
      - Collected 0 metrics, 0 events & 1 service check
 
```

Now go to datadoghq and add the integration
Open https://app.datadoghq.com/account/settings

Search for nginx and click install. You will see a tab. Click on the “Configuration” tab and then you see a button at the bottom named “Install integration”

The nginx tile will move up from the Available integrations to the Installed integrations


