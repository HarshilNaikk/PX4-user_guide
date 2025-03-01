# Gimbal Configuration

This page explains how to configure and control a gimbal that has an attached camera (or any other payload).

## Overview

PX4 contains a generic mount/gimbal control driver that supports different input and output methods:

- The input method defines the protocol used to command a gimbal mount that is managed by PX4. This might be an RC controller, a MAVLink command sent by a GCS, or both — automatically switching between them.
- The output method defines how PX4 communicates with the connected gimbal. The recommended protocol is MAVLink v2, but you can also connect directly to a flight controller PWM output port.

PX4 takes the input signal and routes/translates it to be sent through to the output. Any input method can be selected to drive any output.

Both the input and output are configured using parameters. The input is set using the parameter [MNT_MODE_IN](../advanced_config/parameter_reference.md#MNT_MODE_IN). By default this is set to `Disabled (-1)` and the driver does not run. After selecting the input mode, reboot the vehicle to start the mount driver.

You should set `MNT_MODE_IN` to one of: `RC (1)`, `MAVlink gimbal protocol v2 (4)` or `Auto (0)` (the other options are deprecated). If you select `Auto (0)`, the gimbal will automatically select either RC or or MAVLink input based on the latest input. Note that the auto-switch from MAVLink to RC requires a large stick motion!

The output is set using the [MNT_MODE_OUT](../advanced_config/parameter_reference.md#MNT_MODE_OUT) parameter. By default the output is set to a PXM port (`AUX (0)`). If the [MAVLink Gimbal Protocol v2](https://mavlink.io/en/services/gimbal_v2.html) is supported by your gimbal, you should instead select `MAVLink gimbal protocol v2 (2)`.

The full list of parameters for setting up the mount driver can be found in [Parameter Reference > Mount](../advanced_config/parameter_reference.md#mount). The relevant settings for a number of common gimbal configurations are described below.

## MAVLink 짐벌(MNT_MODE_OUT=MAVLINK)

Each physical gimbal device on the system must have its own high level gimbal manager, which is discoverable by a ground station using the MAVLink gimbal protocol. The ground station sends high level [MAVLink Gimbal Manager](https://mavlink.io/en/services/gimbal_v2.html#gimbal-manager-messages) commands to the manager of the gimbal it wants to control, and the manager will in turn send appropriate lower level "gimbal device" commands to control the gimbal.

PX4 can be configured as the gimbal manager to control a single gimbal device (which can either be physically connected or be a MAVLink gimbal that implements the [gimbal device interface](https://mavlink.io/en/services/gimbal_v2.html#gimbal-device-messages)).

To enable a MAVLink gimbal, first set parameter [MNT_MODE_IN](../advanced_config/parameter_reference.md#MNT_MODE_IN) to `MAVlink gimbal protocol v2` and [MNT_MODE_OUT](../advanced_config/parameter_reference.md#MNT_MODE_OUT) to `MAVLink gimbal protocol v2`.

짐벌은 \[MAVLink 주변 기기 (GCS/OSD/보조)\](../peripherals/mavlink_peripherals.md#mavlink-peripherals-gcsosdcompanion) 에서 다루는 방법과 같이 *어떤 여분의 직렬 포트*에든 연결할 수 있습니다([직렬 포트 구성](../peripherals/serial_configuration.md#serial-port-configuration)도 참고). For example, if the `TELEM2` port on the flight controller is unused you can connect it to the gimbal and set the following PX4 parameters:
- [MAV_1_CONFIG](../advanced_config/parameter_reference.md#MAV_1_CONFIG)에서 **TELEM2**까지(`MAV_1_CONFIG`가 이미 보조 컴퓨터에 사용되고 있는 경우(예: `MAV_2_CONFIG` 사용))
- [MAV_1_MODE](../advanced_config/parameter_reference.md#MAV_1_MODE)에서 **NORMAL**로
- 제조업체 권장 전송 속도에 대한 [SER_TEL2_BAUD](../advanced_config/parameter_reference.md#SER_TEL2_BAUD).

### Multiple Gimbal Support

PX4 can automatically create a gimbal manager for a connected PWM gimbal or the first MAVLink gimbal device with the same system id it detects on any interface. It does not automatically create gimbal manager for any other MAVLink gimbal devices that it detects.

You can support additional gimbals provided that they:

- implement the gimbal _manager_ protocol
- Are visible to the ground station and PX4 on the MAVLink network. This may require that traffic forwarding be configured between PX4, the GCS, and the gimbal.
- Each gimbal must have a unique component id. For a PWM connected gimbal this will be the component ID of the autopilot


## Gimbal on FC PWM Output (MNT_MODE_OUT=AUX)

The gimbal can also be controlled by connecting it to up to three flight controller PWM ports and setting the output mode to `MNT_MODE_OUT=AUX`.

The output pins that are used to control the gimbal are set in the [Acuator Configuration > Outputs](../config/actuators.md#actuator-outputs) by selecting any three unused Actuator Outputs and assigning them the following output functions:
- `Gimbal Roll`: Output controls gimbal roll.
- `Gimbal Pitch`: Output controls Gimbal pitch.
- `Gimbal Yaw`: Output controls Gimbal pitch.

For example, you might have the following settings to assign the gimbal roll, pitch and yaw to AUX1-3 outputs.

![Gimbal Actuator config](../../assets/config/actuators/qgc_actuators_gimbal.png)

The PWM values to use for the disarmed, maximum and minimum values can be determined in the same way as other servo, using the [Actuator Test sliders](../config/actuators.md#actuator-testing) to confirm that each slider moves the appropriate axis, and changing the values so that the gimbal is in the appropriate position at the disarmed, low and high position in the slider. The values may also be provided in gimbal documentation.

## SITL

The [Gazebo Classic](../sim_gazebo_classic/README.md) simulation [Typhoon H480 model](../sim_gazebo_classic/gazebo_vehicles.md#typhoon-h480-hexrotor) comes with a preconfigured simulated gimbal.

실행하려면 다음을 사용하십시오.

```
make px4_sitl gazebo-classic_typhoon_h480
```

To just test the [gimbal driver](../modules/modules_driver.md#gimbal) on other models or simulators, make sure the driver runs (using `gimbal start`), then configure its parameters.

## 시험

The driver provides a simple test command. 다음은 SITL에서의 테스트 방법을 설명합니다. 이 명령은 실제 장치에서도 작동합니다.

다음을 사용하여 시뮬레이션을 시작합니다(이를 위해 매개변수를 변경할 필요는 없음).

```
make px4_sitl gazebo-classic_typhoon_h480
```

예를 들어 시동 여부를 확인하십시오. `commander takeoff` 명령어를 실행한 다음, 다음 명령을 사용하여 짐벌(예)을 제어합니다:

```
gimbal test yaw 30
```

시뮬레이션된 짐벌은 자체적으로 안정적이므로, MAVLink 명령을 보내는 경우 `stabilize` 플래그를 `false`로 설정합니다.

![Gazebo 짐벌 시뮬레이션](../../assets/simulation/gazebo_classic/gimbal-simulation.png)
