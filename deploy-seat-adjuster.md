# Deploy Application

We now want to deploy the application to a target device.
You may follow the remainder of this guide on a separate device like a RaspberryPi, but you can emulate such a device on your development machine too.
Either way, we use Eclipse Leda in version 0.1.0-M2 as the target system, which is a Linux-based distribution with pre-installed SDV components like the KUKSA Databroker
and Eclipse Kanto for container management.
For more details on how to download and run Eclipse Leda, follow the respective guides:

- [QEMU](https://eclipse-leda.github.io/leda/docs/general-usage/running-qemu/)
- [Docker](https://eclipse-leda.github.io/leda/docs/general-usage/docker-setup/)
- [RaspberryPi](https://eclipse-leda.github.io/leda/docs/general-usage/raspberry-pi/)
- [Linux](https://eclipse-leda.github.io/leda/docs/general-usage/linux-setup/)

We recommend to get started with the QEMU setup.
In any case, you now need to configure Eclipse Kanto to execute the application.
For this, it helps to get an overview of which containers are currently running in Eclipse Kanto. You can get this through the command:

```bash
kantui
```

or

```bash
kanto-cm list
```

From this list, ensure that at least the KUKSA Databroker runs, which should be the case since it is are pre-configured
with the Eclipse Leda release.

In Eclipse Kanto, you can manage a container with the command line application `kanto-cm` or container manifest files describing a desired container execution.
The advantage of using the container manifests is that the configuration is persisted across a reboot of the system and is easier to use
to describe a desired software state for the overall vehicle.

Eclipse Leda has the `kanto-auto-deployer` systemd service, which applies any changes to the manifests in `/data/var/containers/manifests` to Eclipse Kanto.
Thus, the typical way to add or adapt containers is to modify the corresponding container manifest.

It is important to note that the given file path is relative to the root of the filesystem due to the leading `/`. For instance, from the home directory, in which you are located by default after the login, you need to go two directories upwards (`cd ../..`) to see the `/data` directory.

## Disable other containers

The release 0.1.0-M2 of Eclipse Leda comes with a number of pre-configured and automatically executed containers.
One of these containers is the `feedercan` that feeds changing values from a recording for signals such as `Vehicle.Speed` to the KUKSA Databroker.
These values interfere with the seat adjuster application, which only moves the seat if the vehicle speed is zero.
Another interfering container is the `seatservice-example` which reacts to changes in the signal `Vehicle.Cabin.Seat.Row1.Pos1.Position`
and which we replace later with the `mock service`.

Therefore, we need to stop the `feedercan` and the `seatservice-example` container.
This is possible in `kantui` by selecting the respective entry and pressing `r`. The processing of this input may take a couple of seconds and the command is case-sensitive.
In addition, you need to remove the corresponding container manifests in `/data/var/containers/manifests` to avoid that the Eclipse Kanto
auto-deployer re-deploys these containers. Another approach is to change the ending of the not-needed manifests to something other than `.json`, like `.disabled`.

If the `feedercan` container still runs, the seat adjuster application app will later respond with the following error message:

```json
seatadjuster/setPosition/response
 {"requestId": "12345", "result": {"status": 1, "message": "Not allowed to move seat because vehicle speed is 9.0 and not 0"}}
```

Since they consume resources and are not needed for the seat adjustment, you may remove the containers and manifests for `cloudconnector`,
`hvacservice-example`, `sua`, or `vum` as well.

## Starting of container

The quickest way to start the container is to use the `kanto-cm create` command from the terminal. However, this configuration does not persist across a reboot of your Eclipse Leda instance. We, therefore, recommend that you use the approach described below to add a container manifest. The container manifest is a bit like creating a Pod or Deployment description in Kubernetes.

### Use `kanto-cm`

To start the container in Eclipse Kanto from the terminal, you can use the following command. Before pasting the command to the terminal, replace the placeholders in `<>` with the URL of your seat application container image. One example is:

`ghcr.io/yourGitHubUser/seat-application/seatadjuster:0.0.5`

The full URL for your container is available on GitHub. Select the container under `Packages` on the right side of the `Code` tab view. You should then see the package overview and can check the URL in the installation example behind `docker pull`.

```bash
kanto-cm create --name seatadjuster-app --e="SDV_SEATSERVICE_ADDRESS=grpc://seatservice-example:50051" --e="SDV_MQTT_ADDRESS=mqtt://mosquitto:1883" --e="SDV_VEHICLEDATABROKER_ADDRESS=grpc://databroker:55555" --e="SDV_MIDDLEWARE_TYPE=native" --hosts="databroker:container_databroker-host, mosquitto:host_ip, seatservice-example:container_seatservice-example-host" ghcr.io/<identifier-for-container>:<tag-for-container>

kanto-cm start --name seatadjuster-app
kanto-cm logs --name seatadjuster-app
```

### Add manifest for seat adjuster

As an alternative to using `kanto-cm`, you can add a container manifest to the directoy watched by the `kanto-auto-deployer` (`data/var/containers/manifests`).

To add the container manifest, create a new file inside this folder.

```bash
touch seat-adjuster.json
nano seat-adjuster.json
```

and copy the [manifest from below](#seat-applicationjson). You can save the file with `strg+s` and close the window with `strg+q`.

Inside the manifest file, you need to adapt the reference to your seat application container image by modifying `image.name`. For more details on the URL for the container image, see the paragraph on using `kanto-cm create` above.

You can create the file on the development machine and copy it via scp too:

`scp -P 2222 myapp.json root@localhost:/data/var/containers/manifests/`

The example deployment descriptor below is available in
[meta-leda-components](https://github.com/eclipse-leda/meta-leda/blob/main/meta-leda-components/recipes-sdv/eclipse-leda/kanto-containers/example/seatadjuster-app.json.disabled)
too.
An interesting aspect of the snippet is the `config.env` section at the bottom of the container manifest.
There, we define a number of environment variables for the container
which configures the Eclipse Velocitas SDK to use the native middleware and where to find the MQTT-broker and the KUKSA Databroker to use.
We did the same in `kanto-cm` call behind the parameter `--e=`.

More details on the general deployment approach can be found in [Leda Vehicle Applications](https://eclipse-leda.github.io/leda/docs/app-deployment/velocitas/)

For the deployment to work,  Eclipse Kanto needs access rights to download the referenced container image. If your application repository is `private`, you have to configure Eclipse Kanto with an access token to get the required access rights. More details on how to create and set the token are available in:

- the [troubleshooting page](./troubleshooting.md#you-get-a-401-unauthorized-error-when-trying-to-download-the-container-image-of-your-application) at the end of this guide
- the [Eclipse Leda documentation](https://eclipse-leda.github.io/leda/docs/device-provisioning/container-management/container-registries/).

To make sure that Eclipse Kanto detects the changes in the `manifests` folder, you can restart the respective system services:

```bash
systemctl restart kanto-auto-deployer
```

## Mock Service

As explained in the description of the code, the seat adjuster application sets the target value for the seat positions in the KUSKSA Datbroker
and waits for the current position to update.

For this to function, there needs to be a component that reacts to this change by moving the seat and updating the current value accordingly.
Because we cannot assume that you have an actual ECU availabe for running this guide, we mock the vehicle behavior
with the [vehicle mock service](https://github.com/eclipse/kuksa.val.services/tree/main/mock_service) from the Eclipse Kuksa project.

The mock service allows the definition of custom interaction sequences with the KUKSA Databroker. For instance, one can react to changes to specific signals
or update signals with a time trigger. You can define the sequences in a Python file like the example [`mock.py`](#mockpy) below.
The snippet shows how to use a change to the target value for the seat position signal as a trigger to update the current value to the target value step-wise
over a duration of 10 seconds.

For the deployment, you create another container manifest in `/data/var/containers/manifest` with the content from [below](#mockservicejson).
The container manifest also mounts a custom `mock.py` into the container to replace the configuration of the [default mock.py](https://github.com/eclipse/kuksa.val.services/blob/main/mock_service/mock.py).
With the container manifest below, Eclipse Kanto instead mounts from `/data/var/mock/mock.py`.
Therefore, you need to create this directory and the file with the content from below.

You may now check with `kantui` or `kanto-cm list` whether all components (`databroker`, `seatadjuster-app`, and `mock-service`) are running well.
The next step is to [interact with the seat adjuster](./interact-seat-adjuster.md).

## seat-application.json

This is the Eclipse Kanto container manifest for the seat adjuster application.

```json
{
    "container_id": "seatadjuster-app",
    "container_name": "seatadjuster-app",
    "image": {
        "name": "ghcr.io/<identifier-for-container>:<tag-for-container>"
    },
    "host_config": {
        "devices": [],
        "network_mode": "bridge",
        "privileged": false,
        "restart_policy": {
            "maximum_retry_count": 0,
            "retry_timeout": 0,
            "type": "unless-stopped"
        },
        "runtime": "io.containerd.runc.v2",
        "extra_hosts": [        
                "mosquitto:host_ip",
                "databroker:container_databroker-host",
                "seatservice-example:container_seatservice-example-host"
        ],
        "port_mappings": [
            {
              "protocol": "tcp",
              "container_port": 30151,
              "host_ip": "localhost",
              "host_port": 50151,
              "host_port_end": 50151
            }
        ],
        "log_config": {
            "driver_config": {
                "type": "json-file",
                "max_files": 2,
                "max_size": "1M",
                "root_dir": ""
            },
            "mode_config": {
                "mode": "blocking",
                "max_buffer_size": ""
            }
        },
        "resources": null
    },
    "config": {
        "env": [
           "SDV_SEATSERVICE_ADDRESS=grpc://seatservice-example:50051",
           "SDV_VEHICLEDATABROKER_ADDRESS=grpc://databroker:55555",
           "SDV_MQTT_ADDRESS=mqtt://mosquitto:1883",
           "SDV_MIDDLEWARE_TYPE=native",
           "RUST_LOG=info",
           "vehicle_data_broker=info"
        ],
        "cmd": []
    }
}
```

## mock.py

This is the example `mock.py` for mocking a seat provider:

```python
from lib.animator import RepeatMode
from lib.dsl import (
    create_animation_action,
    create_behavior,
    create_event_trigger,
    create_set_action,
    get_datapoint_value,
    mock_datapoint,
)

from lib.trigger import ClockTrigger, EventType

mock_datapoint(
    path="Vehicle.Cabin.Seat.Row1.Pos1.Position",
    initial_value=0,
    behaviors=[
        create_behavior(
            trigger=create_event_trigger(EventType.ACTUATOR_TARGET),
            action=create_animation_action(
                duration=10.0,
                values=["$self", "$event.value"],
            ),
        )
    ],
)

```

## mockservice.json

The container manifest for the mockservice may look like the following snippet. Note, that the files referenced in the `source` of the `mount_points`
needs to be present in the file system of your Eclipse Leda instance.

```json
{
    "container_id": "mockservice",
    "container_name": "mockservice",
    "image": {
        "name": "ghcr.io/eclipse/kuksa.val.services/mock_service:latest"
    },
    "mount_points": [
        {
            "source": "/data/var/mock/mock.py",
            "destination": "/mock.py",
            "propagation_mode": "rprivate"
        }
    ],
    "host_config": {
        "network_mode": "bridge",
        "privileged": false,
        "restart_policy": {
            "maximum_retry_count": 0,
            "retry_timeout": 0,
            "type": "unless-stopped"
        },
        "runtime": "io.containerd.runc.v2",
        "extra_hosts": [
            "databroker:container_databroker-host"
        ],
        "log_config": {
            "driver_config": {
                "type": "json-file",
                "max_files": 2,
                "max_size": "1M",
                "root_dir": ""
            },
            "mode_config": {
                "mode": "blocking",
                "max_buffer_size": ""
            }
        }
    },
    "config": {
        "env": [
           "VDB_ADDRESS=databroker:55555"
        ]
    }
}
```
