Dumpulse: an embeddable dumb heartbeat daemon in 260 bytes of RAM and ≈350 bytes of code
========================================================================================

![](diagram.png)

Dumpulse aggregates network monitoring information in extremely
constrained environments; it can handle up to a few million heartbeat
messages per second, returning the latest heartbeat value from each
sender upon request in a 260-byte health-report message.  It needs
about 3% of the RAM found in a traditional Arduino like the
Duemilanove, and it processes each heartbeat message in about 256
instruction executions (about 30 nanoseconds on a 1.6GHz N3700).  It’s designed
to work reliably even in unreliable environments.

Dumpulse is 200–400 bytes of code and 75–200 machine instructions.

Quick start
-----------

Getting it running and trying it out; basically any Unix with a
C compiler and a recent Python 2 or 3 should work:

    $ git clone …dumpulse
    Cloning into 'dumpulse'...
    done.
    $ cd dumpulse/
    $ make
    cc -fPIC -g -Os -Wall -std=c89 -o dumpulse.o -c dumpulse.c
    cc -g -Os -Wall -std=c89   -c -o udpserver.o udpserver.c
    cc   udpserver.o dumpulse.o   -o udpserver
    cc -fPIC -g -Os -Wall -std=c89 -o dumpulse_so.o -c dumpulse_so.c
    cc -shared dumpulse_so.o dumpulse.o -o dumpulse.so
    $ ./udpserver 9060 &
    [1] 10560
    $ Waiting for UDPv4 packets on port 9060.

    $ python client.py localhost 9060
    Health report of 260 bytes:
    checksum 01000001 checks OK
    v0 = 0 at 0 from 0
    v1 = 0 at 0 from 0
    v2 = 0 at 0 from 0
    …
    v63 = 0 at 0 from 0
    $ python client.py localhost 9060 -v 45  # set a value
    $ python client.py localhost 9060
    Health report of 260 bytes:
    checksum 7fb80181 checks OK
    v0 = 45 at 14543 from 76
    v1 = 0 at 0 from 0
    v2 = 0 at 0 from 0
    …
    v63 = 0 at 0 from 0
    $ python client.py localhost 9060 -v 46 -n 1
    $ python client.py localhost 9060
    Health report of 260 bytes:
    checksum 005c0309 checks OK
    v0 = 45 at 14543 from 76
    v1 = 46 at 14550 from 76
    v2 = 0 at 0 from 0
    …
    v63 = 0 at 0 from 0
    $ make test
    py.test-3 test.py
    ============================= test session starts ==============================
    platform linux -- Python 3.5.2, pytest-2.8.7, py-1.4.31, pluggy-0.3.1
    rootdir: /home/user/Downloads/dumpulse, inifile: 
    plugins: hypothesis-3.0.1
    collected 1 items

    test.py .

    =========================== 1 passed in 5.42 seconds ===========================


Alternatively, to not install the test prerequisites outside of
Docker:

    $ make dockertest
    docker build -t dumpulse .
    Sending build context to Docker daemon 609.3 kB
    Step 1 : FROM python:3.6
     ---> 01fd71a97c19
    Step 2 : RUN pip install pytest hypothesis
     ---> Using cache
     ---> ea9af75c12eb
    Step 3 : ADD . /dumpulse
     ---> 132cf83efdc1
    Removing intermediate container 7989f2797eb5
    Step 4 : WORKDIR /dumpulse
     ---> Running in ebf0ca8f7659
     ---> d9c5fcef4fb0
    Removing intermediate container ebf0ca8f7659
    Successfully built d9c5fcef4fb0
    docker run dumpulse py.test test.py
    ============================= test session starts ==============================
    platform linux -- Python 3.6.3, pytest-3.2.3, py-1.4.34, pluggy-0.4.0
    rootdir: /dumpulse, inifile:
    plugins: hypothesis-3.32.2
    collected 1 item

    test.py .

    =========================== 1 passed in 5.42 seconds ===========================

What? Why? What is this good for?
---------------------------------

The problem this solves is the following: you have a distributed
system of microcontrollers that talk to each other over a
network — for example, the CAN network of an automobile — and you
want to monitor the health of all of its subsystems over a
much-lower-bandwidth link, such as a nominally 8kbps Mobitex link.

