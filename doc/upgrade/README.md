# Upgrading from v3 to v4

The configuration for v4 is *somewhat* compatible with the v3
configuration.  It should be possible to reuse most of a v3
reconfiguration with minor tweaks.  This file describes the
differences between v3 and v4.  It does not contain a step-by-step
process for upgrading the server.

In general, we have the following changes:

* most module configuration is very close to v3
* most of the `unlang` processing is very close to v3
* each `server` section need a `namespace` parameter
* Packet processing sections are now `recv Access-Request`, etc.  Not `authorize`, etc.
* each `listen` section needs to be converted to the v4 format

## Upgrading from older versions

### Upgrading from v3

When upgrading, please start with the default configuration of v4.
Then, move your v3 configuration over, one module at a time.  Check
this file for differences in module configuration, and update the
module to use the new configuration.  Start the server after every
change via `radiusd -XC` to see if the configuration is OK.  Then,
convert the `listen` sections, followed by the `server` sections.

Take your time.  It is better to make small incrementatal progress,
than to make massive changes, and then to spend weeks debugging it.

### Upgrading from v2

If you're upgrading from v2 you should read the v3 version of this
file.  It describes changed from v2 to v3.  This file describes only
the changes from v3 to v4.0

## radiusd.conf

The following configurations have been removed.  See the new `listen`
sections for their replacements.

`cleanup_delay` - replaced with `cleanup_delay` in a `listen` section

`reject_delay` - see `mods-available/delay`.  You should list `delay`
last in any `send Access-Reject section.

`status_server` - see `type = Status-Server` in a new `listen` section.

The `log` section has been updated to remove many configuration items
which are specific to RADIUS, and to Access-Request packets.  Please
see `sites-available/default`, and look for the `Access-Request`
subsection there.  See also [`templates.conf`](templates.conf) for
a way to regain one global configuration for Access-Request
packets.

## Virtual Servers

There are some changes to the virtual servers in v4.  First, every
virtual server has to begin with an entry:

    namespace = ...

For RADIUS, use:

    namespace = radius

This tells the server what protocol is being used in that virtual
server.  This configuration was not necessary in v3, because each
protocol was implemented (mostly) in the RADIUS state machine.  In v4,
each protocol is completely independent, and RADIUS is no longer
welded into the server core.

Each example virtual server in the `mods-enabled/` directory contains
a `namespace` parameter.

### Listen Sections

The `listen` sections have changed.  There is now a `type` entry,
which lists the packet types (e.g. `Access-Request`) instead of
`auth`.  To accept multiple kinds of packets, just list `type`
multiple times`:

    type = Access-Request
    type = Accounting-Request

Each `listen` section also has a `transport` entry.  This
configuration can be left out for "headless" servers, such as
"inner-tunnel".  For example, setting UDP transport is done v ia:

    transport = udp

Each type of transport has its configuration stored in a subsection
named for that transport:

    transport = udp
    udp {
        ... udp transport configuration ...
    }

For `udp`, the configuration entries are largley the same as for v3.
e.g. `ipaddr`, `port`, etc.

