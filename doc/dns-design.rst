..
    Portions created or assigned to Joe Hildebrand <jhildebr@cisco.com>. are
    Copyright (c) Joe Hildebrand <jhildebr@cisco.com>.  All Rights Reserved.
..

.. meta::
   :description: BlinkRez Design
   :author: Matthew A. Miller <mamille2@cisco.com>
   :copyright: Joe Hildebrand <jhildebr@cisco.com>.  All Rights Reserved.
   :dateModified: 2011-06-01

BlinkRez Design
===============

.. contents:: Table of Contents

.. sectnum::

Basics
------

The BlinkRez C API performs lookups in an asynchronous, threadless manner. It
provides some primitives for basic DNS query operations, and implements a
mechanism to establish socket connections in rough compliance to `Happy
Eyeballs (IPv6)`_ and `Happy Eyeballs (SCTP)`_.  The API is intended to be
extensible to adapt to new technologies.

This API consists of three major objects: policy context, low-level resolver,
and socket connector.  The policy context provides a common configuration
point for lookup and connection policies, and provide the main point of
extension for the API; the low-level resolver performs the basic lookup of
addresses for names, automatically following aliases and service targets; the
socket connector attempts to establish a socket for a service and/or domain
name.

Policy Context
--------------

The policy context, blinkrez_ctx, establishes the initial rules for lookups and
socket connections.  From it, the other two API types are created.

Properties
~~~~~~~~~~

The properties of a blinkrez_ctx each have their own getters; setters are not
exposed. While the getter functions for these properties operate against
the blinkrez_ctx, it may actually use the settings blinkrez_htable.

* selector (``struct event_base *``) - This is the event_base that lookups and
  socket connectors are selected/polled against.  It can be set as part of
  configuring the context (see below).

Configuration Settings
~~~~~~~~~~~~~~~~~~~~~~

The blinkrez_ctx will have a number of settings, which it will accept during its
creation as a blinkrez_htable. The most common settings will likely have
convenience functions (operating over the blinkrez_htable).  The use of generic
maps/hashtables allows for maximum flexibility, at the expense of some
simplicity.  However, the default settings for a blinkrez_ctx are appropriate for
most uses.

* selector (``struct event_base *``) - This is the selector/polling event base
  to work against; useful for processing resolution requests in line with other
  network operations. By default, each blinkrez_ctx creates it's own if this
  setting is NULL.
* allow_ipv4 (``bool``) - This is a flag to indicate if IPv4 resolves and
  connections should be attempted. Setting this to false effectively ignores
  the {ip_preference}.  The default is true.
* allow_ipv6 (``bool``) - This is a flag to indicate if IPv6 resolves and
  connections should be attempted. Setting this to false effectively ignores
  the {ip_preference} setting.  The default is true.
* ip_preference (``int``) - This denotes the initial preference for IPv4 or
  IPv6, as a scalar.  This value is positive if IPv6 addresses are preferred,
  negative if IPv4 addresses are preferred, and 0 to denote no preference.
  The default is 1, to prefer (slightly) IPv6 over IPv4.
* transports_ipv4 (``const char **``) - This is a list of transports to attempt
  to connect with via IPv4.  Derived socket connectors will attempt each
  transport during a connect call, provided there is a configured
  implementation for that transport (see `Extensibility`_ below) . The default
  is a list with one string, "tcp".
* transports_ipv6 (``const char **``) - This is a list of transports to attempt
  to connect with via IPv6.  Derived socket connectors will attempt each
  transport during a connect call, provided there is a configured
  implementation for that transport (see `Extensibility`_ below) . The default
  is a list with one string, "tcp".
* transport_waittime (``uint32_t``) - This is the amount of time (in
  milliseconds) to wait for transport connection attempts to complete before
  preferring TCP. The default is 50.
* forwarder (``const char *``) - This is the specific server against which
  names are resolved. If NULL, then this API attempts to use the
  platform-configured name servers (e.g. "/etc/resolv.conf" on POSIX systems).
  Otherwise, this is expected to be the address:port of the forwarding server
  (defaulting to port 53 is **NOT** assumed in this case)
* resolve_via_tcp (``bool``) - This indicates whether to prefer TCP over UDP
  when performing DNS queries.  This setting allows the user to always bypass
  UDP if the results are regularly expected to be truncated.  TCP is always
  used as a fallback if UDP results in a truncated result.  The default
  is false (start with UDP, fallback to TCP).

Operation
~~~~~~~~~

The blinkrez_ctx is used to create the other types via the following functions:

* create_resolver() - Creates a resolver
* create_connector() - Creates a connector

