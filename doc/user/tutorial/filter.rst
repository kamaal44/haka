.. This Source Code Form is subject to the terms of the Mozilla Public
.. License, v. 2.0. If a copy of the MPL was not distributed with this
.. file, You can obtain one at http://mozilla.org/MPL/2.0/.

Filter
======

Introduction
------------
Haka is a tool which can filter packets and streams, based on any fields or
combination of fields in the packet or the streams.

How-to
------
This tutorial is divided into two parts. The first one relies on hakapcap
tool and pcap file, in order to show how to do basic filtering.
The second one uses the nfqueue interface in order to alter tcp streams.

Basic TCP/IP filtering
----------------------

Basic IP filtering
^^^^^^^^^^^^^^^^^^
In order to do some basic IP filtering you can read the self-documented
lua script file:

.. literalinclude:: ../../../sample/filter/ipfilter.lua
   :language: lua
   :tab-width: 4

A pcap file is provided in order to run the above lua script file

.. code-block:: console

    $ cd <haka_install_path>/share/haka/sample/filter/
    $ hakapcap ipfilter.pcap ipfilter.lua

You can save the pcap in an output file in order to see the
modification or deletion of packets:

.. code-block:: console

    $ cd <haka_install_path>/share/haka/sample/filter/
    $ hakapcap trace.pcap ipfilter.lua -o output.pcap

Interactive rule debugging
^^^^^^^^^^^^^^^^^^^^^^^^^^
Hereafter, a second lua script file allowing to filter tcp packets based
on destination port.

.. literalinclude:: ../../../sample/filter/tcpfilter.lua
   :language: lua
   :tab-width: 4

If we try to launch this script, we will notice that it will fail to
run successfully. Actually, we deliberately introduced an error into the code.
Haka will raise an error. More precisely, there is no field named ``destport``.

As shown in section :doc:`\../debug`, Haka is featured with debugging capabilities allowing
to get more details about errors. The debugger mode is available through the ``--luadebug`` option:

.. code-block:: console

    $ cd <haka_install_path>/share/haka/sample/filter/
    $ hakapcap tcpfilter.pcap tcpfilter.lua --luadebug

When the debugger starts, it automatically breaks at the first lua line:

