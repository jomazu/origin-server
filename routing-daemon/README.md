Configuring ActiveMQ
--------------------

The ActiveMQ node routing plug-in must be enabled so that it sends routing
updates that the programs in this directory can read.  Install
`rubygem-openshift-origin-routing-activemq` and see its included `README.md` for
instructions.

Configuring the Daemon
----------------------

The daemon must be configured to connect to ActiveMQ. Edit
`/etc/openshift/routing-daemon.conf` and set `ACTIVEMQ_USER`,
`ACTIVEMQ_PASSWORD`, `ACTIVEMQ_HOST`, and `ACTIVEMQ_DESTINATION` to the
appropriate credentials, address, and ActiveMQ destination (topic or
queue).

Exactly one routing module must be enabled.  A module for F5 BIG-IP LTM, a
module for an routing implementing the LBaaS REST API, and a module that
configures nginx as a reverse proxy are included in this repository.  Edit
`/etc/openshift/routing-daemon.conf` to set the `LOAD_BALANCER` setting to "f5",
"lbaas", or "nginx", and then follow the appropriate module-specific
configuration described below.

Internally, the routing daemon logic is divided into controllers, which
encompass higher-level logic, and models, which encompass the logic for
communicating with load balancers.  These controllers include a simple
controller that immediately dispatches commands to the load balancer (used for
nginx and F5), a controller that includes logic to batch configuration changes
and only dispatch commands to the load balancer at an interval (configurable
using the `UPDATE_INTERVAL` option in `routing-daemon.conf`; used if `LOAD_BALANCER` is
set to `f5_batched`), and an asynchronous controller for load balancers (such as
LBaaS) where it is necessary first to issue commands and then to poll for an
asynchronous confirmation on each command.

For testing purposes, a dummy model, which merely prints actions that
a normal model performs rather than performing actions itself, is also
included.  If you specify the "dummy" module, then you will get the
dummy model with the simple controller; if you specify the "dummy_async"
module, then you will get the dummy model with the asynchronous
controller.

Using F5 BIG-IP LTM
-------------------

Edit `/etc/openshift/routing-daemon.conf` to set the appropriate values for
`BIGIP_HOST`, `BIGIP_USERNAME`, `BIGIP_PASSWORD`, `BIGIP_SSHKEY`,
`BIGIP_DEVICE_GROUP`, `VIRTUAL_SERVER`, and `VIRTUAL_HTTPS_SERVER` to match your
F5 BIG-IP LTM configuration.

F5 BIG-IP LTM must be configured with two virtual servers, one for HTTP traffic
and one for HTTPS traffic. Each virtual server needs to be assigned at least one
VIP.  It is not necessary that each virtual server have a unique VIP; the HTTP
virtual server and the HTTPS virtual servers may share a VIP.  Once the LTM
virtual servers have been created, update `VIRTUAL_SERVER` and
`VIRTUAL_HTTPS_SERVER` in `/etc/openshift/routing-daemon.conf` to match the
names you have used.

A default client-ssl profile must also be configured as the default SNI
client-ssl profile. Although the naming of the default client-ssl profile is
unimportant, it does need to be added to the HTTPS virtual server.

The LTM "admin" user's 'Terminal Access' must be set to 'Advanced shell' so that
remote shell commands may be executed. Additionally, for the remote key
management commands to execute, the `BIGIP_SSHKEY` public key must be added to
the "admin" user's `.ssh/authorized_keys` file.

If you configure F5 BIG-IP LTM with a device group, use the `BIGIP_DEVICE_GROUP`
to specify the name of this device group.  If this setting is specified, the
daemon will synchronize the device group at the interval specified by the
`UPDATE_INTERVAL` interval, or the default value of 5 if `UPDATE_INTERVAL` is
left unset.

On initialization, the daemon will create a local-traffic policy named
"openshift_application_aliases" and add it to the HTTP and HTTP virtual servers
if such a policy does not already exist.

As it runs, the daemon will automatically create pools and associated policy
rules for applications, add and manage policy rules and SSL certificates and
keys for aliases, add members to the pools, delete members from the pools, and
delete empty pools and unused policy rules when appropriate.  The daemon will
create the pools in the "/Common" partition; see "Pool and Route Names" below
regarding pool names.  The daemon will also create rules in the
"openshift_application_aliases" policy to forward requests to pools comprising
the proxy gears of the respective applications based on the "Host:" header of
incoming HTTP requests.

Using LBaaS
-----------

After enabling the LBaaS module as described in the section on configuring the
daemon, edit `/etc/openshift/routing-daemon.conf` to set the appropriate values
for `LBAAS_HOST`, `LBAAS_TENANT`, `LBAAS_TIMEOUT`, `LBAAS_OPEN_TIMEOUT`,
`LBAAS_KEYSTONE_HOST`, `LBAAS_KEYSTONE_USERNAME`, `LBAAS_KEYSTONE_PASSWORD`, and
`LBAAS_KEYSTONE_TENANT`, to match your LBaaS configuration.