It is desirable in such a situation to have some kind of
heartbeat-aggregation agent instead of communicating individually
with each subsystem, for the following reasons:

1. The heartbeat-aggregation agent can send a single packet reporting
   the health of the whole system, rather than one packet per
   microcontroller.  If there are, say, 64 microcontrollers, and the
   per-packet overhead is 64 bytes’ worth (roughly correct for Mobitex
   MPAK and link-layer overhead), the entire health report will
   require at least 4096 bytes, which is probably about 8 seconds of
   continuous Mobitex transmission.  By contrast, a single packet
   containing, say, 4 bytes of status data per service will be only
   320 bytes including the overhead, which works out to 640
   milliseconds of airtime, more than an order of magnitude
   improvement.

2. A particular microcontroller may be failed at the time that the
   remote monitoring system attempts to conduct the health check.
   But, to diagnose the problem, it may be desirable to determine with
   more precision when the failure happened.  A heartbeat-aggregation
   agent can store the last-known-good heartbeat information from the
   failed component and report it on request.

3. Any network can lose or corrupt data being transmitted, but
   high-bandwidth, low-latency controller-area networks are many
   orders of magnitude more reliable than low-bandwidth, high-latency
   radio networks, and the higher rate of retransmissions that is
   feasible over the faster networks works to reduce the impact of
   isolated packet loss or corruption.  A remote monitoring system
   that successfully gets a response from a heartbeat-aggregation
   agent knows that any missing data is missing because of a fault
   local to the controller-area network, not due to a radio-network
   failure.  By contrast, if the remote monitoring system expects to
   receive a status report from each controller over the radio
   network, a missing status report may be due to a subsystem failure
   or merely a lost over-the-air packet.

However, the heartbeat-aggregation agent is itself a potential single
point of failure in the monitoring system, so it’s desirable to keep
it as simple and robust as possible and to run it on a reliable
platform.  For example, if you run it from Flash ROM on a bare
microcontroller, only hardware failures or failures in the agent
software itself can cause it to fail; if, instead, you run it on
Linux, then kernel memory corruption, system overload, filesystem
corruption, bootloader failure, and the like all come into play.

It’s also important for the agent to have little or no configuration
of its own, because any configuration data is a potential point of
failure, and if it’s baked into the firmware for a processor that also
does other things, updating the configuration data poses the risk of
breaking whatever else you’re running on that processor.

So this program, Dumpulse, implements a simple protocol suitable for
aggregating heartbeat data sent over a physically secured network like
I²C or CAN, running on bare metal or under a minimal real-time
operating system like eCos, FreeRTOS, or RTEMS, although it also works
on Linux.  It provides no authentication and only minimal data
integrity checking.  All the packets except the 260-byte health report
are 8 bytes, so they can be sent as single CAN bus frames,
although the example Linux implementation uses UDP.  It takes no
configuration data.

Calling interface
-----------------

There are two calling interfaces: the embedding interface provided by
`dumpulse.o`, intended for actual use, and the dynamic-linking
interface provided by `dumpulse.so`, which wraps it and is used for
testing.

### Embedding interface ###

As documented in `dumpulse.h`, `dumpulse.o`
exposes one data type, `dumpulse`, which can be allocated
where you like and needs no finalization but should be initialized to
all zero bytes, and one function:

    uint8_t dumpulse_process_packet(dumpulse *p, char *data, void *context);

It returns nonzero if it successfully processed the packet and zero if
it did not, but generally speaking you don’t need to care.

For embedding, it requires you to provide two handler functions:

    uint16_t dumpulse_get_timestamp(void);
    void dumpulse_send_packet(void *context, char *data, size_t len);

If it is necessary to send a reply packet, the `context` provided to
`dumpulse_process_packet` will be provided to your
`dumpulse_send_packet` function before `dumpulse_process_packet`
returns.  You can use it, for example, to store the address to which
to send the reply packet.  Because it’s only used before
`dumpulse_process_packet` returns, it is okay for it to be a
stack-allocated data structure.  `dumpulse_send_packet` should make a
copy of the data it is provided, or transmit it over the network,
before returning; just saving a pointer to it is not okay, because
that data will be mutated when the next heartbeat packet arrives.

