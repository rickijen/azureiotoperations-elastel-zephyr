# Connecting IIoT Gateway to Azure IoT Operations

<p align="center">
<img title="" src="https://github.com/rickijen/azureiotoperations-elastel-zephyr/blob/main/artifacts/media/elastel-main.jpg?raw=true" alt="main-pic" width="350" data-align="center">
</p>

An Industrial IoT (IIoT) solution deployed at the edge location often integrates microcontroller-based devices, sensors, and PLCs into industrial processes to gather data, monitor operations, and improve efficiency.  The goal is to enable real-time monitoring, predictive maintenance, automation, and data-driven decision-making in industries such as manufacturing, building automation, energy, transportation, and agriculture.

IIoT gateway is a critical component in the solution to aggregate data collected from industry standard protocols (such as [Modbus](https://en.wikipedia.org/wiki/Modbus), [BACnet](https://en.wikipedia.org/wiki/BACnet), [OPC UA](https://en.wikipedia.org/wiki/OPC_Unified_Architecture), [Sparkplug](https://sparkplug.eclipse.org/), etc.). The IIoT gateway then processes and routes data to the cloud.

[Azure IoT Operations](https://learn.microsoft.com/azure/iot-operations/overview-iot-operations) features an enterprise-grade MQTT broker that is deployed locally in an Arc-enabled Kubernetes cluster installed at the edge site. With proper [Data Flows](https://learn.microsoft.com/azure/iot-operations/connect-to-cloud/overview-dataflow) configured, data gathered from the IIoT gateway can be delivered to the cloud. Control commands can be delivered from cloud to devices as well.

## Overview

![Architecture](https://github.com/rickijen/azureiotoperations-elastel-zephyr/blob/main/artifacts/media/architecture.png?raw=true)

This tutorial demonstrates a real-world scenario where a MCU-based device (an ARM Cortex-M4 based ***STM32F429ZI-Nucleo*** from [STMicroelectronics](https://www.st.com/content/st_com/en.html)) publishes MQTT messages to the [Elastel EG324 IIoT Gateway](https://www.elastel.com/products/iot-gateway/eg324-iot-gateway/). The IIoT Gateway, which then routes data to the AIO MQTT broker which then delivers data to the cloud via Data Flow. [Zephyr](https://docs.zephyrproject.org/latest/develop/getting_started/index.html) is the RTOS running on the STM32 MCU device. In this tutorial, we will build a Zephyr app publishing MQTT messages upstream from the device. To validate the data delivered to Azure IoT Operations instance, we will use a MQTTX client subscribing to the topic and confirm messages are delivered correctly. Finally, we will configure Data Flow and examine the data in Azure Data Explorer cluster database.

Here are the MCU device STM32 Nucleo and Elastel EG324 IIoT Gateway configured in this tutorial:

<p align="center">
<img title="" src="https://github.com/rickijen/azureiotoperations-elastel-zephyr/blob/main/artifacts/media/stm32elastel.jpg?raw=true" alt="photo" width="385">
</p>

## Prerequisites

- An [Azure Arc-enabled Kubernetes](https://learn.microsoft.com/azure/azure-arc/kubernetes/overview) cluster (such as [K3s]([K3s](https://k3s.io/))) is already installed locally at the edge. Follow this [doc](https://learn.microsoft.com/iot-operations/deploy-iot-ops/howto-prepare-cluster?tabs=ubuntu) to make sure your cluster is prepared to install IoT Operations.

- Install Elastel EG324 IIoT Gateway and completed the steps in [Getting Started | Elastel Docs Center](https://docs.elastel.com/docs/ElastPro/Getting_Started)

- MCU-based development boards (from Nordic, ST, NXP, etc.). If you are interested in the one I have, order from here: [NUCLEO-F439ZI STMicroelectronics | Mouser](https://www.mouser.com/ProductDetail/STMicroelectronics/NUCLEO-F439ZI?qs=w%2Fv1CP2dgqohyY3gLxBfCw%3D%3D&mgh=1&srsltid=AfmBOooGrMnVJmcE8iWlq3w1uEqN0iw2XGkakoi75SCoOK9dGK_cnOg5U3s)

- Prepare Zephyer SDK toolchains and its Python build environment by following: [Getting Started Guide — Zephyr Project Documentation](https://docs.zephyrproject.org/latest/develop/getting_started/index.html) 

## Install Azure IoT Operations

1. Make sure you have the latest Azure CLI extension.
   
   ```bash
   az extension add --upgrade --name azure-iot-ops
   ```

2. Deploy to the Kubernetes cluster (example: for my single node cluster).
   
   ```bash
   az iot ops create --subscription $SUBSCRIPTION_ID \
   -g iot --cluster $CLUSTER_NAME --custom-location $CUSTOM_LOCATION \
   -n $AIO_INSTANCE_NAME --resource-id /subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG_NAME/providers/Microsoft.DeviceRegistry/schemaRegistries/schema-registry-redondo \
   --broker-frontend-replicas 1 --broker-frontend-workers 1 \
   --broker-backend-part 1 --broker-backend-workers 1 \
   --broker-backend-rf 2 --broker-mem-profile Low \
   --ops-config observability.metrics.openTelemetryCollectorAddress=aio-otel-collector.azure-iot-operations.svc.cluster.local:4317 \
   --ops-config observability.metrics.exportInternalSeconds=60
   ```

3. Verify IoT Ops service deployment output for health, configuration, and usability. If you have installed IoT Ops previously, make sure to **check for upgrade** first.
   
   ```bash
   az extension add --upgrade --name azure-iot-ops
   az iot ops check
   ```
   
   <p align="center">
   <img src="https://github.com/rickijen/azureiotoperations-elastel-zephyr/blob/main/artifacts/media/iot-ops-check.png?raw=true" title="" alt="asdc" width="404">
   </p>

4. In order to secure the MQTT bridge between IIoT Gateway and AIO MQTT Broker, prepare **server and client certificates** to be installed on the AIO MQTT broker and IIoT Gateway by following: [Tutorial: Azure IoT Operations MQTT broker TLS, X.509 client authentication, and ABAC - Azure IoT Operations | Microsoft Learn](https://learn.microsoft.com/azure/iot-operations/manage-mqtt-broker/tutorial-tls-x509)

5. Create a new AIO MQTT Broker Load Balancer listener (with the server certificate created in the previous step) on **port 8883** with **X509-auth**: [Secure MQTT broker communication by using BrokerListener - Azure IoT Operations | Microsoft Learn](https://learn.microsoft.com/azure/iot-operations/manage-mqtt-broker/howto-configure-brokerlistener?tabs=portal%2Ctest)

At this point, the AIO MQTT Broker is ready.

## Install Elastel EG324 IIoT Gateway

**WARNING** *This is an industrial-grade equipment. When used in a residential environment, make sure you follow expert guidance on how to properly supply 12V DC power to the unit to avoid damage/hazards. In my lab, I have properly mounted the gateway on a DIN rail attached to the server rack with the NVVV EDR-120W-12V Industrial DIN rail power supply as shown in the photo at the beginning of this tutorial.*

1. Go to the web portal of EG324 and configure networking: [WAN | Elastel Docs Center](https://docs.elastel.com/docs/ElastPro/Network/WAN#wired-ethernet-settings)

2. *Optional*: When LoRaWAN is appropriate for your environment, follow: [LoRaWAN | Elastel Docs Center](https://docs.elastel.com/docs/ElastPro/Network/LoRaWAN)

3. Configure MQTT setting in: [Reporting Center | Elastel Docs Center](https://docs.elastel.com/docs/ElastPro/Data_Collect/North_Apps/Reporting_Center/#mqtt-protocol)

4. This IIoT Gateway has a built-in **Mosquitto MQTT broker**, so optionally, you can bring in your own configuration file. When you completed the basic broker configuration, move on to the next section. Refer to the manual for Mosquitto: [mosquitto.conf man page | Eclipse Mosquitto](https://mosquitto.org/man/mosquitto-conf-5.html)

## Connect IIoT Gateway to AIO via MQTT with TLS

1. Download the server and client certificates (generated in previous steps) to EG324 Gateway and use the following configuration as a reference, customize it to your environment. The purpose of this section is to securely bridge the Mosquitto broker of EG324 to the local MQTT broker of AIO. With the provided server and client certificates, we can establish the MQTT bridge over mTLS.
   
   For example, you will need to change the address of AIO broker and place proper references to certificates:
   
   | Variable        | Certificate type                               |
   | --------------- | ---------------------------------------------- |
   | bridge_cafile   | The CA certificate installed on the AIO broker |
   | bridge_certfile | The client certificate of Mosquitto            |
   | bridge_keyfile  | The private key of client certificate          |
   
   ```bash
   # Place your local configuration in /etc/mosquitto/conf.d/
   #
   # A full description of the configuration file is at
   # /usr/share/doc/mosquitto/examples/mosquitto.conf.example
   
   pid_file /var/run/mosquitto.pid
   
   persistence true
   persistence_location /var/lib/mosquitto/
   
   log_dest file /var/log/mosquitto/mosquitto.log
   
   include_dir /etc/mosquitto/conf.d
   
   # Bridge connection
   connection cloud-01
   address 10.1.0.5:8883
   bridge_cafile /etc/mosquitto/ca_certificates/contoso_root_ca.crt
   bridge_certfile /etc/mosquitto/certs/thermostat.crt
   bridge_keyfile /etc/mosquitto/certs/thermostat.key
   topic # out 0
   topic # in 0
   #remote_username nucleo
   #remote_password Redondo123Redondo123
   bridge_protocol_version mqttv311
   try_private false
   notifications false
   bridge_attempt_unsubscribe false
   bridge_insecure true
   ```

2. Restart the service to securely bridge the two MQTT brokers
   
   ```bash
   systemctl restart mosquitto
   ```

3. Examine the Kubernetes logs of AIO MQTT broker and confirm it's securely bridged with EG324 Gateway:
   
   ```bash
   sudo kubectl logs aio-broker-frontend-0 -c broker -n azure-iot-operations | grep Elast
   ```
   
   From the output, we can see the MQTT connection established between gateway and AIO:
   
   ```bash
   <6>2025-02-08T17:49:55.248Z aio-broker-frontend-0 [mq@311 tid="28"] - accepting new MQTT client connection: 'ElastPro.cloud-01', tag: '4', clean start: false
   <6>2025-02-08T17:49:55.253Z aio-broker-frontend-0 [mq@311 tid="28"] - new MQTT client connection accepted: 'ElastPro.cloud-01', tag: '4', with credentials that expire at 2025-04-23T02:17:44+00:00
   ```

## Build Zephyr app and publish MQTT to IIoT Gateway

Now that all the plumbing is completed after **securely bridging the two MQTT brokers over mTLS**, we are ready to build the client app running on the MCU device.

1. Open the root folder of your Zephyr project, activate the Python Virtual Environment from the terminal.

2. Git clone the Zephyr demo app and copy the code to the **sample** directory under the Zephyr project
   
   ```sh
   git clone https://github.com/rickijen/zephyr-dhcp-mqtt
   # And copy the code under Zephyr sample dir, for example, mine is:
   # C:\Users\rijen\zephyrproject\zephyr\samples\net\mqtt_publisher
   ```

3. Build the project with the Zephyr util **West**, or if you have already configured proper VS Code IDE extensions, you can simply build directly from VS Code. Update the file **prj.conf** with the correct IP address of your IIoT Gateway before running west build.
   
   ```bash
   CONFIG_NET_CONFIG_PEER_IPV4_ADDR="192.168.1.105"
   ```
   
   Next, verify the current directory and activate the Python VENV for the Zephyr build:
   
   ```sh
   pwd
   
   Path
   ----
   C:\Users\rijen\zephyrproject\zephyr
   ```
   
   ```bash
   ~\zephyrproject\.venv\Scripts\Activate.ps1
   ```
   
   Next, execute the west command with your board target. For example, the board target for my STM32 is **nucleo_f429zi**. Since I have built this app in the past, the pristine build option '**auto**' will not perform a clean build. If you're building it for the first time, you will observe a clean build. 
   
   ```bash
   west build -b nucleo_f429zi .\samples\net\mqtt_publisher --build-dir mqtt_publisher --p auto
   ```
   
   Here's the output of the build:
   
   ```bash
   [1/13] Generating include/generated/zephyr/version.h
   -- Zephyr version: 4.0.99 (C:/Users/rijen/zephyrproject/zephyr), build: v4.0.0-4184-ga253fe27c9f1
   [13/13] Linking C executable zephyr\zephyr.elf
   Memory region         Used Size  Region Size  %age Used
              FLASH:      177584 B         2 MB      8.47%
                RAM:       71016 B       192 KB     36.12%
                CCM:          0 GB        64 KB      0.00%
           IDT_LIST:          0 GB        32 KB      0.00%
   Generating files from C:/Users/rijen/zephyrproject/zephyr/mqtt_publisher/zephyr/zephyr.elf for board: nucleo_f429zi
   ```

4. Once build is completed, flash the binary to the board.
   
   ```bash
   west flash --build-dir mqtt_publisher
   ```
   
   And here's the expected output of download and flash:
   
   ```bash
   -- west flash: rebuilding
   ninja: no work to do.
   -- west flash: using runner stm32cubeprogrammer
         -------------------------------------------------------------------
                          STM32CubeProgrammer v2.18.0
         -------------------------------------------------------------------
   
   ST-LINK SN  : 066AFF333837424757174644
   ST-LINK FW  : V2J39M27
   Board       : NUCLEO-F429ZI
   Voltage     : 3.24V
   SWD freq    : 4000 KHz
   Connect mode: Under Reset
   Reset mode  : Hardware reset
   Device ID   : 0x419
   Revision ID : Rev 5/B
   Device name : STM32F42xxx/F43xxx
   Flash size  : 2 MBytes
   Device type : MCU
   Device CPU  : Cortex-M4
   BL Version  : 0x91
   
   Opening and parsing file: zephyr.hex
   Opening and parsing file: zephyr.hex
   Opening and parsing file: zephyr.hex
   
   Memory Programming ...
   Memory Programming ...
     File          : zephyr.hex
     File          : zephyr.hex
     Size          : 173.42 KB
     Address       : 0x08000000
   
   Erasing memory corresponding to segment 0:
   Erasing internal memory sectors [0 5]
   Download in Progress:
   ██████████████████████████████████████████████████ 100%
   File download complete
   Time elapsed during download operation: 00:00:04.253
   RUNNING Program ...
     Address:      : 0x8000000
   Application is running, Please Hold on...
   Start operation achieved successfully
   ```

5. From the serial console of the STM32 MCU device, we can see the MQTT messages are delivered to EG324 Gateway. You will see the device acquiring a DHCP address and performing MQTT CONNECT & MQTT PUBLISH. You will also see mqtt_ping to keep the session alive.
   
   ```bash
   [00:00:08.453,000] <inf> net_mqtt_publisher_sample:    Address[1]: 192.168.1.104
   [00:00:08.453,000] <inf> net_mqtt_publisher_sample:     Subnet[1]: 255.255.255.0
   [00:00:08.453,000] <inf> net_mqtt_publisher_sample:     Router[1]: 192.168.1.1
   [00:00:08.453,000] <inf> net_mqtt_publisher_sample: Lease time[1]: 172800 seconds
   [00:00:08.453,000] <inf> net_config: IPv4 address: 192.168.1.104
   [00:00:08.453,000] <inf> net_config: Lease time: 172800 seconds
   [00:00:08.453,000] <inf> net_config: Subnet: 255.255.255.0
   [00:00:08.453,000] <inf> net_config: Router: 192.168.1.1
   [00:00:11.817,000] <inf> net_mqtt_publisher_sample: attempting to connect: 
   [00:00:11.818,000] <dbg> net_sock: zsock_socket_internal: (main): socket: ctx=0x20002f20, fd=5
   [00:00:11.819,000] <inf> net_mqtt: Connect completed
   [00:00:11.821,000] <dbg> net_sock: zsock_received_cb: (rx_q[0]): ctx=0x20002f20, pkt=0x2000f9c8, st=0, user_data=0
   [00:00:11.821,000] <inf> net_mqtt_publisher_sample: MQTT client connected!
   [00:00:11.821,000] <inf> net_mqtt_publisher_sample: try_to_connect: 0 <OK>
   [00:00:11.821,000] <inf> net_mqtt_publisher_sample: mqtt_ping: 0 <OK>
   [00:00:11.822,000] <dbg> net_sock: zsock_received_cb: (rx_q[0]): ctx=0x20002f20, pkt=0x2000f9c8, st=0, user_data=0
   [00:00:11.822,000] <inf> net_mqtt_publisher_sample: PINGRESP packet
   [00:00:12.321,000] <inf> net_mqtt_publisher_sample: mqtt_publish: 0 <OK>
   [00:00:12.821,000] <inf> net_mqtt_publisher_sample: mqtt_publish: 0 <OK>
   [00:00:12.822,000] <dbg> net_sock: zsock_received_cb: (rx_q[0]): ctx=0x20002f20, pkt=0x2000f9c8, st=0, user_data=0
   [00:00:12.822,000] <inf> net_mqtt_publisher_sample: PUBACK packet id: 13567
   ```

## Verify MQTT from IIoT Gateway to AIO MQTT broker

1. Install your favorite MQTT client, or download [MQTTX: Your All-in-one MQTT Client Toolbox](https://mqttx.app/)

2. Use the same set of server and client certificates to connect and subscribe to the topic **sensors/#** on AIO MQTT broker to confirm messages are delivered from EG324 gateway to AIO broker.
   
   Here's the screen of mTLS configuration of MQTTX:
   
   <p align="center">
   <img src="https://github.com/rickijen/azureiotoperations-elastel-zephyr/blob/main/artifacts/media/mqttx-config.png?raw=true" title="" alt="s" width="485">
   </p>
   
   And here's the screen of subscribing to the topic "**sensors**":
   
   <p align="center">
   <img src="https://github.com/rickijen/azureiotoperations-elastel-zephyr/blob/main/artifacts/media/mqttx.png?raw=true" title="" alt="s" width="485">
   </p>

## Configure AIO Data Flow Endpoints & Data Flows

Now that we have the data collected from sensors, let's configure AIO data flow endpoints and route data via Data Flows. We will leverage **Azure Data Explorer** as our cloud destination.

1. Create an Azure Data Explore Cluster and a database (mapped to the JSON fields of your MQTT data from MCU sensors) in the cluster. Then, assign AIO's user assigned or system assigned managed identity to the database with the RBAC role "**Database Ingestor**". Once that's completed, we will be able to configure the AIO data flow endpoint for Data Explorer. For example, my user assigned managed identity for AIO is "iot-ops-cloud-connections" and my database permission of my data explorer cluster "redondoadx" looks like:
   
   <p align="center">
   <img src="https://github.com/rickijen/azureiotoperations-elastel-zephyr/blob/main/artifacts/media/dx-db-perm.png?raw=true" title="" alt="s" width="485">
   </p>

2. Next, in your AIO Operations Portal, create a new **data flow endpoint** with references to your Data Explorer Cluster. For example, my cluster is "redondoadx":
   
   <p align="center">
   <img src="https://github.com/rickijen/azureiotoperations-elastel-zephyr/blob/main/artifacts/media/aio-data-flow-endpoint.png?raw=true" title="" alt="s" width="485">
   </p>

3. Next, create a new **data flow** with the source as message broker.
   
   <p align="center">
   <img src="https://github.com/rickijen/azureiotoperations-elastel-zephyr/blob/main/artifacts/media/data-flow-source.png?raw=true" title="" alt="s" width="485">
   </p>

4. After adding the destination data flow endpoint created earlier, the data flow should look like:
   
   <p align="center">
   <img src="https://github.com/rickijen/azureiotoperations-elastel-zephyr/blob/main/artifacts/media/data-flow.png?raw=true" title="" alt="s" width="485">
   </p>

## Query the data in the Data Explorer Database

You may want to add a few more sensor telemetry fields from the MCU device so we can query with more meaningful data. For example, besides temperature, I also added humidity and pressure.

To verify data, execute the KQL query command in the Data Explorer

```sql
SensorTelemetry
| where isnotnull(temperature)
| take 100
| project temperature, humidity, pressure
```

You will find the output as:
    <p align="center">
   <img src="https://github.com/rickijen/azureiotoperations-elastel-zephyr/blob/main/artifacts/media/kql.png?raw=true" title="" alt="s" width="485">
   </p>

**Congratulations!** At this point, you have completed the end-to-end data ingestion flow: MCU device -> IIoT Gateway -> AIO -> Azure Data Explorer.

## Conclusion

As mentioned in the beginning of tutorial, this scenario is a very common method of establishing data communication between IIoT sensors and the cloud via an IIoT Gateway. Be aware that you do have the option to establish MQTT session over TLS end-to-end: from MCU device to IIoT Gateway and finally AIO MQTT. But we also need to consider the MCU devices are typically resource-constrained, therefore performing crypto operations on MCU devices will increase the binary footprint and impact performance. In this tutorial, we opted to only secure the MQTT bridge between gateway and AIO, but do discuss with your team to decide the trade-off for your project.

## Resources

- [Deployment overview - Azure IoT Operations | Microsoft Learn](https://learn.microsoft.com/azure/iot-operations/deploy-iot-ops/overview-deploy)

- [EG324 IIoT Gateway, Arm-based industrial Computer - Elastel](https://www.elastel.com/products/iot-gateway/eg324-iot-gateway/)

- [Zephyr demo app](https://github.com/rickijen/zephyr-dhcp-mqtt)
