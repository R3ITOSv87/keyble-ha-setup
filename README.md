# Equiva key-ble Smart Lock Integration with Home Assistant using keyble and keyble-mqtt

![eq3](https://github.com/Strange-Panda/keyble-ha-setup/blob/267c88ebff28d724d5295abd09ff63c72d269e11/eqiva-bluethooth-smart-tuerschlossantrieb-r-os_142950a0-fcdf538f.png)

Hey there! Are you struggeling to integrate your eqiva smart lock into your smart home and control it using Home Assistant? Don't worry, there's a solution called keyble that allows you to control the Equiva key-ble Smart Lock using your computer and integrate it with Home Assistant.

This tutorial enables the integration of the Equiva key-ble Smart Lock into the Home Assistant system using keyble and keyble-mqtt. It allows you to conveniently control the door lock through the Home Assistant interface and trigger automated actions. You can lock, unlock, and open the door, as well as monitor the lock's status. However, it's important to note that the smart lock cannot handle two concurrent Bluetooth connections. If the manufacturer's official smartphone app is connected to the lock, keyble will not be able to connect to it simultaneously. Therefore, make sure to close the smartphone app for keyble to function properly. Additionally, it's crucial to be aware that making the door lock accessible over the network or the internet comes with security risks. It's important to take appropriate security measures to protect your network and Home Assistant installation.

Before we begin, I want to emphasize that making a door lock accessible over the network or the internet comes with certain security risks. If you choose to follow this guide, please be aware of the risks involved and take appropriate measures to secure your network and Home Assistant installation.

Alright, let's get started! I'll walk you through the steps on how to set up the Equiva key-ble Smart Lock with keyble and keyble-mqtt on a Raspberry Pi and integrate it with Home Assistant. Please note that you should already have a running Home Assistant instance and an MQTT broker installed. I won't cover the installation of these components in detail in this guide.

All Glory to https://github.com/oyooyo for developing key-ble! 

## Prerequisites

Before we begin, make sure you meet the following prerequisites:

- A Raspberry Pi or another Linux device (I used the Raspberry Pi Zero 2 W for this guide, but you can use other devices like the Banana Pi M2 Zero, which is more affordable.)
- An installed Home Assistant instance (https://www.home-assistant.io/getting-started/)
- A working MQTT broker (https://www.home-assistant.io/integrations/mqtt/)
- Bluetooth 4.0-compatible hardware (The Raspberry Pi Zero 2 W already has a built-in Bluetooth module, so you don't need an additional Bluetooth dongle.)
- Node.js (Version 10 or higher)
- A Debian-based operating system like Raspbian
- A equiva key-ble lock (https://www.eq-3.com/products/eqiva/detail/bluetooth-smart-lock.html)

Ensure that your Raspberry Pi is connected to the internet and update the system to have the latest updates and patches.

## Installation of keyble

To install keyble, open a terminal window on your Raspberry Pi and execute the following commands:

```
sudo apt-get -y update && sudo apt-get -y dist-upgrade
curl -sL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt-get install --upgrade -y build-essential nodejs
sudo apt-get -y install bluetooth bluez libbluetooth-dev libudev-dev
sudo npm install --update --global --unsafe-perm keyble
sudo setcap cap_net_raw+eip $(eval readlink -f `which node`)
```

These commands perform the following actions:

1. Update the system and install the latest updates.
2. Install Node.js LTS and the required dependencies.
3. Install Bluetooth libraries for communication with the smart lock.
4. Install keyble globally using npm. (This step might take a while! Take a coffee break.)
5. Allow keyble to run without sudo privileges.

Make sure the installation process completes without any errors.

## Registering a User with keyble-registeruser

To control the Equiva key-ble Smart Lock, you'll need a user ID and the corresponding 128-bit user key. Since the manufacturer's official app doesn't provide a way to retrieve this information, you'll first need to register a new user. For that, we'll use the keyble-registeruser tool.

Use the following command to register a new user:

```
keyble-registeruser -n homeassistant -q M0123456789ABK0123456789ABCDEF0123456789ABCDEFNEQ1234567
```

Replace `homeassistant` with your desired username and `M0123456789ABK0123456789ABCDEF0123456789ABCDEFNEQ1234567` with the information encoded in the QR code of the lock's "Key Card." Hold down the "Unlock" button until the yellow light flashes to activate the pairing mode. The tool will register the user and provide the necessary arguments you'll need to control the smart lock. If you encounter any issues with setting the name, you can cancel the action with Ctrl+C. Nevertheless, the user should still be registered, and you can verify it in the manufacturer's smartphone app.

Note down the MAC address of the smart lock (`address`), the user ID (`user_id`), and the user key (`user_key`) provided by the keyble-registeruser tool.

## Installation of keyble-mqtt

keyble-mqtt is an MQTT client that allows you to control the Equiva key-ble Smart Lock via MQTT. Install keyble-mqtt by executing the following command:

```
sudo npm install --update --global --unsafe-perm keyble-mqtt
```

Ensure that the installation completes without any errors.

## Running keyble-mqtt as a Service

To run keyble-mqtt as a service on your Raspberry Pi and ensure it starts automatically on system startup, we can create a systemd service file.

Create a new file named `keyble-mqtt.service` in the directory `/etc/systemd/system/` and insert the following content:

```plaintext
[Unit]
Description=keyble-mqtt
After=network.target

[Service]
ExecStart=/usr/bin/node /usr/lib/node_modules/keyble-mqtt/keyble-mqtt.js 01:23:45:67:89:ab 1 ca78ad9b96131414359e5e7cecfd7f9e --host 127.0.0.1
WorkingDirectory=/usr/lib/node_modules/keyble-mqtt
Restart=always

[Install]
WantedBy=multi-user.target
```

Make sure to replace the MAC address of the smart lock (`01:23:45:67:89:ab`) and the user data (`1` and `ca78ad9b96131414359e5e7cecfd7f9e`) in the `ExecStart` line.

Save the file and execute the following commands to enable and start the service:

```
sudo systemctl enable keyble-mqtt.service
sudo systemctl start keyble-mqtt.service
```

keyble-mqtt will now run as a service and automatically start on system startup. To ensure that the service started correctly, you can use the following command:

```
sudo systemctl status keyble-mqtt.service
```


## Configuration in Home Assistant

To integrate the Equiva key-ble Smart Lock into Home Assistant, you'll need to edit the Home Assistant configuration file.

Open the `configuration.yaml` file of Home Assistant and add the following YAML code:

```yaml
mqtt:
  lock:
    - name: Front Door
      state_topic: "keyble/01:23:45:67:89:ab/status"
      command_topic: "keyble/01:23:45:67:89:ab/command"
      payload_lock: "lock"
      payload_unlock: "unlock"
      payload_open: "open"
      state_locked: "LOCKED"
      state_unlocked: "UNLOCKED"
      state_locking: "MOVING"
      state_unlocking: "MOVING"
      optimistic: false
      qos: 0
      retain: true
      value_template: "{{ value_json.lock_status }}"
```

Make sure to replace the MAC address of the smart lock (`01:23:45:67:89:ab`) in the `state_topic` and `command_topic` properties. You can also adjust the name of the lock (`Front Door`).

Save the `configuration.yaml` file and restart Home Assistant to apply the configuration changes.

## Creating a Button Card to Control the Smart Lock

To control the Equiva key-ble Smart Lock in Home Assistant, we'll create a button card. Open the Lovelace configuration file of Home Assistant and add the following YAML code:

```yaml
type: vertical-stack
title: Front Door
cards:
  - type: entities
    entities:
      - entity: lock.front_door
    show_header_toggle: true
    state_color: true
  - type: grid
    cards:
      - type: custom:button-card
        name: Lock
        tap_action:
          action: call-service
          service: mqtt.publish
          service_data:
            topic: keyble/01:23:45:67:89:ab/command
            payload: lock
      - type: custom:button-card
        name: Unlock
        tap_action:
          action: call-service
          service: mqtt.publish
          service_data:
            topic: keyble/01:23:45:67:89:ab/command
            payload: unlock
      - type: custom:button-card
        name: Open
        tap_action:
          action: call-service
          service: mqtt.publish
          service_data:
            topic: keyble/01:23:45:67:89:ab/command
            payload: open
    columns: 3
    square: false
```

Ensure that you replace the MAC address of the smart lock (`01:23:45:67:89:ab`) in the `topic` properties of the button card.

Save the Lovelace configuration file and refresh the Home Assistant interface. You should now see a button card that allows you to lock, unlock, and open the smart lock.

## Conclusion

Congratulations! You have successfully integrated the Equiva key-ble Smart Lock with keyble and keyble-mqtt in Home Assistant. You can now control the smart lock through the Home Assistant interface and trigger automated actions. Explore further features of Home Assistant and expand your smart home control.

Enjoy controlling your Equiva key-ble Smart Lock in your smart home!