Because `dumpulse_send_packet` has no return value, Dumpulse cannot
tell if the response packet was successfully sent or not, and so it
always considers a health report request message to have been
successfully handled.

Dumpulse does not allocate memory and can only fail in the sense of
not being able to process a packet because it is ill-formed.

There is an example of hooking it up to UDP on Linux in `udpserver.c`.

### Dynamic-linking interface ###

A more modern, but less efficient, dynamically-linkable interface is
provided in `dumpulse_so.c`, mostly for testing.  This requires you to
define a `dumpulse_so` struct with your function pointers and invoke
the `dumpulse_process_packet_so` function with it.  At least with GCC,
it uses thread-local storage so that you should be able to
simultaneously invoke `dumpulse_process_packet_so` with different
handlers on different threads.

A `server.py` is provided which uses the dynamically-linkable
interface via the standard, if awkward, Python `ctypes` module.  This
is used for property-based generative testing in `test.py`.

Protocol
--------

![(diagram)](heartbeat.png)

The heartbeat message consists of 8 bytes: a four-byte big-endian
Adler-32 checksum of the other four bytes, a single byte with the
constant value 241, a single-byte variable ID identifying the variable
to update (from 0 to 63), a byte indicating the ostensible identity of
the packet sender, and a final byte containing the value of the
variable, in that order with no padding.

![(diagram)](health-report.png)

The health report request message is the fixed 8 ASCII bytes
“AreyouOK”, and will be responded to with a 260-byte response
consisting of a four-byte big-endian Adler-32 of the rest of the
message, followed by 64 four-byte triples (16-bit timestamp, sender
ID, value).

The 16-bit timestamp has a resolution determined by the application;
there’s a tradeoff between wraparound time and precision.  Taking the
low 16 bits of a `time_t` is adequate for many applications, giving a
wraparound time of 18.2 hours and a precision of 1 second.

Performance and code weight
---------------------------

For details on performance measurements, see `performance.md`.

Testing and prerequisites
-------------------------

Dumpulse is somewhat exhaustively tested using the Hypothesis
property-based generative testing framework.

The only prerequisite for compiling Dumpulse itself is an ANSI C
compiler and Make.  To build the example UDP server, you probably
need some kind of Unix.  client.py and server.py require a
relatively recent Python, either 2 or 3.

test.py, and the `test` target in the Makefile, additionally requires
Hypothesis and py.test.  This is used to feed a few thousand random
packets to the server and verify its behavior against a Python model,
and it found an off-by-one bounds-checking bug when I introduced it.

I’ve hacked together a Dockerfile that installs Hypothesis and py.test
inside a Docker container to run the tests inside of.

Reliability and security
------------------------

This protocol is not designed to be operated in an environment, such
as a LAN or the public internet, where malicious actors can inject
packets.  Anyone who can send crafted packets to Dumpulse can do the
following:

- overwrite the whole heartbeat table with arbitrary data;
- cause Dumpulse to send many times more data than they are themselves
  sending;
- cause Dumpulse to send almost arbitrary data in the health report,
  limited only by timestamp control and the minor restrictions of
  Adler-32.

However, they cannot do the following, unless it has bugs:

- crash Dumpulse;
- cause Dumpulse to use a large amount of CPU time.

However, it should be sufficiently tolerant of random or corrupted
data that you can run it over a noisy broadcast network without a
separate addressing layer underneath (something such as UDP which
ensures it only receives data intended for it).  It will spuriously
recognize roughly one in 2³² random 8-byte packets.

Provenance
----------

The ideas came out of discussions with Lucio Torre, Alejandro Cura,
Nicolás Demarchi, Ezequiel Alfíe, and others,
but Kragen Javier Sitaker wrote Dumpulse, so the
deficiencies of the design and implementation are all his.

Dumpulse is copyright Ⓒ 2017 Satellogic.

<script src="http://canonical.org/~kragen/sw/addtoc.js">
</script>

<style>
img {float: right; clear: right; margin-left: 1em}
h1, h2, h3, h4, h5, h6 { clear: right }
</style>