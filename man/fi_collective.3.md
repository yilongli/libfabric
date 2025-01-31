---
layout: page
title: fi_collective(3)
tagline: Libfabric Programmer's Manual
---
{% include JB/setup %}

# NAME

fi_join_collective
: Operation where a subset of peers join a new collective group.

fi_barrier
: Collective operation that does not complete until all peers have entered
  the barrier call.

fi_broadcast
: A single sender transmits data to all receiver peers.

fi_allreduce
: Collective operation where all peers broadcast an atomic operation to all
  other peers.

fi_reduce_scatter
: Collective call where data is collected from all peers and merged (reduced).
  The results of the reduction is distributed back to the peers, with each
  peer receiving a slice of the results.

fi_alltoall
: Each peer distributes a slice of its local data to all peers.

fi_allgather
: Each peer sends a complete copy of its local data to all peers.

fi_query_collective
: Returns information about which collective operations are supported by a
  provider, and limitations on the collective.

# SYNOPSIS

```c
#include <rdma/fi_collective.h>

int fi_join_collective(struct fid_ep *ep, fi_addr_t coll_addr,
	const struct fid_av_set *set,
	uint64_t flags, struct fid_mc **mc, void *context);

ssize_t fi_barrier(struct fid_ep *ep, fi_addr_t coll_addr,
	void *context);

ssize_t fi_broadcast(struct fid_ep *ep, void *buf, size_t count, void *desc,
	fi_addr_t coll_addr, enum fi_datatype datatype, enum fi_op op,
	uint64_t flags, void *context);

ssize_t fi_allreduce(struct fid_ep *ep, const void *buf, size_t count,
	void *desc, void *result, void *result_desc,
	fi_addr_t coll_addr, enum fi_datatype datatype, enum fi_op op,
	uint64_t flags, void *context);

ssize_t fi_reduce_scatter(struct fid_ep *ep, const void *buf, size_t count,
	void *desc, void *result, void *result_desc,
	fi_addr_t coll_addr, enum fi_datatype datatype, enum fi_op op,
	uint64_t flags, void *context);

ssize_t fi_alltoall(struct fid_ep *ep, const void *buf, size_t count,
	void *desc, void *result, void *result_desc,
	fi_addr_t coll_addr, enum fi_datatype datatype,
	uint64_t flags, void *context);

ssize_t fi_allgather(struct fid_ep *ep, const void *buf, size_t count,
	void *desc, void *result, void *result_desc,
	fi_addr_t coll_addr, enum fi_datatype datatype,
	uint64_t flags, void *context);

int fi_query_collective(struct fid_domain *domain,
	enum fi_datatype datatype, enum fi_op op,
	struct fi_collective_attr *attr, uint64_t flags);
```

# ARGUMENTS

*ep*
: Fabric endpoint on which to initiate collective operation.

*set*
: Address vector set defining the collective membership.

*mc*
: Multicast group associated with the collective.

*buf*
: Local data buffer that specifies first operand of collective operation

*datatype*
: Datatype associated with atomic operands

*op*
: Atomic operation to perform

*result*
: Local data buffer to store the result of the collective operation.

*desc / result_desc*
: Data descriptor associated with the local data buffer
  and local result buffer, respectively.

*coll_addr*
: Address referring to the collective group of endpoints.

*flags*
: Additional flags to apply for the atomic operation

*context*
: User specified pointer to associate with the operation.  This parameter is
  ignored if the operation will not generate a successful completion, unless
  an op flag specifies the context parameter be used for required input.

# DESCRIPTION

In general collective operations can be thought of as coordinated atomic
operations between a set of peer endpoints.  Readers should refer to the
[`fi_atomic`(3)](fi_atomic.3.html) man page for details on the
atomic operations and datatypes defined by libfabric.

A collective operation is a group communication exchange.  It involves
multiple peers exchanging data with other peers participating in the
collective call.  Collective operations require close coordination by all
participating members.  All participants must invoke the same collective call
before any single member can complete its operation locally.  As a result,
collective calls can strain the fabric, as well as local and remote data
buffers.

Libfabric collective interfaces target fabrics that support offloading
portions of the collective communication into network switches, NICs, and
other devices.  However, no implementation requirement is placed on the
provider.

The first step in using a collective call is identifying the peer endpoints
that will participate.  Collective membership follows one of two models, both
supported by libfabric.  In the first model, the application manages the
membership.  This usually means that the application is performing a
collective operation itself using point to point communication to identify
the members who will participate.  Additionally, the application may be
interacting with a fabric resource manager to reserve network resources
needed to execute collective operations.  In this model, the application will
inform libfabric that the membership has already been established.

A separate model moves the membership management under libfabric and directly
into the provider.  In this model, the application must identify which peer
addresses will be members.  That information is conveyed to the libfabric
provider, which is then responsible for coordinating the creation of the
collective group.  In the provider managed model, the provider will usually
perform the necessary collective operation to establish the communication
group and interact with any fabric management agents.

In both models, the collective membership is communicated to the provider by
creating and configuring an address vector set (AV set).  An AV set
represents an ordered subset of addresses in an address vector (AV).
Details on creating and configuring an AV set are available in
[`fi_av_set`(3)](fi_av_set.3.html).

