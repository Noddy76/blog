---
layout: post
title:  "Installing Graphite 0.10 on Debian Jessie"
date:   2015-08-03 00:00:00
categories: graphite debian jessie
---

I wanted a Graphite server to record time series data from my home automation setup and I really wanted a feature introduced after the version in Debian. I could manually patch some of the python files as described <a href="http://grafana.org/docs/performance/">here</a> but I thought I would have a go and see if I could install it myself.

So starting with a fresh install of Debian Jessie.
 
Install the required dependenciesi.
<pre>
# apt-get install -y python python-pip python-cairo python-django \
python-django-tagging libapache2-mod-wsgi python-twisted python-memcache \
python-pysqlite2 apache2 libapache2-mod-wsgi python-simplejson
</pre>

Now install everything using Pip.

<pre>
# pip install https://github.com/graphite-project/ceres/tarball/master
# pip install whisper
# pip install carbon
# pip install graphite-web
</pre>

The settings for Carbon are defined in a file called <code>local_settings.py</code> located in <code>/opt/graphite/webapp/graphite</code>. There is an example of this file in the same directory called <code>local_settings.py.example</code>. Copy this file.

<pre>
# cp /opt/graphite/webapp/graphite/local_settings.py.example \
/opt/graphite/webapp/graphite/local_settings.py
</pre>

You should only need to change two values in this file. <code>SECRET_KEY</code> should be some secret random value and <code>TIME_ZONE</code> should be your local timezone to use when rendering the graphs.

<figure>
<figcaption>/opt/graphite/webapp/graphite/local_settings.py</figcaption>
<pre>
SECRET_KEY = 'blah blah blah'
TIME_ZONE = 'Europe/London'
</pre></figure>

You will also need to create <code>graphite.wsgi</code> and <code>carbon.conf</code>. You shouldn't need to change any values in these files so you can just copy the examples.
<pre>
# cp /opt/graphite/conf/graphite.wsgi.example /opt/graphite/conf/graphite.wsgi
# cp /opt/graphite/conf/carbon.conf.example /opt/graphite/conf/carbon.conf
</pre>

Next we create two files to describe the storage schemas used and the method of aggregation used to roll values up to the lower resolution buckets.

<figure>
<figcaption>/opt/graphite/conf/storage-schemas.conf</figcaption>
<pre>
[carbon]
pattern = ^carbon\.
retentions = 60:90d

[statsd]
pattern = ^stats.*
retentions = 10s:1d,1m:7d,10m:1y,1h:10y

[collectd]
pattern = ^collectd
retentions = 10s:1d,1m:28d,1h:10y

[default_1min_for_1day]
pattern = .*
retentions = 60s:1d
</pre>
</figure>

<figure>
<figcaption>/opt/graphite/conf/storage-aggregation.conf</figcaption>
<pre>
[min]
pattern = \.min$
xFilesFactor = 0.1
aggregationMethod = min

[max]
pattern = \.max$
xFilesFactor = 0.1
aggregationMethod = max

[count]
pattern = \.count$
xFilesFactor = 0
aggregationMethod = sum

[lower]
pattern = \.lower(_\d+)?$
xFilesFactor = 0.1
aggregationMethod = min

[upper]
pattern = \.upper(_\d+)?$
xFilesFactor = 0.1
aggregationMethod = max

[sum]
pattern = \.sum$
xFilesFactor = 0
aggregationMethod = sum

[default_average]
pattern = .*
xFilesFactor = 0.5
aggregationMethod = average
</pre>
</figure>

Now it's time to configure the web application. First create the database and change ownerships. You will be asked to create a superuser.

<pre>
# cd /opt/graphite/webapp/graphite
# python manage.py syncdb
# chown -R www-data:www-data /opt/graphite/storage/
</pre>


And now tell Apache about the Graphite web application.

<figure>
<figcaption>/etc/apache2/sites-available/graphite.conf</figcaption>
<pre>
&lt;VirtualHost *:80&gt;
  DocumentRoot "/opt/graphite/webapp"
  ErrorLog /opt/graphite/storage/log/webapp/error.log
  CustomLog /opt/graphite/storage/log/webapp/access.log common

  # I've found that an equal number of processes & threads tends
  # to show the best performance for Graphite (ymmv).
  WSGIDaemonProcess graphite processes=5 threads=5 display-name='%{GROUP}' inactivity-timeout=120
  WSGIProcessGroup graphite
  WSGIApplicationGroup %{GLOBAL}
  WSGIImportScript /opt/graphite/conf/graphite.wsgi process-group=graphite application-group=%{GLOBAL}

  WSGIScriptAlias / /opt/graphite/conf/graphite.wsgi

  Alias /content/ /opt/graphite/webapp/content/
  &lt;Location "/content/"&gt;
    SetHandler None
  &lt;/Location&gt;

  # XXX In order for the django admin site media to work you
  # must change @DJANGO_ROOT@ to be the path to your django
  # installation, which is probably something like:
  # /usr/lib/python2.6/site-packages/django
  Alias /media/ "@DJANGO_ROOT@/contrib/admin/media/"
  &lt;Location "/media/"&gt;
    SetHandler None
  &lt;/Location&gt;

  # The graphite.wsgi file has to be accessible by apache. It won't
  # be visible to clients because of the DocumentRoot though.
  &lt;Directory /opt/graphite/conf/&gt;
    Order deny,allow
    Allow from all
    Require all granted
  &lt;/Directory&gt;
  &lt;Directory "/opt/graphite/webapp/"&gt;
    Options +ExecCGI
    Order deny,allow
    Allow from all
    Require all granted
  &lt;/Directory&gt;
&lt;/VirtualHost&gt;
</pre>
</figure>

Now disable the default graphite site and  enable the Graphite one we just created.

<pre>
# a2dissite 000-default.conf
# a2ensite graphite
</pre>

We need carbon to start on system boot so we need to tell SystemD about it. Create a file called <code>/etc/systemd/system/carbon-cache.service</code> containing the following.
<figure>
<figcaption>/etc/systemd/system/carbon-cache.service</figcaption>
<pre>
[Unit]
Description=Graphite Carbon Cache
After=network.target

[Service]
Type=forking
ExecStartPre=/bin/rm -f /opt/graphite/storage/carbon-cache-a.pid
ExecStart=/opt/graphite/bin/carbon-cache.py start
ExecReload=/bin/kill -USR1 $MAINPID
PIDFile=/opt/graphite/storage/carbon-cache-a.pid

[Install]
WantedBy=multi-user.target
</pre>
</figure>

Now enable it by running <code>systemctl enable carbon-cache</code>.

We can now either reboot or run the following commnad to start carbon and get Apache to reload it's config.

<pre>
# systemctl start carbon-cache
# systemctl reload apache2
</pre>
 
Carbon should be available by pointing a browser at the IP address of your server.

