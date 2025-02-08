# Connecting Industrial IoT Gateway to Azure IoT Operations

An Industrial IoT solution deployed at the edge location often integrates microcontroller-based devices, sensors, and PLCs (programmable logic controllers) into industrial processes to gather data, monitor operations, and improve efficiency.  The goal is to enable real-time monitoring, predictive maintenance, automation, and data-driven decision-making in industries such as manufacturing, building automation, energy, transportation, and agriculture.

IIoT gateway is a critical component in the solution to aggregate data via industry standard protocols (such as [Modbus]([Modbus - Wikipedia](https://en.wikipedia.org/wiki/Modbus)), [BACnet]([BACnet - Wikipedia](https://en.wikipedia.org/wiki/BACnet)), [OPC UA]([OPC Unified Architecture - Wikipedia](https://en.wikipedia.org/wiki/OPC_Unified_Architecture)), [Sparkplug]([Eclipse Sparkplug Working Group | The Eclipse Foundation](https://sparkplug.eclipse.org/)), etc.) and send data to the cloud.

[Azure IoT Operations]([What is Azure IoT Operations? - Azure IoT Operations | Microsoft Learn](https://learn.microsoft.com/en-us/azure/iot-operations/overview-iot-operations)) features an enterprise-grade MQTT broker that is deployed locally in an Arc-enabled Kubernetes cluster running at the edge site. With proper [Data Flows]([Process and route data with data flows - Azure IoT Operations | Microsoft Learn](https://learn.microsoft.com/en-us/azure/iot-operations/connect-to-cloud/overview-dataflow)) configured, data gathered from the IIoT Gateway can be delivered to the cloud and control commands can be delivered from cloud to devices as well.

## Overview

This tutorial demonstrates a real-world scenario where a MCU-based device (an ARM Cortex-M4 based ***STM32F429ZI-Nucleo*** from [STMicroelectronics]([STMicroelectronics: Our technology starts with you](https://www.st.com/content/st_com/en.html))) publishes MQTT messages to the [Elastel EG324]([EG324 IoT Gateway, Arm-based industrial Computer - Elastel](https://www.elastel.com/products/iot-gateway/eg324-iot-gateway/)) IIoT Gateway, which then sends data to the AIO MQTT broker. [Zephyr]([Getting Started Guide — Zephyr Project Documentation](https://docs.zephyrproject.org/latest/develop/getting_started/index.html)) is the RTOS running on the STM32 device. In this tutorial, we will build a Zephyr app publishing MQTT messages upstream. Finally, we will subscribe the AIO MQTT broker to confirm messages are delivered correctly.

## Architecture

![Architecture](C:\Users\rijen\Downloads\Elastel-HiveMQ.png)

## Prerequisites

- An [Arc-enabled Kubernetes]([Overview of Azure Arc-enabled Kubernetes - Azure Arc | Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/overview)) cluster (such as [K3s]([K3s](https://k3s.io/))) locally at the edge.

- Elastel EG324 IIoT Gateway

- MCU-based development boards (from Nordic, STM32, NXP, etc.)

- Zephyer RTOS SDK toolchains and build environment.

## Install Azure IoT Operations

## Install Elastel EG324 IIoT Gateway

## Connect IIoT Gateway to AIO

## Build Zephyr app to publish MQTT to IIoT Gateway

## Resources

- [Deploy Azure IoT Operations to your cluster]([Deployment overview - Azure IoT Operations | Microsoft Learn](https://learn.microsoft.com/en-us/azure/iot-operations/deploy-iot-ops/overview-deploy))

- [EG324 IIoT Gateway, Arm-based industrial Computer - Elastel](https://www.elastel.com/products/iot-gateway/eg324-iot-gateway/)

- [Zephyr demo app](https://github.com/rickijen/zephyr-dhcp-mqtt)