Using nginx
-----------

Edit `/etc/openshift/routing-daemon.conf` to set the appropriate values for
`NGINX_CONFDIR`, `NGINX_SERVICE`, `NGINX_SSL_CERTIFICATE`, `NGINX_SSL_KEY`,
`HTTP_PORT`, and `SSL_PORT`.

The daemon will automatically create and manage configuration files under the
directory specified by `NGINX_CONFDIR`: a `server.conf` file with the frontend
server configuration for all applications, `pool_*.conf` files with the backend
configuration, `alias_*_.conf` files for application aliases, and `*.key` and
`*.crt` files for application aliases and custom SSL certificates.  The
`NGINX_SSL_CERTIFICATE` and `NGINX_SSL_KEY` settings specify default SSL
configuration for applications.  `HTTP_PORT` and `SSL_PORT` specify the nginx
listen ports.  After each update, the daemon will reload the service specified
by `NGINX_SERVICE`.


Pool and Route Names
--------------------

By default, new pools will be created with the name
`pool_ose_{appname}_{namespace}_80` while new routes will be created with the
name `route_ose_{appname}_{namespace}.`  You can override these defaults by
setting appropriate values for the `POOL_NAME` and `ROUTE_NAME` settings,
respectively.  The values for these settings should contain the following
formats so that each application gets its own uniquely named pool and routing
rule: `%a` is expanded to the name of the application, and `%n` is expanded to
the application's namespace.

Monitors
--------

The F5 and LBaaS backends can add an existing monitor to newly created pools.
The following settings control how these monitors are created.

Set the `MONITOR_NAME` to the name of the monitor you would like to use, and set
`MONITOR_PATH` to the pathname to use for the monitor, or leave either option
unspecified to disable the monitor functionality.

Set `MONITOR_UP_CODE` to the code that indicates that a pool member is up, or
leave `MONITOR_UP_CODE` unset to use the default value of "1".

Set `MONITOR_TYPE` to either "http-ecv" or "https-ecv", depending on whether you
want to use HTTP or HTTPS for the monitor, or leave `MONITOR_TYPE` unset to use
the default value of "http-ecv".

Set `MONITOR_INTERVAL` to the interval at which the monitor will send requests,
or leave `MONITOR_INTERVAL` unset to use the default value of "10".

Set `MONITOR_TIMEOUT` to the monitor's timeout for its requests, or leave
`MONITOR_TIMEOUT` unset to use the default value of "5".

As with `POOL_NAME` and `ROUTE_NAME`, `MONITOR_NAME` and `MONITOR_PATH` both can
contain `%a` and `%n` formats, which are expanded the same way.  Unlike
`POOL_NAME` and `ROUTE_NAME`, you may or may not want to re-use the same monitor
for different applications.  The daemon will automatically create a new monitor
when `MONITOR_NAME` expands a string that does not match the name of any
existing monitor.

It is expected that for each pool member, the load balancer will send a `GET`
request to the resource identified on that host by the value of `MONITOR_PATH`
for the associated monitor, and that the host will respond with the value of
`MONITOR_UP_CODE` if the host is up, or some other response if the host is not
up.

Endpoint Types
--------------

By default, the routing daemon adds only proxy gears to pools.  Specifically,
the routing daemon ignores any gear endpoints that do not have the
"load_balancer" type.  These "load_balancer" endpoints are the endpoints for
HAProxy gears. Thus, requests coming into the external load-balancer (nginx or
F5 BIG-IP) will be routed through applications' HAProxy gears to reach the
application gears.

Routing requests through HAProxy gears enables the auto-scaling logic to react
to changes in load.  However, because only scalable applications have HAProxy
gears, this approach also means that only scalable applications can be reached
through the external load-balancer.

The `ENDPOINT_TYPES` setting specifies which gear or endpoint types the routing
daemon will add to pools.  This setting is provided for flexibility, but it is
not recommended to change it from the default value of "load_balancer".

##Notice of Export Control Law

This software distribution includes cryptographic software that is subject to the U.S. Export Administration Regulations (the "*EAR*") and other U.S. and foreign laws and may not be exported, re-exported or transferred (a) to any country listed in Country Group E:1 in Supplement No. 1 to part 740 of the EAR (currently, Cuba, Iran, North Korea, Sudan & Syria); (b) to any prohibited destination or to any end user who has been prohibited from participating in U.S. export transactions by any federal agency of the U.S. government; or (c) for use in connection with the design, development or production of nuclear, chemical or biological weapons, or rocket systems, space launch vehicles, or sounding rockets, or unmanned air vehicle systems.You may not download this software or technical information if you are located in one of these countries or otherwise subject to these restrictions. You may not provide this software or technical information to individuals or entities located in one of these countries or otherwise subject to these restrictions. You are also responsible for compliance with foreign law requirements applicable to the import, export and use of this software and technical information.
