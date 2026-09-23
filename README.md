# VOLTTRON Forward Historian

The Forward Historian subscribes to selected message-bus topics on the source
VOLTTRON platform and republishes those messages on the destination platform.
If the destination is temporarily unreachable, messages are retained in the
source platform's local historian cache and forwarded after the connection is
restored.

Forwarded messages retain their original topic and add these headers:

```text
X-Forwarded: True
X-Forwarded-From: <source instance name>
```

## Before You Start

You need two working VOLTTRON platforms. The source platform runs the Forward
Historian. The destination platform receives forwarded messages.

This example uses the fake driver to forward simulated device data. Set up the
fake driver by following the [develop-branch installation instructions](https://github.com/eclipse-volttron/volttron-lib-fake-driver/tree/develop).
For a smoke test without the fake-driver checkout, see the `vctl publish`
example in Step 5.

Once the fake driver is publishing, its device messages are normally below the
`devices/` topic tree. This guide forwards that entire tree.

## 1. Choose the Destination Address

Use TCP when the source and destination platforms run on different hosts. When
both platforms run on the same Linux host, they can connect through the
destination platform's abstract IPC socket. Both transports still use platform
authentication.

### Different Hosts: TCP VIP

For a destination on another host, the destination must listen on a TCP VIP
address. A platform using only its default IPC address cannot receive messages
from another host.

On the destination host, edit the platform config using `nano` or your editor:

```bash
nano YOUR_VOLTTRON_HOME/config
```

Under `[volttron]`, set an externally reachable address. For example:

```ini
[volttron]
address = tcp://0.0.0.0:22916
instance-name = YOUR_DESTINATION_INSTANCE_NAME
messagebus = zmq
auth-enabled = True
```

Verify that it is listening:

```bash
ss -lntp | grep 22916
```

From the source host, verify network access:

```bash
nc -vz YOUR_REMOTE_HOST_IP 22916
```

Do not continue until this succeeds. If it fails, correct the destination
address or firewall first.

<details>
<summary>Same Linux Host: Use the destination IPC socket</summary>

When both platforms run on the same Linux host, no TCP listener or firewall
rule is needed. VOLTTRON binds its local VIP socket as a Linux abstract Unix
socket. Use the destination platform's absolute VOLTTRON_HOME in this address
(including the leading `@`):

```text
ipc://@/YOUR_DESTINATION_VOLTTRON_HOME/run/vip.socket
```

Confirm that the abstract socket is listening after starting the destination
platform. It does not appear as a filesystem socket, so do not use `ls -l`:

```bash
ss -xl | grep -F 'vip.socket'
```

Use the IPC address and the destination platform's public server key in
`forwarder.config`:

```json
{
  "destination-address": "ipc://@/YOUR_DESTINATION_VOLTTRON_HOME/run/vip.socket",
  "destination-serverkey": "paste-the-destination-platform-publickey-here",
  "required_target_agents": [],
  "capture_device_data": true,
  "capture_analysis_data": false,
  "capture_log_data": false,
  "capture_record_data": false,
  "custom_topic_list": [],
  "topic_replace_list": [],
  "cache_only": false
}
```

Retrieve the destination public server key with `vctl auth servercred` as
described in Step 2. Even over IPC, the source and destination platforms have
separate credentials; the forwarder needs the destination's key to authenticate
the server correctly. The source forwarder's public key must also be registered
on the destination in Step 4.

</details>

## 2. Get the Destination Server Key

For a TCP or IPC destination, retrieve the destination platform's public server
key. Skip this step only if the destination does not use authenticated ZMQ
connections.

Run this on the destination host. Set `VOLTTRON_HOME` to the destination
platform's home directory so `vctl` addresses the correct platform. If both
platforms are on the same machine, use the destination's path here; the source
and destination have separate `VOLTTRON_HOME` directories.

```bash
VOLTTRON_HOME=/path/to/DESTINATION_VOLTTRON_HOME vctl auth servercred
```

Copy the returned key. The Forward Historian uses this as its
`destination-serverkey`.

## 3. Configure the Forwarder

On the source host, create a file named `forwarder.config` anywhere convenient.
It does not need to live in the Forward Historian source checkout:

```bash
nano forwarder.config
```

For a TCP destination, paste the following, then replace the destination IP
address and the key copied in Step 2:

```json
{
  "destination-address": "tcp://YOUR_REMOTE_HOST_IP:22916",
  "destination-serverkey": "paste-the-destination-platform-publickey-here",
  "required_target_agents": [],
  "capture_device_data": true,
  "capture_analysis_data": false,
  "capture_log_data": false,
  "capture_record_data": false,
  "custom_topic_list": [],
  "topic_replace_list": [],
  "cache_only": false
}
```

`capture_device_data: true` forwards all messages under `devices/`, including
the messages published by a correctly configured fake driver.

For a same-host IPC destination, use the configuration in the collapsible
section above instead.

Run the following commands on the source host, selecting the source platform
with its `VOLTTRON_HOME`. This is especially important when both platforms run
on the same machine. When installing from a local checkout, pass its path
instead of the package name. The tag makes the agent easy to start without
looking up its generated UUID:

```bash
VOLTTRON_HOME=/path/to/SOURCE_VOLTTRON_HOME \
  vctl install /path/to/volttron-forward-historian \
  --vip-identity platform.forwarder \
  --tag forwarder
```

Add `forwarder.config` to the Forward Historian configuration store:

```bash
VOLTTRON_HOME=/path/to/SOURCE_VOLTTRON_HOME \
  vctl config store platform.forwarder config /path/to/forwarder.config
```

Still on the source host and source `VOLTTRON_HOME`, retrieve the Forward
Historian's public key:

```bash
VOLTTRON_HOME=/path/to/SOURCE_VOLTTRON_HOME \
  vctl auth agentcred platform.forwarder
```

Copy the public key returned for `platform.forwarder`. You can use
`--json` when a script needs structured output:

```bash
VOLTTRON_HOME=/path/to/SOURCE_VOLTTRON_HOME \
  vctl auth agentcred platform.forwarder --json
```

## 4. Authorize the Forwarder on the Destination

Run this on the destination host, selecting the destination platform with its
`VOLTTRON_HOME`. If source and destination are on the same machine, this must
point to the destination's home directory, not the source's.

```bash
VOLTTRON_HOME=/path/to/DESTINATION_VOLTTRON_HOME \
  vctl auth add platform.forwarder \
  --publickey "paste-the-source-forwarder-publickey-here"
```

This stores the remote agent's public credentials without requiring a private
key or manual JSON file creation. If `platform.forwarder` already exists on the
destination, remove the stale record before registering a new key:

```bash
VOLTTRON_HOME=/path/to/DESTINATION_VOLTTRON_HOME \
  vctl auth remove platform.forwarder
VOLTTRON_HOME=/path/to/DESTINATION_VOLTTRON_HOME \
  vctl auth add platform.forwarder \
  --publickey "paste-the-source-forwarder-publickey-here"
```

Confirm the removal when prompted. `vctl auth add` does not replace an existing
credential record.

## 5. Start and Verify

Run these commands on the source host with the source platform's `VOLTTRON_HOME`:

```bash
VOLTTRON_HOME=/path/to/SOURCE_VOLTTRON_HOME vctl start --tag forwarder
```

Check the agent status:

```bash
VOLTTRON_HOME=/path/to/SOURCE_VOLTTRON_HOME vctl status
```

With the fake driver running, publish device data and confirm that the
destination receives the message under the same `devices/` topic with the
`X-Forwarded` and `X-Forwarded-From` headers.

For a smoke test without the fake driver, publish a sample device message from
the source platform:

```bash
VOLTTRON_HOME=/path/to/SOURCE_VOLTTRON_HOME vctl publish \
  devices/campus/building/device/all \
  '{"Temperature":72.5,"Humidity":41}'
```

On the destination platform, subscribe to `devices/` or inspect the platform
logs. `vctl publish` sends its data argument as a string, so this checks topic
routing and forwarding headers; it does not emulate the fake driver's
structured data-and-metadata payload. The received message should include:

```text
X-Forwarded: True
X-Forwarded-From: YOUR_SOURCE_INSTANCE_NAME
```

To follow the platform logs while testing:

```bash
tail -f YOUR_SOURCE_VOLTTRON_HOME/volttron.log
tail -f YOUR_DESTINATION_VOLTTRON_HOME/volttron.log
```

## Tests

Run the focused Forward Historian tests with the local modular VOLTTRON
dependencies installed:

```bash
pytest -q tests/test_forward_historian.py \
  tests/test_forwarder_reconnections.py \
  tests/test_multi_messagebus_forwarder.py
```

The core forwarding tests run without the legacy `PlatformWrapper` fixture.
The optional multi-messagebus test remains skipped until the external
`volttron-testing` fixture is updated for the current `bind_web_address` API
and valid CurveZMQ keys.