.. ansi-block::
    :string_escape:

    \x1b[0m\x1b[1minfo\x1b[0m  \x1b[36mdebugger:\x1b[0m \x1b[0mlua debugger activated
    \x1b[0m\x1b[31m   5\x1b[1m=> \x1b[0mrequire('protocol/ipv4')
    \x1b[32mdebug\x1b[1m>


At the debug prompt, type ``continue`` to immediately break into the erroneous
code. When a lua error code occurs, the debugger breaks and outputs the error
and a backtrace.

.. ansi-block::
    :string_escape:

    \x1b[32mdebug\x1b[1m>  \x1b[0mcontinue
    ...
    \x1b[0m\x1b[32mentering debugger\x1b[0m: unknown field 'destport'
    Backtrace
    \x1b[31m\x1b[1m=>\x1b[0m0    \x1b[36m[C]\x1b[0m: in function '\x1b[35m(null)\x1b[0m'
     #1    \x1b[36m[C]\x1b[0m: in function '\x1b[35m(null)\x1b[0m'
     #2    \x1b[36m[C]\x1b[0m: in function '\x1b[35m__index\x1b[0m'
     #3    \x1b[36mtcpfilter.lua:21\x1b[0m: in function '\x1b[35meval\x1b[0m'
     #4    \x1b[36m/opt/haka/share/haka/core/rule.bc:0\x1b[0m: in the main chunk
     #5    \x1b[36m/opt/haka/share/haka/core/rule.bc:0\x1b[0m: in the main chunk
     #6    \x1b[36m/opt/haka/share/haka/core/rule.bc:0\x1b[0m: in the main chunk
     #7    \x1b[36m[C]\x1b[0m: in function '\x1b[35mxpcall\x1b[0m'
     #8    \x1b[36m/opt/haka/share/haka/core/rule.bc:0\x1b[0m: in the main chunk
    ...

As we are interested in debugging lua code, we skip the first frames and jump
directly to the frame #3 (type ``frame 3``). To get the line number that
generated the error, we simply list the lua source code using the ``list``
command.

.. ansi-block::
    :string_escape:

    \x1b[32mdebug\x1b[1m>  \x1b[0mlist
    \x1b[33m  16:  \x1b[0mhooks = { 'tcp-up' },
    \x1b[33m  17:  \x1b[0m    eval = function (self, pkt)
    \x1b[33m  18:  \x1b[0m        -- The next line will generate a lua error:
    \x1b[33m  19:  \x1b[0m        -- there is no 'destport' field. replace 'destport' by 'dstport'
    \x1b[31m  20\x1b[1m=> \x1b[0m        if pkt.destport == 80 or pkt.srcport == 80 then
    \x1b[33m  21:  \x1b[0m            haka.log("Filter", "Authorizing trafic on port 80")
    \x1b[33m  22:  \x1b[0m        else
    \x1b[33m  23:  \x1b[0m            haka.log("Filter", "Trafic not authorized on port %d", pkt.dstport)
    \x1b[33m  24:  \x1b[0m            pkt:drop()
    \x1b[33m  25:  \x1b[0m        end

Printing the `pkt` table (``print pkt``) will show that we misspelled the ``dstport`` field.

.. ansi-block::
    :string_escape:

    \x1b[32mdebug\x1b[1m>  \x1b[0mprint pkt
      #1  \x1b[34;1muserdata\x1b[0m tcp {
              ...
            \x1b[34;1mchecksum\x1b[0m : 417
            \x1b[34;1mres\x1b[0m : 0
            \x1b[34;1mnext_dissector\x1b[0m : \x1b[35;1m"tcp-connection"\x1b[0m
            \x1b[34;1msrcport\x1b[0m : 37542
            \x1b[34;1mpayload\x1b[0m : \x1b[35;1muserdata\x1b[0m tcp_payload
            \x1b[34;1mip\x1b[0m : \x1b[36;1muserdata\x1b[0m ipv4 {
              ...
            }
            \x1b[34;1mflags\x1b[0m : \x1b[36;1muserdata\x1b[0m tcp_flags {
              ...
            }
            \x1b[34;1mack_seq\x1b[0m : 0
            \x1b[34;1mseq\x1b[0m : 3827050607
            \x1b[34;1mdstport\x1b[0m : 80
            \x1b[34;1mhdr_len\x1b[0m : 40
        }

Press CTRL-C to quit or type ``help`` to get the list of available commands.

Using the rule group
^^^^^^^^^^^^^^^^^^^^

.. literalinclude:: ../../../sample/filter/groupfilter.lua
    :language: lua
    :tab-width: 4

Advanced TCP/IP Filtering
-------------------------

Filtering with NFQueue
^^^^^^^^^^^^^^^^^^^^^^
The lua configuration files can be used with nfqueue and haka daemon.
Haka will hook itself at startup to the raw nfqueue table in order to
inspect, modify, create and delete packets in real time.

For the rest of this tutorial, we will assume that you've installed
haka package on a host with an interface named eth0.

This is the configuration of the daemon:

.. literalinclude:: ../../../sample/filter/daemon.conf
   :language: ini
   :tab-width: 4

In order to start haka, you have to be root. The ``--no-daemon`` option
won't send haka daemon on background. Plus, all logs messages are
printed on output, instead of syslogd.

.. code-block:: console

   # cd <haka_install_path>/share/haka/sample/filter/
   # haka -c daemon.conf --no-daemon

The filtering will be done according to the .lua configuration file seen
previously.

HTTP filtering
^^^^^^^^^^^^^^
You can filter through all HTTP fields thanks to http module:

.. literalinclude:: ../../../sample/filter/httpfilter.lua
   :language: lua
   :tab-width: 4

Modify the ``dameon.conf`` in order to load the ``httpfilter.lua``
configuration file:

.. code-block:: ini

    [general]
    # Select the haka configuration file to use.
    configuration = "httpfilter.lua"

And start it

.. code-block:: console

   # cd <haka_install_path>/share/haka/sample/filter/
   # haka -c dameon.conf --no-daemon


Modifying HTTP responses
^^^^^^^^^^^^^^^^^^^^^^^^
Haka can also alter data sent by webserver. We want to be able to
filter all obsolete Web browser based on the User-Agent header.
More, we want to force these obsolete browsers to go only to update websites.
Haka will check User-Agent, and if the User-Agent is considered
obsolete, it will change HTTP response to redirect request to a safer
site (web site of the browser).

.. literalinclude:: ../../../sample/filter/httpmodif.lua
   :language: lua
   :tab-width: 4

Modify the ``dameon.conf`` in order to load the ``httpmodif.lua``
configuration file:

.. code-block:: ini

    [general]
    # Select the haka configuration file to use
    configuration = "httpmodif.lua"

And start it

.. code-block:: console

   # cd <haka_install_path>/share/haka/sample/filter/
   # haka -c dameon.conf --no-daemon