---
layout: doctoc
title: Distributed Monitoring
---

{% include review_required.md %}

### Introduction

Naemon can be configured to support distributed monitoring of network services and resources.  I'll try to briefly explain how this can be accomplished...

### Goals

The goal in the distributed monitoring environment that I will describe is to offload the overhead (CPU usage, etc.) of performing service checks from a "central" server onto one or more "distributed" servers.  Most small to medium sized shops will not have a real need for setting up such an environment.  However, when you want to start monitoring hundreds or even thousands of <i>hosts</i> (and several times that many services) using Naemon, this becomes quite important.

### Reference Diagram

The diagram below should help give you a general idea of how distributed monitoring works with Naemon.  I'll be referring to the items shown in the diagram as I explain things...

<a href="images/svg/distributed.svg"><img src="images/svg/distributed.svg" class="svg-image" border="1"></a>

### Central Server vs. Distributed Servers

When setting up a distributed monitoring environment with Naemon, there are differences in the way the central and distributed servers are configured.  I'll show you how to configure both types of servers and explain what effects the changes being made have on the overall monitoring.  For starters, lets describe the purpose of the different types of servers...

The function of a <i>distributed server</i> is to actively perform checks all the services you define for a "cluster" of hosts.  I use the term "cluster" loosely - it basically just mean an arbitrary group of hosts on your network.  Depending on your network layout, you may have several clusters at one physical location, or each cluster may be separated by a WAN, its own firewall, etc.  The important thing to remember to that for each cluster of hosts (however you define that), there is one distributed server that runs Naemon and monitors the services on the hosts in the cluster.  A distributed server is usually a bare-bones installation of Naemon.  It doesn't have to have the web interface installed, send out notifications, run event handler scripts, or do anything other than execute service checks if you don't want it to.  More detailed information on configuring a distributed server comes later...

The purpose of the <i>central server</i> is to simply listen for service check results from one or more distributed servers.  Even though services are occasionally actively checked from the central server, the active checks are only performed in dire circumstances, so lets just say that the central server only accepts passive check for now.  Since the central server is obtaining <a href="passivechecks.html">passive service check</a> results from one or more distributed servers, it serves as the focal point for all monitoring logic (i.e. it sends out notifications, runs event handler scripts, determines host states, has the web interface installed, etc).

### Obtaining Service Check Information From Distributed Monitors

Okay, before we go jumping into configuration detail we need to know how to send the service check results from the distributed servers to the central server.  I've already discussed how to submit passive check results to Naemon from same host that Naemon is running on (as described in the documentation on <a href="passivechecks.html">passive checks</a>), but I haven't given any info on how to submit passive check results from other hosts.

In order to facilitate the submission of passive check results to a remote host, I've written the <a href="addons.html#nsca">nsca addon</a>.  The addon consists of two pieces.  The first is a client program (send_nsca) which is run from a remote host and is used to send the service check results to another server.  The second piece is the nsca daemon (nsca) which either runs as a standalone daemon or under inetd and listens for connections from client programs.  Upon receiving service check information from a client, the daemon will submit the check information to Naemon (on the central server) by inserting a <i>PROCESS_SVC_CHECK_RESULT</i> command into the <a href="configmain.html#command_file">external command file</a>, along with the check results.  The next time Naemon checks for <a href="extcommands.html">external commands</a>, it will find the passive service check information that was sent from the distributed server and process it.   Easy, huh?

### Distributed Server Configuration

So how exactly is Naemon configured on a distributed server?  Basically, its just a bare-bones installation.  You don't need to install the web interface or have notifications sent out from the server, as this will all be handled by the central server.

Key configuration changes:

<ul>
<li>Only those services and hosts which are being monitored directly by the distributed server are defined in the <a href="configobject.html">object configuration file</a>.</li>
<li>The distributed server has its <a href="configmain.html#enable_notifications">enable_notifications</a> directive set to 0.  This will prevent any notifications from being sent out by the server.</li>
<li>The distributed server is configured to <a href="configmain.html#obsess_over_services">obsess over services</a>.</li>
<li>The distributed server has an <a href="configmain.html#ocsp_command">ocsp command</a> defined (as described below).</li>
</ul>

In order to make everything come together and work properly, we want the distributed server to report the results of <i>all</i> service checks to Naemon.  We could use <a href="eventhandlers.html">event handlers</a> to report <i>changes</i> in the state of a service, but that just doesn't cut it.  In order to force the distributed server to report all service check results, you must enabled the <a href="configmain.html#obsess_over_services">obsess_over_services</a> option in the main configuration file and provide a <a href="configmain.html#ocsp_command">ocsp_command</a> to be run after every service check.  We will use the ocsp command to send the results of all service checks to the central server, making use of the send_nsca client and nsca daemon (as described above) to handle the transmission.

In order to accomplish this, you'll need to define an ocsp command like this:

```
ocsp_command=submit_check_result
```

