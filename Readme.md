# Companion Application Blueprint

The **companion application** is an example to
showcase how to create a vehicle application which senses and actuates signals in the vehicle
for Eclipse Leda with the help of Eclipse Velocitas and Eclipse Kuksa
. The aim is not to build the best available application possible but to show how one can use the applied technologies
to build a companion application for the interaction with a vehicle, e.g., to move a seat.
If you are new to the concepts around Eclipse SDV and the mentioned projects
we recommend to read the [SDV Tutorial](https://eclipse-leda.github.io/leda/docs/general-usage/sdv-introduction/) first.

## Description

The idea of the specific companion application introduced in this blueprint is to move the driver seat to positions defined
in a driver profile hosted by a third-party web service.

![Leda Seat Adjuster Use Case](./img/seatadjuster.png)

The setup contains the following components:

- Cloud or mobile trigger: not part of the Leda image, but we simulate it by issuing MQTT messages
- **Seat Adjuster** : Developed with Eclipse Velocitas to be deployed by user
- Eclipse Kuksa.val - **KUKSA Databroker** (pre-installed with Eclipse Leda)
- **Mock Service**: Example provider for Eclipse Kuksa.VAL which mocks the behavior of the vehicle.

In the following pages, we first introduce the [architecture and the assumed data flow](./architecture-seat-adjuster.md) in more detail.

We then show how to realize the introduced architecture with Eclipse Velocitas and Eclipse Leda 0.1.0-M2. On a high level, you will perform the following steps and we describe each step in more detail in this guide. So simply [continue reading](./architecture-seat-adjuster.md):

1. [Setup the Eclipse Velocitas template repository](./develop-seat-adjuster.md#setup-eclipse-velocitas-from-template-repository) to develop, build and deploy your
version of the seat adjuster example in a VSCode DevContainer.

2. [Run Eclipse Leda](./deploy-seat-adjuster.md), for example, as container or with other options like QEMU, physical hardware, etc.

3. [Manage the Eclipse Kanto container runtime](./deploy-seat-adjuster.md) in Eclipse Leda to deploy your seat adjuster application.

4. [Test the deployed setup](./interact-seat-adjuster.md) in Eclipse Leda by interacting with the seat adjuster over MQTT to change the seat position.

If you run into any issues while following this guide, you can check the [troubleshooting page](./troubleshooting.md) for possible hints.
