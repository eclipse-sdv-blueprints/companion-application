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

## Disable other containers

The release 0.1.0-M2 of Eclipse Leda comes with a number of pre-configured and automatically executed containers.
One of these containers is the `feedercan` that feeds changing values from a recording for signals such as `Vehicle.Speed` to the KUKSA Databroker.
These values interfere with the seat adjuster application, which only moves the seat if the vehicle speed is zero.
Another interfering container is the `seatservice-example` which reacts to changes in the signal `Vehicle.Cabin.Seat.Row1.Pos1.Position`
and which we replace later with the `mock service`.

Therefore, we need to stop the `feedercan` and the `seatservice-example` container.
This is possible in `kantui` by selecting the respective entry and pressing `R`.
In addition, you need to remove the corresponding container manifests in `/data/var/containers/manifests` of the Eclipse Leda instance to avoid that the Eclipse Kanto
auto-deployer re-deploys these containers. Another approach is to change the ending of the not-needed manifests to something other than `.json`.

If the `feedercan` container still runs, the seat adjuster application app will later respond with the following error message:

```json
seatadjuster/setPosition/response
 {"requestId": "12345", "result": {"status": 1, "message": "Not allowed to move seat because vehicle speed is 9.0 and not 0"}}
```

Since they consume resources and are not needed for the seat adjustment, you may remove the containers and manifests for `cloudconnector`,
`hvacservice-example`, `sua`, or `vum` as well.

## Starting of container

### Use `kanto-cm`

```bash
kanto-cm create \
    --name seatadjuster-app \
    --e="SDV_SEATSERVICE_ADDRESS=grpc://seatservice-example:50051" \
    --e="SDV_MQTT_ADDRESS=mqtt://mosquitto:1883" \
    --e="SDV_VEHICLEDATABROKER_ADDRESS=grpc://databroker:55555" \
    --e="SDV_MIDDLEWARE_TYPE=native" \
    --hosts="databroker:container_databroker-host, mosquitto:host_ip, seatservice-example:container_seatservice-example-host" \
    ghcr.io/<YOUR_ORG>/seat-adjuster-app:latest

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

and copy the example [seatapplication.json](./data/var/containers/manifests/seatapplication.json). You can save the file with `strg+s` and close the window with `strg+q`.
You can create the file on the development machine and copy it via scp too:

`scp -P 2222 myapp.json root@localhost:/data/var/containers/manifests/`

The 'seatadjuster.json' references the container image to be used in line 4:

```json
"image": {
        "name": "ghcr.io/<identifier-for-container>:<tag-for-container>"
    },
```

You need to insert the address of the container image built in GitHub by the Eclipse Velocitas Release pipeline.

Another interesting aspect of the snippet is the `config.env` section at the bottom of the container manifest.
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
or update signals with a time trigger. You can define the sequences in a Python file like the example [`mock.py`](./data/var/mock/mock.py) from this blueprint.
The example shows how to use a change to the target value for the seat position signal as a trigger to update the current value to the target value step-wise
over a duration of 10 seconds:

```python
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

For the deployment, you create another container manifest in `/data/var/containers/manifest` with the content from [mockservice.json](./data/var/containers/manifests/mockservice.json).
The container manifest also mounts a custom `mock.py` into the container to replace the configuration of the [default mock.py](https://github.com/eclipse/kuksa.val.services/blob/main/mock_service/mock.py).
With the referenced container manifest, Eclipse Kanto instead mounts from `/data/var/mock/mock.py`.
Therefore, you need to create this directory and the file from the example [`mock.py`](./data/var/mock/mock.py).

You may now check with `kantui` or `kanto-cm list` whether all components (`databroker`, `seatadjuster-app`, and `mock-service`) are running well.
The next step is to [interact with the seat adjuster](./interact-seat-adjuster.md).