The `listen` section then compiles each [Processing
Section](#processing-sections) based on the named packet types.  It
has a `recv` section for receiving packets, and a `send` section for
sending packets.  e.g.

    recv Access-Request {
       ... unlang ...
    }
    
    send Access-Accept {
        ... unlang ....
    }

This configuration is different from v3.  The benefit is that it is
much easier to understand.  Instead of using things like
`Post-Auth-Type Reject`, we now have just `send Access-Reject`.

See also [Processing Sections](#processing-sections) for how the
`unlang` statements are parsed.

### Clients

The server supports global clients in the `clients.conf` file, as with v3.

Client can also be defined in a `client` subsection of a virtual
server.  Unlike v3, there is no need to have a `clients` section that
contains a `client` definition.  See `sites-available/default` for examples.

The server also supports dynamic clients.  See
`sites-available/dynamic_clients` for a worked example.  There are
many changes from v3.  First, there is no need to have a `client`
definition that contains a network.  Instead, there is a `network`
section which has a number of `allow` and `deny` rules.  Second,
dynamic clients can be defined on a per-connection basis.  Finally,
the `sites-available/dynamic_clients` virtual server has full access
to the entire RADIUS packet.

The result of these changes is that it is now possible to have
multiple clients behind a NAT gateway, and to have different shared
secrets for each client.  e.g. by keying off of the `NAS-Identifier`
attribute.

The dynamic client functionality behaves the same for all protocols
supported by the server.  e.g. RADIUS, DHCP, VMPS, TACACS+, etc.

## Processing Sections

All of the processing sections have been renamed.  Sorry, but this was
required for the new features in v4.

| Old Name			| New Name
|-------------------------------|-----------------------
| authorize			| recv Access-Request
| authenticate			| authenticate <Auth-Type>
| post-auth			| send Access-Accept
|				| 
| preacct			| recv Accounting-Request
| accounting			| send Access-Accept
|				| 
| recv-coa			| recv CoA-Request
| send-coa			| send CoA-ACK
| send-coa			| send CoA-NAK
|				| 
| Post-Auth-Type Reject		| send Access-Reject
| Post-Auth-Type Challenge	| send Access-Challenge

i.e. instead of the section names being (mostly) randomly named, the
names are now consistent.  The `recv` sections receive packets from
the network.  The `send` sections send packets back to the network.
The second name of the section is the *type* of the packet that is
being received or sent.

## Proxying

Proxying has undergone massive changes.  The `proxy.conf` file no
longer exists, along with everything in it.  e.g. `realm`,
`home_server`, `home_server_pool`.  The old proxying functionality was
welded into the server core, which made many useful features
impossible to configure.

The `radius` module now handles basic proxying to home servers.  We
recommend creating one instance of the `radius` module per home
server.  e.g.

    radius home_server_1 {
       ... configuration for home server 1 ...
    }

You can then use `home_server_1` in any processing section, and the
request will be proxied when processing reaches the module.

For ease of management, we recommend naming the modules for the host
name of the home server.

It is often simplest to do proxying via an `authenticate proxy`
section, though that section can have any name.  e.g. setting
`Auth-Type := proxy` will call the `authenticate proxy` section, and
is similar to the previous setting `Proxy-To-Realm`.

    authenticate proxy {
        home_server_1
    }

For more detailed examples, see the Wiki page:
https://wiki.freeradius.org/upgrading/version4/proxy.  That page also
describes how to upgrade a v3 configuration to the new v4 style.

### home_server

The `home_server` configuration has been replaced with the `radius`
module.  See `raddb/mods-available/radius` for examples and
documentation.

### home_server_pool

The `home_server_pool` configuration has been replaced with standard
unlang configurations.  The various load-balancing options can be
re-created using in-place 'unlang' configuration.

The mappings for `type` are as follows:

* `type = fail-over` - replaced with 'unlang'

    redundant {
        home_server_1
	home_server_2
	home_server_3
    }

Note, of course, you will have to use the names of the `radius`
modules in your configuration, and not `home_server_1`, etc.

* `type = load-balance` - replaced with 'unlang'

    load-balance {
        home_server_1
	home_server_2
	home_server_3
    }

* `type = client-balance` - replaced with 'unlang'

    load-balance "%{%{Packet-Src-IP-Address}:-%{Packet-Src-IPv6-Address}}" {
        home_server_1
	home_server_2
	home_server_3
    }


* `type = client-port-balance` - replaced with 'unlang'

    load-balance "%{%{Packet-Src-IP-Address}:-%{Packet-Src-IPv6-Address}}-%{Packet-Src-Port}" {
        home_server_1
	home_server_2
	home_server_3
    }

* `type = keyed-balance` - replaced with 'unlang'

    load-balance "%{Load-Balance-Key}" {
        home_server_1
	home_server_2
	home_server_3
    }

While the `Load-Balance-Key` was a special attribute in v3, it has no
special meaning in v4.  You can use any attribute or string expansion
as part of the `load-balance` key.

### Things which were impossible in v3

In v3, it was impossible to proxy the same request to multiple
destinations.  This is now trivial.  In any processing section, just do:

    ...
    home_server_1
    home_server_2
    ...

When processing reaches that point, it will proxy the request to
home_server_1, followed by home_server_2.

This functionality can be used to send Accounting-Request packets to
multiple destinations.

You can also catch "failed" proxying, and do something else.  In the
example below, try to proxy to home_server_1, if that fails, just
"accept" the request.

    ...
    home_server_1
    if (fail) {
        accept
    }
    ...

### CoA and Originate-Coa

The `sites-available/originate-coa` virtual server has been removed.
The functionality has been replaced with the `fork` keyword.

## Dictionaries

The `struct` data type is now supported.  See `man dictionary`.

There are many more sanity checks and helpful messages for people
who create new dictionaries.

## Attribute references

In previous versions of the user attributes could be referred to
by their name only e.g. `if (User-Name == 'foo')`.

To allow for more thorough error checking, the requirement to prefix
attribute references with `&` is now strictly enforced.

Common places which will need to be checked and corrected are the
left and right hand side of `update {}` sections, and conditions.

The v3 server has warned about using non prefixed attribute references
for some time.  If users have paid attention to those warnings, few
modifications will be required.

Use of attributes in xlats e.g. `%{User-Name}` remains unchanged.
There is no plan to require prefixes here.

As of v33, the preferred format for "unknown" attributes is
`&Attr-oid.oid.oid`, e.g. `&Attr-26.11344.255`.  However, v3 would
still parse (but not generate) attributes of the form
`Vendor-FreeRADIUS-Attr-255`.  The `Vendor-` syntax has been removed
in version 4.  The server would never produce such names, and enabling
them made attribute parsing significantly more complex.

## Update sections and Filtering

The filtering operators in v4 have been modified.  They no longer
*set* the attribute to a value.  Instead, they only *filter* the
attribute list, and delete any attributes which do not match.  The
filtering operators do not *create* any attribute.

In order to achieve the same functionality as v3, you will need to set
the value of the attribute, in addition to filtering it.

## load-balance and redundant-load-balance sections

Before v4, the `load-balance` sections implemented load balancing by
picking a child at random.  This meant that load balancing was
probabilistically fair, but not perfectly fair.

In v4, `load-balance` sections track how many requests are in each
sub-section, and pick the subsection which is used the least.  This is
like the v3 proxy behavior of load balancing across home server pools.

The `load-balance` and `redundant-load-balance` sections now allow for
a load-balance key:

    load-balance "%{Calling-Station-Id}" {
        module1
	module2
	module3
	...
    }

If the key exists, it is hashed, and used to pick one of the
subsections.  This behavior allows for deterministic load-balancing,
similar to the v3 proxy `keyed-balance` configuration.

## Connection timeouts

In v3 and earlier, the config items for configuring connection
timeouts were either confusingly named, or completely absent in
the case of many contributed modules.

In v4, connection timeouts can be configured universally for all
modules with the `connect_timeout` config item of the module's `pool
{}` section.

The following modules will apply `connect_timeout`:

- rlm_rest
- rlm_linelog (network connections only)
- rlm_ldap
- rlm_couchbase
- rlm_cache_memcached
- rlm_redis_* (all the redis modules)
- rlm_sql_cassandra
- rlm_sql_db2
- rlm_sql_freetds
- rlm_sql_mysql
- rlm_sql_unixodbc

Some modules such as rlm_sql_postgresql can have their timeout set via an alternative
configuration item (e.g. `radius_db` in the case of postgresql).

## New Modules

The following modules are new in v4.

### rlm_radius

The `radius` module has taken over much of the functionality of
`proxy.conf`.  See `raddb/mods-available/radius` for documentation and
configuration examples.

The `radius` module connects to one home server, just like the
`home_server` configuration in v3.  Some of the configuration items
are similar to the `home_server` configuration, but not all.

The module can send multiple packet types to one home server.
e.g. Access-Request and Accounting-Request.

This module also replaces the old 'coa' and 'originate-coa'
configuration.  See also `fork` for creating child requests that are
different from the parent requests.

Unlike v3, the module can do asynchronous proxying.  That is, proxying
where the server controls the retransmission behavior.  In v3, the
server retransmitted proxied packets only when it received a
retransmission from the NAS.  That behavior is good, but there are
times where retransmitting packets at the proxy is better.

## Changed Modules

The following modules exhibit changed behaviour.

### rlm_cache

`&control:Cache-Merge` has been renamed to `&control:Cache-Merge-New` and controls 
whether new entries are merged into the current request.  It defaults to `no`.
The primary use case, is if you're using xlat expansions in the cache module itself
to retrieve information for caching, and need the result of those expensions to be
available immediately.

Two new control attributes `&control:Cache-Allow-Merge` and `&control:Cache-Allow-Insert`
have been added.  These control whether existing entries are to be merged, and new entries
created on the next call to a cache module instance. Both default to `yes`.

### rlm_eap

All certificate attributes are available in the `&session-state:` list,
immediately after they're parsed from their ASN1 form.

The certificates are no longer added to the `&request:` list.  You are
advised to update any references during the upgrade to 4.0:

    s/TLS-Cert-/session-state:TLS-Cert-/

The `rlm_eap_ikev2` module was removed.  It does not follow RFC
5106, and no one was maintaining it.

The `rlm_eap_tnc` module was removed.  No one was using or maintaining
it.

The in-memory SSL cache was removed.  Changes in OpenSSL and
FreeRADIUS made it difficult to continue using the OpenSSL
implementation of a cache.  See `raddb/sites-available/tls-cache`
for a better replacement.  The OpenSSL cache can now be placed on
disk, in memory, in memcache, or in a redis cache.  The result is
higher performance, and is more configurable.

The `use_tunneled_reply` and `copy_request_to_tunnel`
configuration items have been removed.  Their functionality has been
replaced with the `use_tunneled_reply` and
`copy_request_to_tunnel` policies.  See
`raddb/sites-available/inner-tunnel` and `raddb/policy.d/eap` for
more information.

These configuration items were removed because they caused issues for
a number of users, and they made the code substantially more
complicated.  Experience shows that having configurable policies in
`unlang` is preferable to having them hard-coded in C.

### rlm_eap_pwd

The `virtual_server` configuration has been removed from EAP-PWD.  The
module now looks for &request.control:Cleartext-Password.

### rlm_exec

Exec-Program and Exec-Program-Wait have been removed.

The `packet_type` configuration has been removed.  Use `unlang` checks
to see if you want to execute the module.

### rlm_expr

Allow `&Attr-Name[*]` to mean "sum".  Previously, it just referred to
the first attribute.

Using `%{expr:0 + &Attr-Name[*]}` will cause it to return the sum of the values
of all attributes with the given name.

Note that `%{expr:1 * &Attr-Name[*]}` does *not* mean repeated
multiplication.  Instead, the sum of the attributes is taken as
before, and then the result is multiplied by one.

### rlm_mschap

The `winbind_*` configuration options are now in a `winbind`
subsection.  See `mods-available/mschap` for details.

### rlm_perl

Attributes of type `octets` are now passed directly to Perl as binary
data, instead of as hex strings.

All data received from the network is marked "tainted" by default.

### rlm_rest

`REST-HTTP-Code` is now inserted into the `&request:` list instead of the `&reply:`
list, to be compliant with the [list usage](http://wiki.freeradius.org/guide/List-Usage) guidelines.

### rlm_sql

Driver-specific options have moved from `mods-available/sql` to
`mods-config/sql/driver/<drivername>`.

#### rlm_sql_mysql

Now calls `mysql_real_escape_string` and no longer produces
`=<hexit><hexit>` escape sequences in expanded values.  The
`safe_characters` config item is ignored when using MySQL databases.

#### rlm_sql_postgresql

Now calls `PQescapeStringConn` and no longer produces
`=<hexit><hexit>` escape sequences in expanded values.  The
`safe_characters` config item is ignored when using PostgreSQL
databases.

### rlm_sqlcounter

Attribute references:

The following config items must now be defined as attribute references::

    key
    count_attribute
    counter_name
    check_name
    reply_name

For example where in v3 you would specify the attribute names as::

    count_attribute	= Acct-Session-Time
    counter_name	= Daily-Session-Time
    check_name		= Max-Daily-Session
    reply_name		= Session-Timeout
    key			= User-Name

In v4 they must now be specified as::

    count_attribute	= &Acct-Session-Time
    counter_name	= &Daily-Session-Time
    check_name		= &control:Max-Daily-Session
    reply_name		= &reply:Session-Timeout
    key                 = &User-Name

Just adding the `&` prefix to the attribute name is not sufficient.
Attributes must be qualified with the list to search in, or add to.

This allows significantly greater flexibility, and better integration
with newer features in the server such as CoA, where reply_name can
now be `&coa:Session-Timeout`.  That allows the server to send a CoA
packet which updates the `Session-Timeout` for the user.

### rlm_sqlippool

The `ipv6` configuration item has been deleted.  It was deprecated in
3.0.16.

Instead, use `attribute-name`.  See `mods-available/sqlippool` for
more information.

## Deleted Modules

The following modules have been deleted

### rlm_counter

Instead of using this, please use the `sqlcounter` module with sqlite.

It is difficult to maintain multiple implementations of the same
functionality.  As a result, we have simplified the server by removing
duplicate functionality.

### rlm_ippool

Instead of using this, please use the `sql_ippool` module with sqlite.

It is difficult to maintain multiple implementations of the same
functionality.  As a result, we have simplified the server by removing
duplicate functionality.

