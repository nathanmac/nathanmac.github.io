---
layout: post
title:  "Installing Kibana on Laravel Homestead"
description: "Visualizing and Exploring your Elastic Search Data"
date:   2015-09-05 09:00:00
tags: [elasticsearch, kibana, laravel, homestead]
comments: true
---

Kibana 4 is an analytics and visualization platform that builds on Elasticsearch to give you a better understanding of
your data. Here we cover how to quickly and easily get Kibana up and running on you're Laravel Homestead environment.

![Kibana Screenshot](/assets/images/posts/kibana.png)

For ease of installation we can run the installation script using a single cli command via wget, just ssh into you
Laravel Homestead VM and run the following command and the installation script will run.

{% highlight bash %}
sh -c "$(wget https://gist.githubusercontent.com/nathanmac/ee18320275fdbc7614c1/raw/febfdb1b6ab8ffecde3dfd52397f52b3f2e80fb8/kibana.sh -O -)"
{% endhighlight %}

Wanna see what's happening checkout the installation script in full below or checkout the [gist](https://gist.github.com/nathanmac/ee18320275fdbc7614c1).

{% highlight bash %}
#!/usr/bin/env bash

echo ">>> Installing Kibana"

# Set some variables
KIBANA_VERSION=4.1.1 # Check https://www.elastic.co/downloads/kibana for latest version

sudo mkdir -p /opt/kibana
wget --quiet https://download.elastic.co/kibana/kibana/kibana-$KIBANA_VERSION-linux-x64.tar.gz
sudo tar xvf kibana-$KIBANA_VERSION-linux-x64.tar.gz -C /opt/kibana --strip-components=1
rm kibana-$KIBANA_VERSION-linux-x64.tar.gz

# Configure to start up Kibana automatically
cd /etc/init.d && sudo wget --quiet https://gist.githubusercontent.com/thisismitch/8b15ac909aed214ad04a/raw/bce61d85643c2dcdfbc2728c55a41dab444dca20/kibana4

sudo chmod +x /etc/init.d/kibana4
sudo update-rc.d kibana4 defaults 95 10
sudo service kibana4 start
{% endhighlight %}

#### Optional Port Configuration
By default once you've installed Kibana using the script above you should now have access to the Kibana UI from
[http://192.168.10.10:5601](http://192.168.10.10:5601).
*(the IP address may differ if you're not using the default configuration for homestead)*

Great so we're up and running, however just to make it a little easier and accessible from [http://localhost:5601](http://localhost:5601).
We can use the [Laravel Homesteads](http://laravel.com/docs/5.1/homestead#ports) configuration file to setup some
custom port forwarding configurations.

{% highlight bash %}
$ homestead edit
{% endhighlight %}

Edit you're homestead.yaml file and add the following port configuration, you can set the 'to' port to suit you're own needs
and preference.

{% highlight bash %}
ports:
     - send: 5601
       to: 5601
{% endhighlight %}

Restart Homestead to apply the new port settings, once you're backup and running you're ready to access the
Kibana UI via [http://localhost:5601](http://localhost:5601/).

And start exploring you're data...