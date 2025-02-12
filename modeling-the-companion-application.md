# Modeling the companion application

In this chapter, we suggest a way to design and model the application's architecture using the Arcadia methodology, implemented in Eclipse Capella.

## Introduction to Capella

### What is Capella

Capella is an Eclipse project and an open-source MBSE tool and method to sucesfully design system architecture, providing a comprehensive, industrial-grade and customizable solutions to engineering teams worldwide.
For more information about Capella, you can refer to https://mbse-capella.org

Capella embeds the Arcadia methodology, a method devoted to Systems and Architecture Engineering, that supports understanding of customer needs, definition and sharing of system architecture, and early validation and verification. 
It consists of four major perspectives, with their own goals, that provide guidance along the different of the architecture design process, and featuring a clear distinction between need and solution. The four main perspectives of Arcadia are the following:
- Operational Analysis: What the users of the system need to accomplish. In this step, the architect will define the problem by identifying the system users and actors (human and non-human) who will operate with the system, as well as their operational activities and capabilities.
- System Analysis: What the system has to accomplish for its users. This steps focuses on the system to be designed, the formalization of its requirements and the definition of its functions, that will ensure it can fulfill the user needs (defined at the previous step).
- Logical Architecture: How the system will work to fulfill the expectations. In this perspective, architects build a coarse-grained component breakdown of the system carrying most important engineering decisions, and which is unlikely to be challenged later in the development process.
- Physical Architecture: How the system will be developed and built. This steps defines the system's final architecture. After this step, the system is considered ready to develop for subsequent engineering specialties.

For more information about the Arcadia methodology, you can for instance refer to https://mbse-capella.org/arcadia.html.

### Installing and running Capella

To run Capella, go to the download page (https://mbse-capella.org/download.html) and download the zip corresponding to the latest release of Capella.
Once the download is complete, you can simply unzip it to the location of your choice (preferably using 7zip). Then, run capella.exe in the Capella folder.

### Opening the Capella model

To open the Capella model, check first that you have the folder locally saved on your device.
Then, in Capella, go to the File menu and select "Open Projects from File System", then click the Directory button, select the root directory in which the .aird file of the project is contained, and click "Finish".
You can then open the project and show the diagrams by double-clicking the .aird file of the project.


## High-level architecture design with Capella

When starting the development of an application such as this one, you can use the MBSE tool Eclipse Capella to help designing its architecture. In this tutorial, we provide a Capella model to illustrate how the tool, with the Systems Engineering methodology it embeds, can help you to identify needs and define a suitable architecture from there. For more information on Eclipse Capella and how to open the provided model, you can refer to the [dedicated section](./how-to-open-capella-model.md)

### Operational Analysis

The companion application developed in this blueprint is a seat adjuster. The main user need here is to adapt the seat position based on a client's request. In the Arcadia methodology, this user need is captured in the Operational analysis, along with the main actors, i.e. stakeholders of the considered cases.

The Operational Analysis for this tutorial, identifies two Operational entities: A user wanting to adapt the position of the seat, and the vehicle that must react to the user's input. The analysis can be summarized by the Operational Architecture Diagram shown below. Note that, at this stage, the scope of the application is not defined yet, so as to keep it open.

![Operational analysis](./img/operational-architecture-blank.png)

### System Analysis

In the Arcadia methodology, the System Analysis aims at refininf the Operational Analysis, and defining the companion application's boundaries, and the functions it will perform for the stakeholder. At this stage, we focus on what the systmem will do, in a solution-neutral way.

The diagram below from the Capella model represents these features at system level (the full Capella model can be found in folder "Seat adjuster application architecture"). It now introduces the System of interest, here the companion application, as an entity, whereas the Operational Analysis voluntarily leaves it out. It also defines a first, high-level allocation of functions to each entity.

While defining the application's own capabilities in Capella's System Analysis level, we also determine that the application can accept or refuse the request in certain conditions, for instance, while the vehicle is moving. 

The diagram below from the Capella model represents these features at system level (the full Capella model can be found in folder "Seat adjuster application architecture").

![System analysis](./img/system-architecture-blank.png)

### Logical Architecture

After defining the application's functions in the System Analysis perspective of Arcadia, the Logical Architecture perspective helps design and define an architecture that enables the realization of these functions. 
This perspective provides a first break-down of the system and a notional view of components without taking care yet of the technological detail (i.e. we do not yet try to decide what exactly each component will be).

In the Logical Architecture perspective, we give a first break-down of the application's architecture, i.e. of the way it will be constructed, and how the functions will be allocated to each part of the application.
Note that, at this stage of the architecture design, we make the decision to break down the Actor "Vehicle" from the System Analysis into two Logical Actors, "Vehicle Abstraction Layer" and "Vehicle provider". 

![PLogical architecture](./img/logical-architecture-blank.png)

### Physical Architecture

Finally, based on the abstract breakdown realized in the Logical Architecture, we can define the concrete Physical Components that will compose the system (despite their naming, Physical Components can be software as well as physical parts).

The goal of the Physical Architecture is to extensively describe the final solution. It is at this stage that we can decide, for instance, that the "Vehicle Abstraction Layer" defined in the Logical Architecture will be a KUKSA Databroker, or that interface of the application with the user will be composed of MQTT Topics, and will communicate with the client through JSON requests etc.

![Physical architecture](./img/physical-architecture-blank.png)


The seat adjuster application interacts with the vehicle through a *Vehicle Abstraction Layer* created by the KUKSA Databroker,
which uses the [Vehicle Signal Specification (VSS)](https://covesa.github.io/vehicle_signal_specification/)
to express the current value and in case of actuators also the desired state of the vehicle signal.
By developing against the abstraction layer, the application becomes independent from the actual physical seat.

To control the position of the driver seat, the seat adjuster sets the target value of the `Vehicle.Cabin.Seat.Row1.Pos1.Position` signal in the KUKSA Databroker.

The architecture assumes so-called actuation providers that apply the changes to the actual vehicle as indicated in the target value, e.g.,
through interaction with the responsible ECUs.
For this tutorial, we do not expect you to interface with an actual vehicle and thus abstract the vehicle by using the Kuksa.Val vehicle mock service.
This vehicle mock service allows the definition of behavior toward the KUKSA Databroker like we would expect from the vehicle, for example,
setting the current value after reacting to changes to the target value of a signal.

```mermaid
flowchart TB
    client[Client]
    anotherClient[Another Client]
    seatadjuster[Seat Adjuster]
    databroker[(KUKSA<br>Databroker)]
    mqttRequest[[MQTT topic<br>seatadjuster/setPosition/request]]
    mockservice[Provider: Mock Service]

    client -- "JSON Request: position, requestId" --> mqttRequest
    anotherClient -.-> mqttRequest
    mqttRequest --> seatadjuster
    seatadjuster -- "Set Target<br>Vehicle.Cabin.Seat.Row1.Pos1.Position<br>gRPC" --> databroker
    databroker <-- "Set Current<br>Vehicle.Cabin.Seat.Row1.Pos1.Position<br><br>Notify subscriber of changed target<br>Vehicle.Cabin.Seat.Row1.Pos1.Position" --> mockservice
```

As interface to the user, we assume a **client** that can, for example, be a local app in the infotainment domain with a user interface
or an off-board application sending the request from a backend.
Either way, the client controls our seat adjuster application through a JSON encoded message over MQTT using the topic `setPosition/request`.

An example request looks like this:

```bash
mosquitto_pub -t seatadjuster/setPosition/request 
    -m '{"position": 1000, "requestId": "12345"}'
```

The position parameter can be any value between 0 and 1000.