The command definition for the <i>submit_check_result</i> command looks something like this:

```
define command{
	command_name	submit_check_result
	command_line	/usr/lib/naemon/plugins/eventhandlers/submit_check_result $HOSTNAME$ '$SERVICEDESC$' $SERVICESTATE$ '$SERVICEOUTPUT$'
	}
```

The <i>submit_check_result</i> shell scripts looks something like this (replace <i>central_server</i> with the IP address of the central server):

```
	#!/bin/sh
	# Arguments:
	#  $1 = host_name (Short name of host that the service is
	#       associated with)
	#  $2 = svc_description (Description of the service)
	#  $3 = state_string (A string representing the status of
	#       the given service - "OK", "WARNING", "CRITICAL"
	#       or "UNKNOWN")
	#  $4 = plugin_output (A text string that should be used
	#       as the plugin output for the service checks)
	#

	# Convert the state string to the corresponding return code
	return_code=-1

	case "$3" in
    	    OK)
        	        return_code=0
            	    ;;
	        WARNING)
    	            return_code=1
        	        ;;
	        CRITICAL)
    	            return_code=2
        	        ;;
	        UNKNOWN)
    	            return_code=-1
        	        ;;
	esac

	# pipe the service check info into the send_nsca program, which
	# in turn transmits the data to the nsca daemon on the central
	# monitoring server

	/bin/printf "%s\t%s\t%s\t%s\n" "$1" "$2" "$return_code" "$4" | /usr/local/nagios/bin/send_nsca -H <central_server> -c /usr/local/nagios/etc/send_nsca.cfg
```

The script above assumes that you have the send_nsca program and it configuration file (send_nsca.cfg) located in the <i>/usr/local/nagios/bin/</i> and <i>/usr/local/nagios/etc/</i> directories, respectively.

That's it!  We've successfully configured a remote host running Naemon to act as a distributed monitoring server.  Let's go over exactly what happens with the distributed server and how it sends service check results to Naemon (the steps outlined below correspond to the numbers in the reference diagram above):

<ol>
<li>After the distributed server finishes executing a service check, it executes the command you defined by the <a href="configmain.html#ocsp_command">ocsp_command</a> variable.  In our example, this is the <i>/usr/lib/naemon/plugins/eventhandlers/submit_check_result</i> script.  Note that the definition for the <i>submit_check_result</i> command passed four pieces of information to the script: the name of the host the service is associated with, the service description, the return code from the service check, and the plugin output from the service check.</li>
<li>The <i>submit_check_result</i> script pipes the service check information (host name, description, return code, and output) to the <i>send_nsca</i> client program.</li>
<li>The <i>send_nsca</i> program transmits the service check information to the <i>nsca</i> daemon on the central monitoring server.</li>
<li>The <i>nsca</i> daemon on the central server takes the service check information and writes it to the external command file for later pickup by Naemon.</li>
<li>The Naemon process on the central server reads the external command file and processes the passive service check information that originated from the distributed monitoring server.</li>
</ol>

### Central Server Configuration

We've looked at how distributed monitoring servers should be configured, so let's turn to the central server.  For all intents and purposes, the central is configured as you would normally configure a standalone server.  It is setup as follows:

<ul>
<li>The central server has the web interface installed (optional, but recommended)</li>
<li>The central server has its <a href="configmain.html#enable_notifications">enable_notifications</a> directive set to 1.  This will enable notifications. (optional, but recommended)</li>
<li>The central server has <a href="configmain.html#execute_service_checks">active service checks</a> disabled (optional, but recommended - see notes below)</li>
<li>The central server has <a href="configmain.html#check_external_commands">external command checks</a> enabled (required)</li>
<li>The central server has <a href="configmain.html#accept_passive_service_checks">passive service checks</a> enabled (required)</li>
</ul>

There are three other very important things that you need to keep in mind when configuring the central server:

<ul>
<li>The central server must have service definitions for <i>all services</i> that are being monitored by all the distributed servers.  Naemon will ignore passive check results if they do not correspond to a service that has been defined.</li>
<li>If you're only using the central server to process services whose results are going to be provided by distributed hosts, you can simply disable all active service checks on a program-wide basis by setting the <a href="configmain.html#execute_service_checks">execute_service_checks</a> directive to 0.  If you're using the central server to actively monitor a few services on its own (without the aid of distributed servers), the <i>enable_active_checks</i> option of the definitions for service being monitored by distributed servers should be set to 0.  This will prevent Naemon from actively checking those services.</li>
</ul>

It is important that you either disable all service checks on a program-wide basis or disable the <i>enable_active_checks</i> option in the definitions for each service that is monitored by a distributed server.  This will ensure that active service checks are never executed under normal circumstances.  The services will keep getting rescheduled at their normal check intervals (3 minutes, 5 minutes, etc...), but the won't actually be executed.  This rescheduling loop will just continue all the while Naemon is running.  I'll explain why this is done in a bit...

