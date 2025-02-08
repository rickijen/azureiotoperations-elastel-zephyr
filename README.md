# Connecting Industrial IoT Gateway to Azure IoT Operations

<img title="" src="https://github.com/rickijen/azureiotoperations-elastel-zephyr/blob/main/artifacts/media/elastel-main.jpg?raw=true" alt="main-pic" width="196">

An Industrial IoT (IIoT) solution deployed at the edge location often integrates microcontroller-based devices, sensors, and PLCs into industrial processes to gather data, monitor operations, and improve efficiency.  The goal is to enable real-time monitoring, predictive maintenance, automation, and data-driven decision-making in industries such as manufacturing, building automation, energy, transportation, and agriculture.

IIoT gateway is a critical component in the solution to aggregate data via industry standard protocols (such as [Modbus](https://en.wikipedia.org/wiki/Modbus), [BACnet](https://en.wikipedia.org/wiki/BACnet), [OPC UA](https://en.wikipedia.org/wiki/OPC_Unified_Architecture), [Sparkplug](https://sparkplug.eclipse.org/), etc.) and send data to the cloud.

[Azure IoT Operations](https://learn.microsoft.com/en-us/azure/iot-operations/overview-iot-operations) features an enterprise-grade MQTT broker that is deployed locally in an Arc-enabled Kubernetes cluster running at the edge site. With proper [Data Flows](https://learn.microsoft.com/en-us/azure/iot-operations/connect-to-cloud/overview-dataflow) configured, data gathered from the IIoT Gateway can be delivered to the cloud and control commands can be delivered from cloud to devices as well.

## Overview

This tutorial demonstrates a real-world scenario where a MCU-based device (an ARM Cortex-M4 based ***STM32F429ZI-Nucleo*** from [STMicroelectronics](https://www.st.com/content/st_com/en.html)) publishes MQTT messages to the [EG324 IoT Gateway](https://www.elastel.com/products/iot-gateway/eg324-iot-gateway/) The IIoT Gateway, which then sends data to the AIO MQTT broker. [Zephyr](https://docs.zephyrproject.org/latest/develop/getting_started/index.html) is the RTOS running on the STM32 MCU device. In this tutorial, we will build a Zephyr app publishing MQTT messages upstream from the device. Finally, we will use a MQTTX client subscribing to the topic and confirm messages are delivered correctly.

Here are the MCU device and Elastel EG324 IIoT Gateway configured in this tutorial:

<img title="" src="https://github.com/rickijen/azureiotoperations-elastel-zephyr/blob/main/artifacts/media/stm32f429zi.jpg?raw=true" alt="stm32" width="148">  <img title="" src="https://github.com/rickijen/azureiotoperations-elastel-zephyr/blob/main/artifacts/media/elastel-2.JPG?raw=true" alt="eg324" width="158">

## Architecture

<img title="" src="https://github.com/rickijen/azureiotoperations-elastel-zephyr/blob/main/artifacts/media/Elastel-HiveMQ.png?raw=true" alt="Architecture" data-align="inline">

## Prerequisites

- An [Azure Arc-enabled Kubernetes](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/overview) cluster (such as [K3s]([K3s](https://k3s.io/))) locally at the edge. Follow this [doc](https://learn.microsoft.com/en-us/azure/iot-operations/deploy-iot-ops/howto-prepare-cluster?tabs=ubuntu) to make sure your cluster is prepared to install IoT Operations.

- Elastel EG324 IIoT Gateway

- MCU-based development boards (from Nordic, STM32, NXP, etc.)

- Zephyer RTOS SDK toolchains and build environment.

## Install Azure IoT Operations

1. Make sure you have the latest Azure CLI extension
   
   ```
   az extension add --upgrade --name azure-iot-ops
   ```

2. Deploy to cluster (example: for my single node cluster)
   
   ```
   az iot ops create --subscription XXX \
   -g iot --cluster redondok3s --custom-location redondok3s-cl-3792 \
   -n redondok3s-ops-instance --resource-id /subscriptions/XXX/resourceGroups/iot/providers/Microsoft.DeviceRegistry/schemaRegistries/schema-registry-redondo \
   --broker-frontend-replicas 1 --broker-frontend-workers 1 \
   --broker-backend-part 1 --broker-backend-workers 1 \
   --broker-backend-rf 2 --broker-mem-profile Low \
   --ops-config observability.metrics.openTelemetryCollectorAddress=aio-otel-collector.azure-iot-operations.svc.cluster.local:4317 \
   --ops-config observability.metrics.exportInternalSeconds=60
   ```

3. Verify IoT ops service deployment for health, configuration, and usability

4. next

## Install Elastel EG324 IIoT Gateway

## Connect IIoT Gateway to AIO

## Build Zephyr app to publish MQTT to IIoT Gateway

## Resources

- [Deployment overview - Azure IoT Operations | Microsoft Learn](https://learn.microsoft.com/en-us/azure/iot-operations/deploy-iot-ops/overview-deploy)

- [EG324 IIoT Gateway, Arm-based industrial Computer - Elastel](https://www.elastel.com/products/iot-gateway/eg324-iot-gateway/)

- [Zephyr demo app](https://github.com/rickijen/zephyr-dhcp-mqtt)
