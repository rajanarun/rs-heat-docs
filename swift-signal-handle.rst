===================================
 SwiftSignal and SwiftSignalHandle
===================================

Brief summary
=============

SwiftSingal can be used to coordinate resource creation with
notifications/signals that could be coming from sources external or
internal to the stack. It is often used in conjunction with
SwiftSignalHandle resource.

SwiftSignalHandle is used to create a temporary URL and this URL is used
by applications/scripts to send signals. SwiftSignal resource waits on
this URL for a specified number of signals in given time.

Example template
================

In the following example template, we will set up a single node linux
server that signals success/failure of user_data script
execution at a given URL.

Start by adding the top-level template sections:

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      Single node linux server with swift signaling.

    resources:

    outputs:

Resources section
-----------------

Add a SwiftSignalHandle resource
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

SwiftSignalHandle is a resource to create temporary URL to receive
notification/signals. Note that the temporary URL is created using Rackspace
Cloud Files.

.. code:: yaml

      signal_handle:
        type: "OS::Heat::SwiftSignalHandle"

Add a Server resource
~~~~~~~~~~~~~~~~~~~~~

Add a linux server with a bash script in user_data property. At
the end of the script execution send a success/failure message to the
temporary URL created by the above SwiftSignalHandle resource.

.. code:: yaml

      linux_server:
        type: OS::Nova::Server
        properties:
          image: 4b14a92e-84c8-4770-9245-91ecb8501cc2
          flavor: 1 GB Performance
          user_data:
            str_replace:
              template: |
                #!/bin/bash -x
                # assume you are doing a long running operation here
                sleep 300

                # Assuming long running operation completed successfully,
                # notify success signal
                wc_notify --data-binary '{"status": "SUCCESS"}'

              params:
                # Replace all occurances of "wc_notify" in the script with an
                # appropriate curl PUT request using the "curl_cli" attribute
                # of the SwiftSignalHandle resource
                wc_notify: { get_attr: ['signal_handle', 'curl_cli']


Add SwiftSignal resource
~~~~~~~~~~~~~~~~~~~~~~~~

The SwiftSignal resource waits for specified number of signals (number
is provided as 'count' property) on the given URL ('handle' property).
The stack will be marked as failure if specificed number of signals are
not received in given timeout.

.. code:: yaml

      wait_on_server:
        type: OS::Heat::SwiftSignal
        properties:
          handle: {get_resource: signal_handle}
          count: 1
          timeout: 600

Here SwiftSignal resource would wait for 600 seconds to receive 1 signal
on the handle.

Outputs section
---------------

Add swift signal URL to the outputs section.

.. code:: yaml

      signal_url:
        value: { get_attr: ['wait_handle', 'curl_cli'] }
        description: Swift signal URL

      server_public_ip:
        value:{ get_attr: [ linux_server, accessIPv4 ] }
        description: Linux server public IP

Full Example Template
---------------------

.. code:: yaml

    heat_template_version: 2014-10-16

    description: |
      Single node linux server with swift signaling.

    resources:

      wait_on_server:
        type: OS::Heat::SwiftSignal
        properties:
          handle: {get_resource: signal_handle}
          count: 1
          timeout: 600

      signal_handle:
        type: "OS::Heat::SwiftSignalHandle"

      linux_server:
        type: OS::Nova::Server
        properties:
          image: 4b14a92e-84c8-4770-9245-91ecb8501cc2
          flavor: 1 GB Performance
          user_data:
            str_replace:
              template: |
                #!/bin/bash -x
                # assume you are doing a long running operation here
                sleep 300

                # Assuming long running operation completed successfully, notify success signal
                wc_notify --data-binary '{"status": "SUCCESS"}'

              params:
                wc_notify: { get_attr: ['signal_handle', 'curl_cli'] }

    outputs:
      signal_url:
        value: { get_attr: ['signal_handle', 'curl_cli'] }
        description: Swift signal URL

      server_public_ip:
        value: { get_attr: [ linux_server, accessIPv4 ] }
        description: Linux server public IP

Reference
=========

-  `Cloud Orchestration API Developer
   Guide <http://docs.rackspace.com/orchestration/api/v1/orchestration-devguide/content/overview.html>`__
-  `Heat Orchestration Template (HOT)
   Specification <http://docs.openstack.org/developer/heat/template_guide/hot_spec.html>`__
-  `Cloud-init format
   documentation <http://cloudinit.readthedocs.org/en/latest/topics/format.html>`__
-  `Swift
   TempURL <http://docs.rackspace.com/files/api/v1/cf-devguide/content/TempURL-d1a4450.html>`__