Once an AV set has been programmed with the collective membership
information, an endpoint is joined to the set.  This uses the fi_join_collective
operation and operates asynchronously.  This differs from how an endpoint is
associated synchronously with an AV using the fi_ep_bind() call.  Upon
completion of the fi_join_collective operation, an fi_addr is provided that
is used as the target address when invoking a collective operation.

For developer convenience, a set of collective APIs are defined.  However,
these are inline wrappers around the atomic interfaces.  Collective APIs
differ from message and RMA interfaces in that the format of the data is
known to the provider, and the collective may perform an operation on that
data.  This aligns collective operations closely with the atomic interfaces.

## Join Collective (fi_join_collective)

This call attaches an endpoint to a collective membership group.  Libfabric
treats collective members as a multicast group, and the fi_join_collective
call attaches the endpoint to that multicast group.  By default, the endpoint
will join the group based on the data transfer capabilities of the endpoint.
For example, if the endpoint has been configured to both send and receive data,
then the endpoint will be able to initiate and receive transfers to and from
the collective.  The input flags may be used to restrict access to the
collective group, subject to endpoint capability limitations.

Join collective operations complete asynchronously, and may involve fabric
transfers, dependent on the provider implementation.  An endpoint must be bound
to an event queue prior to calling fi_join_collective.  The result of the join
operation will be reported to the EQ as an FI_JOIN_COMPLETE event.  Applications
cannot issue collective transfers until receiving notification that the join
operation has completed.  Note that an endpoint may begin receiving
messages from the collective group as soon as the join completes, which can
occur prior to the FI_JOIN_COMPLETE event being generated.

The join collective operation is itself a collective operation.  All
participating peers must call fi_join_collective before any individual peer
will report that the join has completed.  Application managed collective
memberships are an exception.  With application managed memberships, the
fi_join_collective call may be completed locally without fabric communication.
For provider managed memberships, the join collective call requires as
input a coll_addr that refers to either an address associated with an
AV set (see fi_av_set_addr) or an existing collective group (obtained through
a previous call to fi_join_collective).  The
fi_join_collective call will create a new collective subgroup.
If application managed memberships are used, coll_addr should be set to
FI_ADDR_UNAVAIL.

Applications must call fi_close on the collective group to disconnect the
endpoint from the group.  After a join operation has completed, the
fi_mc_addr call may be used to retrieve the address associated with the
multicast group.  See [`fi_cm`(3)](fi_cm.3.html) for additional details on
fi_mc_addr().

## Barrier (fi_barrier)

The fi_barrier operation provides a mechanism to synchronize peers.  Barrier
does not result in any data being transferred at the application level.  A
barrier does not complete locally until all peers have invoked the barrier
call.  This signifies to the local application that work by peers that
completed prior to them calling barrier has finished.

## Broadcast (fi_broadcast)

fi_broadcast transfers an array of data from a single sender to all other
members of the collective group.  The sender of the broadcast data must
specify the FI_SEND flag, while receivers use the FI_RECV flag.  The input
buf parameter is treated as either the transmit buffer, if FI_SEND is set, or
the receive buffer, if FI_RECV is set.  Either the FI_SEND or FI_RECV flag
must be set.  The broadcast operation acts as an atomic write or read to a
data array.  As a result, the format of the data in buf is specified through
the datatype parameter.  Any non-void datatype may be broadcast.

The following diagram shows an example of broadcast being used to transfer an
array of integers to a group of peers.

```
[1]  [1]  [1]
[5]  [5]  [5]
[9]  [9]  [9]
 |____^    ^
 |_________|
 broadcast
```

## All Reduce (fi_allreduce)

fi_allreduce can be described as all peers providing input into an atomic
operation, with the result copied back to each peer.  Conceptually, this can
be viewed as each peer issuing a multicast atomic operation to all other
peers, fetching the results, and combining them.  The combining of the
results is referred to as the reduction.  The fi_allreduce() operation takes
as input an array of data and the specified atomic operation to perform.  The
results of the reduction are written into the result buffer.

Any non-void datatype may be specified.  Valid atomic operations are listed
below in the fi_query_collective call.  The following diagram shows an
example of an all reduce operation involving summing an array of integers
between three peers.

```
 [1]  [1]  [1]
 [5]  [5]  [5]
 [9]  [9]  [9]
   \   |   /
      sum
   /   |   \
 [3]  [3]  [3]
[15] [15] [15]
[27] [27] [27]
  All Reduce
```

## All to All (fi_alltoall)

The fi_alltoall collective involves distributing (or scattering) different
portions of an array of data to peers.  It is best explained using an
example.  Here three peers perform an all to all collective to exchange
different entries in an integer array.

```
[1]   [2]   [3]
[5]   [6]   [7]
[9]  [10]  [11]
   \   |   /
   All to all
   /   |   \
[1]   [5]   [9]
[2]   [6]  [10]
[3]   [7]  [11]
```

Each peer sends a piece of its data to the other peers.

All to all operations may be performed on any non-void datatype.  However,
all to all does not perform an operation on the data itself, so no operation
is specified.