create_resolver() creates a blinkrez_resolver based on this context's
configuration.  If successful, this function returns true and the configured
resolver.  Otherwise it returns false and error information.

::

    bool blinkrez_create_resolver(blinkrez_ctx ctx,
                                  blinkrez_resolver *resolver,
                                  blinkrez_err *err)

create_connetor() creates a blinkrez_connector based on this context's
configuration.  If successful, this function returns true and the configured
connector.  Otherwise it returns false and error information.

::

    bool blinkrez_create_connetor(blinkrez_ctx ctx,
                                  blinkrez_connector *conn,
                                  blinkrez_err *err)

Low-Level Resolver
------------------

The low-level resolver, blinkrez_resolver, performs DNS operations for a given
domain or service name.  Multiple lookups can be pending for the same
blinkrez_resolver instance.

Properties
~~~~~~~~~~

Each of the following properties has a getter, but no setter.  The values are
determined when the blinkrez_resolver is created, or as its state changes while
processing lookups:

* context (``blinkrez_ctx``) - The owning context.
* running (``bool``) - Flag to indicate this resolver has at least one
  outstanding lookup in progress.
  
Operations
~~~~~~~~~~

The blinkrez_resolver provides the following functions:

* lookup() - Initiates a lookup based on type and name
* cancel() - Cancels a pending lookup (if any).

lookup() takes a record type and a name (along with a callback and optional
callback data), and finds all of the associated records. The socket
establishment builds on an instance of this type to actually create a socket,
based on the policies for addressing and transport. For A/AAAA lookups, this
resolves IPv4 and IPv6 addresses, depending on the configuration's allowed
addressing (the {allow_ipv4} and {allow_ipv6} settings, respectively); for
SRV lookups, this further resolves the target names, ordered according to the
priority (and possibly weight); other types will simply return the record
data.  CNAMEs are automatically followed when encountered.

::

    bool blinkrez_resolver_lookup(int type,
                                  const char *name,
                                  blinkrez_lookup_cb cb,
                                  void *arg,
                                  blinkrez_handle *handle,
                                  blinkrez_errcode *err)

The type is the integer RR type value, and can be either 29 (A + AAAA) or 33
(SRV). Future versions of this API may support other RR types. Note that A
and AAAA are **not** separately allowed here.

The name is the string to resolve.  For A/AAAA lookups, it is the
fully-qualified domain name (e.g. "example.com"); for SRV lookups, it is the
combination of the service name, service protocol, and domain name (e.g.
"_xmpp-client._tcp.example.com").

The cb is the callback to execute when a record is found, or a non-recoverable
error is encountered.  This callback is executed once for each individual
record, and once more after all records have been reported.  For example,
a lookup of A/AAAA for "example.com" will result in the callback executing
three times, once for the A record result, once for the AAAA record result,
and once to indicate the lookup is complete.

The arg is the user-provided callback data, and is passed to the callback
each time it is executed.

The handle is returned by lookup() to identify a pending lookup operation,
and used by cancel() to terminate that operation.  This value is an opaque
key used by blinkrez_resolver, and has no semantic meaning outside of that
instance.

lookup() returns false and error information if the provided data is invalid,
or memory has been exhausted.  Otherwise, it returns true and a handle.
Further success or failure is indicated via the callback.

cancel()
!!!!!!!!

cancel() takes handle returned by lookup(), and terminates the outstanding
lookup (if any).  If handle is NULL, then all outstanding operations are
terminated.  Each terminated operation will execute the associated callback
with a BLINKREZ_ERR_CANCELED error code.

::

    void blinkrez_resolver_cancel(blinkrez_handle handle)

Callback
~~~~~~~~

The lookup() callback is expected to match the following signature::

    void (*blinkrez_resolver_lookup_cb)(blinkrez_lookup_handle handle,
                                        blinkrez_err_code retcode,
                                        struct blinkrez_lookup_result *result,
                                        void *arg);

This callback is executed for each found record, and when the lookup() is
complete (successful or failed).

The handle indicates the lookup() request this callback is associated with.

The retcode indicates the status of the lookup():
    
* ``BLINKREZ_ERR_NONE`` if the lookup completed successfully
* ``BLINKREZ_ERR_CONTINUE`` if more results are expected
* ``BLINKREZ_ERR_CANCELED`` if the lookup was canceled by the user
* ``BLINKREZ_ERR_NOT_FOUND`` if name and type could not be resolved
* ``BLINKREZ_ERR_NO_MEM`` if an out-of-memory condition was reached

The blinkrez_lookup_result is a structure describing the resolved record:

