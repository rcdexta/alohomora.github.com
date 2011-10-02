---
layout: post
title: "Deploying a Rails Application on TorqueBox"
date: 2011-10-02 14:57
comments: true
categories: [rails, jruby]
---
{% img left /images/posts/knob.png %}
Previously, we used Warbler/Tomcat setup to deploy our JRuby on Rails applications. We were working on performance tuning one of those applications and came across [TorqueBox](http://torquebox.org/). Support for rack applications, scheduled jobs, built-in high-performance cache store on top of [Infinispan](http://www.jboss.org/infinispan) and [JBoss AS](http://www.jboss.org/jbossas) application server all sounded pretty exciting. So, we went ahead and migrated our application to TorqueBox. We could clearly see improvements in performance and the actual benchmarking comparisons will be published in another post in the future. 

Though the TorqueBox documentation is well written, when it comes to migrating a fully-featured application to another framework, we faced several hiccups. I had to look-up several blogs, forums and many SO threads to resolve those issues. This article aims at documenting all that information in once place.

<!--more-->

###Adding TorqueBox to your application###

Add the `torquebox` gem to your Gemfile only for the deployment environments. This is because TorqueBox gems have several dependencies and this affects the load time for development and test environments. It is essential that you specify the platform as `jruby` to make bundler install the dependencies correctly. 

{% codeblock Gemfile lang:ruby %}
group :qa, :staging, :production do
  gem 'torquebox', :platforms => :jruby
end
{% endcodeblock %}

Installing the gem gives us access to a couple of rake tasks that assist in deployment and also the TorqueBox cache store client, in case your application uses caching.

###Installing the TorqueBox binary###

Let's assume you are configuring TorqueBox in the production environment. The next step is to install the binary distribution that contains the TorqueBox server.  You can download the latest stable version Torque 1.1.1 [here](http://repository-torquebox.forge.cloudbees.com/release/org/torquebox/torquebox-dist/1.1.1/torquebox-dist-1.1.1-bin.zip). Please note that you need to have at least Java JDK 6 installed on your deployment machine for TorqueBox to work.

Unzip the binary to some folder on your machine. I assume the folder name as `torquebox-current`. Add the following environment variables to your `.bash_profile`:

{% codeblock .bash_profile lang:ruby %}
export JAVA_HOME=/usr/java/jdk1.6.0_25
export TORQUEBOX_HOME=$HOME/torquebox-current
export JRUBY_HOME_=$$TORQUEBOX_HOME/jruby
export JBOSS_HOME=$TORQUEBOX_HOME/jboss
{% endcodeblock %}

The TorqueBox binary comes along with JRuby. If you already have JRuby installed as part of the RVM, you can change the `JRUBY_HOME` path accordingly. Also, your `JAVA_HOME` must point to the appropriate JRE path.

###Deploying your application on JBoss###

TorqueBox needs a file `torquebox.yml` inside your config folder. This file is called the deployment descriptor which is simply a configuration of the various components required at the time of deployment. Let's create that file.

{% codeblock config/torquebox.yml lang:ruby %}
application:
  env: production
pooling:
  web:
    min: 4
    max: 6
{% endcodeblock %}

You will have to specify the deployment environment and the number of min and max runtimes. This [page](http://torquebox.org/documentation/1.1.1/pooling.html) discusses the various types of runtime pools and helps you figure out what works best for you application. What we have defined here is a bounded pool. From the documentation...

> A bounded pool is a typical resource pool with minimum and maximum capacity. Each interpreter managed by the pool is given out to a single client at a time. It is unavailable for any other client until the current owner returns it to the pool. The pool will ensure that a minimum number of interpreters are kept in the pool at all times. Additionally, a maximum capacity is specified to ensure that the pool does not grow unbounded. 

`RAILS_ENV=production rake torquebox:archive` will package the application into a java archive. The archive would be of the form `<app_name>.knob` where knob is a special suffix that TorqueBox uses. Copy the archive to the `$TORQUEBOX_HOME/apps` folder.

To test your application, execute `$JBOSS_HOME/bin/run.sh`. This will boot up the JBOSS instance on port `8080` and spawn the minimum runtimes that we specified.

###Daemonizing the application server###

If your deployment environment is Ubuntu, TorqueBox offers a couple of rake tasks to add JBOSS server as an upstart service.

{% codeblock lang:ruby %}
torquebox:upstart:check    #Check if TorqueBox is installed as an upstart service
torquebox:upstart:install  #Install TorqueBox as an upstart service
torquebox:upstart:restart  #Restart TorqueBox when it is running as an upstart service
torquebox:upstart:start    #Start TorqueBox when it is installed as an upstart service
torquebox:upstart:stop     #Stop TorqueBox when it is installed as an upstart service
{% endcodeblock %}

In our case, it was Cent OS. Refer to [Step 5 of this article](http://davidghedini.blogspot.com/2011/03/install-jboss-6-on-centos.html) to add JBOSS as a service. After including that script, we should be able to start the server using `service jboss start`

###Some performance tuning tips###

**Adding the InfiniSpan Cache Store**

   You can replace the typical MemCache store with the TorqueBox cache store. The cool thing is that all the runtimes will share this cache store.

{% codeblock config/production.rb %}
MyJRubyApp::Application.configure do
  config.action_controller.perform_caching = true
  config.cache_store = :torque_box_store
end
{% endcodeblock %}

**Turning on gzip compression**

To enable gzip compression on all the responses, edit the JBoss server descriptor and add `compression="force"`

{% codeblock default/deploy/jbossweb.sar/server.xml %}
<Connector protocol="HTTP/1.1" port="${jboss.web.http.port}" address="${jboss.bind.address}"
         redirectPort="${jboss.web.https.port}" compression="force" />
{% endcodeblock %}

**Optimizing the Java memory params**

Our setup spawns 6 runtimes at peak load and we should be setting the heap size accordingly. Edit `$JBOSS_HOME/bin/run.conf` and set the values accordingly. In our experience, 6 runtimes never required more than `3098m` of heap size. Also, it is a good practice to set `xms` and `xmx` to the same values.

{% codeblock $JBOSS_HOME/bin/run.conf lang:sh %}
if [ "x$JAVA_OPTS" = "x" ]; then
   JAVA_OPTS="-Xms3098m -Xmx3098m -XX:MaxPermSize=512m -Dorg.jboss.resolver.warning=true -Dsun.rmi.dgc.client.gcInterval=3600000 -Dsun.rmi.dgc.server.gcInterval=3600000"
fi
{% endcodeblock %}