That's it!  Easy, huh?

### Problems With Passive Checks

For all intents and purposes we can say that the central server is relying solely on passive checks for monitoring.  The main problem with relying completely on passive checks for monitoring is the fact that Naemon must rely on something else to provide the monitoring data.  What if the remote host that is sending in passive check results goes down or becomes unreachable?   If Naemon isn't actively checking the services on the host, how will it know that there is a problem?

Fortunately, there is a way we can handle these types of problems...

### Freshness Checking

Naemon supports a feature that does "freshness" checking on the results of service checks.  More information freshness checking can be found <a href="freshness.html">here</a>.  This features gives some protection against situations where remote hosts may stop sending passive service checks into the central monitoring server.  The purpose of "freshness" checking is to ensure that service checks are either being provided passively by distributed servers on a regular basis or performed actively by the central server if the need arises.  If the service check results provided by the distributed servers get "stale", Naemon can be configured to force active checks of the service from the central monitoring host.

So how do you do this?  On the central monitoring server you need to configure services that are being monitoring by distributed servers as follows...

<ul>
<li>The <i>check_freshness</i> option in the service definitions should be set to 1.  This enables "freshness" checking for the services.</li>
<li>The <i>freshness_threshold</i> option in the service definitions should be set to a value (in seconds) which reflects how "fresh" the results for the services (provided by the distributed servers) should be.</li>
<li>The <i>check_command</i> option in the service definitions should reflect valid commands that can be used to actively check the service from the central monitoring server.</li>
</ul>

Naemon periodically checks the "freshness" of the results for all services that have freshness checking enabled.  The <i>freshness_threshold</i> option in each service definition is used to determine how "fresh" the results for each service should be.  For example, if you set this value to 300 for one of your services, Naemon will consider the service results to be "stale" if they're older than 5 minutes (300 seconds).  If you do not specify a value for the <i>freshness_threshold</i> option, Naemon will automatically calculate a "freshness" threshold by looking at either the <i>normal_check_interval</i> or <i>retry_check_interval</i> options (depending on what <a href="statetypes.html">type of state</a> the service is in).  If the service results are found to be "stale", Naemon will run the service check command specified by the <i>check_command</i> option in the service definition, thereby actively checking the service.

Remember that you have to specify a <i>check_command</i> option in the service definitions that can be used to actively check the status of the service from the central monitoring server.  Under normal circumstances, this check command is never executed (because active checks were disabled on a program-wide basis or for the specific services).  When freshness checking is enabled, Naemon will run this command to actively check the status of the service <i>even if active checks are disabled on a program-wide or service-specific basis</i>.

If you are unable to define commands to actively check a service from the central monitoring host (or if turns out to be a major pain), you could simply define all your services with the <i>check_command</i> option set to run a dummy script that returns a critical status.  Here's an example...  Let's assume you define a command called 'service-is-stale' and use that command name in the <i>check_command</i> option of your services.  Here's what the definition would look like...

```
define command{
	command_name	service-is-stale
	command_line	/usr/lib/naemon/plugins/check_dummy 2 "CRITICAL: Service results are stale"
	}
```

When Naemon detects that the service results are stale and runs the <b>service-is-stale</b> command, the <i>check_dummy</i> plugin is executed and the service will go into a critical state.  This would likely cause notifications to be sent out, so you'll know that there's a problem.

### Performing Host Checks

At this point you know how to obtain service check results passively from distributed servers.  This means that the central server is not actively checking services on its own.  But what about host checks?  You still need to do them, so how?

Since host checks usually compromise a small part of monitoring activity (they aren't done unless absolutely necessary), I'd recommend that you perform host checks actively from the central server.  That means that you define host checks on the central server the same way that you do on the distributed servers (and the same way you would in a normal, non-distributed setup).

Passive host checks are available (read <a href="passivechecks.html">here</a>), so you could use them in your distributed monitoring setup, but they suffer from a few problems.  The biggest problem is that Naemon does not translate passive host check problem states (DOWN and UNREACHABLE) when they are processed.  This means that if your monitoring servers have a different parent/child host structure (and they will, if you monitoring servers are in different locations), the central monitoring server will have an inaccurate view of host states.

If you do want to send passive host checks to a central server in your distributed monitoring setup, make sure:

<ul>
<li>The central server has <a href="configmain.html#accept_passive_host_checks">passive host checks</a> enabled (required)</li>
<li>The distributed server is configured to <a href="configmain.html#obsess_over_hosts">obsess over hosts</a>.</li>
<li>The distributed server has an <a href="configmain.html#ochp_command">ochp command</a> defined.</li>
</ul>

The ochp command, which is used for processing host check results, works in a similar manner to the ocsp command, which is used for processing service check results (see documentation above).  In order to make sure passive host check results are up to date, you'll want to enable <a href="freshness.html">freshness checking</a> for hosts (similar to what is described above for services).