* name (``const char *``) - The name resolved against.
* type (``int``) - The type of record resolved.
* address (``struct sockaddr_storage *``) - The resolved address; may be NULL
  if the result is an empty record
* verified (``bool``) - Indicates the chain of records is signed and
  verified, via DNSSEC (OPEN ISSUE: does this accept for the AD flag from a
  recursive name server, or must every record be verified separately?)

The value of result is undefined if retcode is **not** BLINKREZ_ERR_CONTINUE.

Socket Connector
----------------

The socket connector, blinkrez_connector, builds upon the low-level resolver and
policy context to establish a best-case socket connection from a name.  Like
the resolver, the socket connector can have multiple operations running at
a time.

Properties
~~~~~~~~~~

Each of the following properties have a getter, but no setter.  The values are
determined when the blinkrez_connector is created, or as its state changes while
processing lookups:

* context (``blinkrez_ctx``) - The owning context.
* running (``bool``) - Flag to indicate this connector has at least one
  outstanding operation in progress.

Operations
~~~~~~~~~~

The blinkrez_connector provides the following functions:

* connect() - Initiates a connection attempt.
* cancel() - Terminates an outstanding connect (if any).

connect() takes a record type (A/AAAA, SRV), a name, port, and (optional)
initial data and establishes a socket connection.  The established socket is
determined by the addressing and transport agility algorithms specified below.
For SRV-based operations, only the transport specified by the service protocol
portion of the name (e.g. "tcp" for "_xmpp-client._tcp.example.com") is used.

::

    bool blinkrez_connector_connect(int type,
                                    const char *name,
                                    uint16_t port,
                                    struct evbuffer *initdata,
                                    blinkrez_connector_cb cb,
                                    void *arg,
                                    blinkrez_handle handle,
                                    blinkrez_errcode *err);

The type is the integer RR type value, and can be either 29 (A + AAAA) or 33
(SRV). Note that A and AAAA are **not** separately allowed here.

The name is the string to resolve.  For A/AAAA lookups, it is the
fully-qualified domain name (e.g. "example.com"); for SRV lookups, it is the
combination of the service name, server protocol, and domain name (e.g.
"_xmpp-client._tcp.example.com").

The port is used directly for A/AAAA-based operations, or as a fallback for
SRV-based operations.

The (optional) initdata is used as part of establishing the socket connection.
If provided, the transport sends this data as part of finalizing the
connection. This can result in important optimizations for some transports,
such as SCTP.

The handle is returned by connect() to identify a pending connection operation,
and used by cancel() to terminate that operation.  This value is an opaque
key used by blinkrez_connector, and has no semantic meaning outside of the API.

connect() returns false and error information if the provided data is invalid,
or memory has been exhausted.  Otherwise, it returns true and a handle.
Further success or failure is indicated via the callback.

cancel() takes the handle returned by connect(), and terminates the
outstanding lookup (if any).  If handle is NULL, then all outstanding operations
are terminated.  Each terminated operation will execute its associated callback
with a BLINKREZ_ERR_CANCELED error code.

Callback
~~~~~~~~

The connect() callback is expected to match the following signature::

    void (*blinkrez_connector_lookup_cb)(blinkrez_lookup_handle handle,
                                         blinkrez_err_code retcode,
                                         struct blinkrez_connect_result *result,
                                         void *arg);

This callback is executed when connect() completes (successful or failed).

The handle indicates the connect() request this callback is associated with.

The retcode indicates the status of the connect():
    
* ``BLINKREZ_ERR_NONE`` if the connect completed successfully
* ``BLINKREZ_ERR_CANCELED`` if the connect was canceled by the user
* ``BLINKREZ_ERR_NOT_FOUND`` if name and type could not be resolved
* ``BLINKREZ_ERR_SOCKET`` if a socket error was encountered, and could not be
  recovered (e.g. failed to connect to any candidate)
* ``BLINKREZ_ERR_NO_MEM`` if an out-of-memory condition was reached

The result is a structure describing the connection:

* transport (``const char *``) - The transport name used to establish the
  connection
* socket (``evutil_socket_t``) - The socket handle/file descriptor
* address (``struct sockaddr_storage``) - The resolved address
* initdata (``struct evbuffer *``) - Received initial data, can be NULL and/or
  an empty buffer.  If this value is not NULL, the listener SHOULD consume
  this data first, before processing the socket's recv buffer.
* verified (``bool``) - Indicates the chain of records is signed and
  verified, via DNSSEC (OPEN ISSUE: does this accept for the AD flag from a
  recursive name server, or must every record be verified separately?)

The value of result is undefined if retcode is **not** BLINKREZ_ERR_NONE.