## Reduce-Scatter (fi_reduce_scatter)

The fi_reduce_scatter collective is similar to an fi_allreduce operation,
followed by all to all.  With reduce scatter, all peers provide input into an
atomic operation, similar to all reduce.  However, rather than the full result
being copied to each peer, each participant receives only a slice of the result.

This is shown by the following example:

```
[1]  [1]  [1]
[5]  [5]  [5]
[9]  [9]  [9]
  \   |   /
     sum (reduce)
      |
     [3]
    [15]
    [27]
      |
   scatter
  /   |   \
[3] [15] [27]
```

The reduce scatter call supports the same datatype and atomic operation as
fi_allreduce.

## All Gather (fi_allgather)

Conceptually, all gather can be viewed as the opposite of the scatter
component from reduce-scatter.  All gather collects data from all peers into
a single array, then copies that array back to each peer.

```
[1]  [5]  [9]
  \   |   /
 All gather
  /   |   \
[1]  [1]  [1]
[5]  [5]  [5]
[9]  [9]  [9]
```

All gather may be performed on any non-void datatype.  However, all gather
does not perform an operation on the data itself, so no operation is
specified.

## Query Collective Attributes (fi_query_collective)

The fi_query_collective call reports which collective operations are
supported by the underlying provider, for suitably configured endpoints.
Collective operations needed by an application that are not supported
by the provider must be implemented by the application.  The query
call checks whether a provider supports a specific collective operation
for a given datatype and operation, if applicable.

The datatype and operation of the collective are provided as input
into fi_query_collective.  For operations that do not exchange
application data, such as fi_barrier, the datatype should be set to
FI_VOID.  The op parameter may reference one of these atomic opcodes:
FI_MIN, FI_MAX, FI_SUM, FI_PROD, FI_LOR, FI_LAND, FI_BOR, FI_BAND,
FI_LXOR, FI_BXOR, or a collective operation: FI_BARRIER, FI_BROADCAST,
FI_ALLTOALL, FI_ALLGATHER.  The use of an atomic opcode will indicate
if the provider supports the fi_allreduce() call for the given
operation and datatype, unless the FI_SCATTER flag has been specified.  If
FI_SCATTER has been set, query will return if the provider supports the
fi_reduce_scatter() call for the given operation and datatype.
Specifying a collective operation for the op parameter queries support
for the corresponding collective.

On success, fi_query_collective will provide information about
the supported limits through the struct fi_collective_attr parameter.

{% highlight c %}
struct fi_collective_attr {
	struct fi_atomic_attr datatype_attr;
	size_t max_members;
	uint64_t mode;
};
{% endhighlight %}

For a description of struct fi_atomic_attr, see
[`fi_atomic`(3)](fi_atomic.3.html).

*datatype_attr.count*
: The maximum number of elements that may be used with the collective.

*datatype.size*
: The size of the datatype as supported by the provider.  Applications
  should validate the size of datatypes that differ based on the platform,
  such as FI_LONG_DOUBLE.

*max_members*
: The maximum number of peers that may participate in a collective
  operation.

*mode*
: This field is reserved and should be 0.

If a collective operation is supported, the query call will return 0,
along with attributes on the limits for using that collective operation
through the provider.

## Completions

Collective operations map to underlying fi_atomic operations.  For a
discussion of atomic completion semantics, see
[`fi_atomic`(3)](fi_atomic.3.html).  The completion, ordering, and
atomicity of collective operations match those defined for point to
point atomic operations.

# FLAGS

The following flags are defined for the specified operations.

*FI_SEND*
: Applies to fi_broadcast() operations.  This indicates that the caller
  is the transmitter of the broadcast data.  There should only be a single
  transmitter for each broadcast collective operation.

*FI_RECV*
: Applies to fi_broadcast() operation.  This indicates that the caller
  is the receiver of broadcase data.

*FI_SCATTER*
: Applies to fi_query_collective.  When set, requests attribute information
  on the reduce-scatter collective operation.

# RETURN VALUE

Returns 0 on success. On error, a negative value corresponding to fabric
errno is returned. Fabric errno values are defined in
`rdma/fi_errno.h`.

# ERRORS

*-FI_EAGAIN*
: See [`fi_msg`(3)](fi_msg.3.html) for a detailed description of handling
  FI_EAGAIN.

*-FI_EOPNOTSUPP*
: The requested atomic operation is not supported on this endpoint.

*-FI_EMSGSIZE*
: The number of collective operations in a single request exceeds that
  supported by the underlying provider.

# NOTES

Collective operations map to atomic operations.  As such, they follow
most of the conventions and restrictions as peer to peer atomic operations.
This includes data atomicity, data alignment, and message ordering
semantics.  See [`fi_atomic`(3)](fi_atomic.3.html) for additional
information on the datatypes and operations defined for atomic and
collective operations.

# SEE ALSO

[`fi_getinfo`(3)](fi_getinfo.3.html),
[`fi_av`(3)](fi_av.3.html),
[`fi_atomic`(3)](fi_atomic.3.html),
[`fi_cm`(3)](fi_cm.3.html)