Addresses vs. Names
~~~~~~~~~~~~~~~~~~~

For simplicity, the blinkrez_connector will not reject IP addresses (e.g.
"192.168.0.24" or "[fe80:0:0:0:200:f8ff:fe21:67cf]") when performing
A/AAAA-based operations.  Instead, the blinkrez_connector will bypass the normal
lookup operations and attempt to establish a socket based on the transports
appropriate to the address.

Polling/Selecting
-----------------

This API will expose some of its libevent internals in order to grant the user
enough control to properly monitor its activity.  At a minimum, there will be a
getter for the event_base object in use.  The actual logic to block until
input/output is complete will not be provided by this API.

There may be some concerns around resource locking, as the libevent dispatching
will most likely take place on one thread while the calls to lookup and connect
happen on others.  We may rely on libevent's locking mechanisms here, and
require the user to properly configure them.  The blinkrez_dns functions will
call libevent's lock/unlock functions as appropriate, and against the specific
structure the blinkrez_dns is using (the current event/bufferevent is
recommended).

Addressing Agility
------------------

This API will follow the recommended approach documented in `Happy Eyeballs
(IPv6)`_ to support IPv4 and IPv6.  This algorithm is applied if IPv4 *and*
IPv6 addressing is allowed; if either is disabled, then connections will only
be made using the one allowed.

The simplified approach is as follows:

0) Start with the following parameters:

   * Service to lookup (e.g. "_xmpp-client._tcp.example.com")
   * Integer value P, which is biased toward IPv6 (P > 0) or IPv4 (P < 0), or
     neither (P == 0) (initially set as {ip_preference} in the settings).

1) start lookup A and AAAA records (in that order)

   * If P<0, delay reporting the AAAA lookup by abs(P * 10) milliseconds
   * If P>0, delay reporting A lookup by abs(P * 10) milliseconds

2) For each reported result, attempt connection immediately; this step is
   skipped for DNS lookups without connection attempts.

3) Adjust P for future lookups (only if both A and AAAA records are reported)

    3.1) If P>0 ...

         3.1.1) If winning lookup is IPv6, P = P + 1
         
         3.1.2) If winning lookup is IPv4, P = P / 2

    3.2) If P<0

	     3.2.1) If winning lookup is IPv6, P = P / 2
	     
	     3.2.2) If winning lookup is IPv4, P = P - 1

    3.3) If P=0

         3.3.1) If winning lookup is IPv6, P = P + 1
         
         3.3.2) If winning lookup is IPv4, P = P - 1

Transport Agility
-----------------

This API approximately follows the recommended approach documented in
`Happy Eyeballs (SCTP)`_ to support various transport protocols.  This
algorithm is applied if there are multiple transports enabled in the settings;
if there is only one listed, then that is the transport protocol used for all
connection attempts.

This algorithm is applied on top of the addressing agility algorithm; once an
address is resolved (either IPv4 or IPv6), this set of 

The simplified approach is as follows:

0) Start with the following parameters:

   * Address to connect to (e.g. resolved from "example.com")
   * Integer value SWAIT, which is the number of milliseconds to wait for all
     transport connection attempts (initially set as {transport_waittime} in
     the settings).
     
1) For each transport, attempt a connection

   * If the details for establishing a connection for a transport is not
     understood (see `Extensibility`_ below), it is skipped.  The configuration
     MAY be adjusted to remove this transport from the list.

2) First established connection to complete within SWAIT wins

   * If the transport is "tcp", it is ignored unless it is the only
     transport to complete.
   * The specific transport is noted for the connected address; the next
     connection attempt SHOULD use this transport.

Extensibility
-------------

Transports
~~~~~~~~~~

Support for additional transport protocols is provided by registering a set of
callback functions against a transport name.  When the API determines it needs
to establish a connection, it will look in the registry of transports, and use
the callbacks if it finds a mapping.  There will be a default implementation
for "tcp".

Ideally, the transport functions work with something that can map to
``evutil_socket_t``, and is something libevent can select/poll against.

<< More to be determined >>

Testability
-----------

To aid with testability, the API can take the address of a specific name server
to use via the {forwarder} setting.  This name server should be one that is
easily controlled, and can be used in automated environments.  A possible
example is `dnsmasq <http://www.thekelleys.org.uk/dnsmasq/doc.html>`_.

.. _Happy Eyeballs (IPv6): http://tools.ietf.org/html/draft-ietf-v6ops-happy-eyeballs
.. _Happy Eyeballs (SCTP): http://tools.ietf.org/html/draft-wing-tsvwg-happy-eyeballs-sctp
