# IoT Central Lorawan TTN Hackaton
The goal of this workshop is to setup a working end-to-end IoT solution based on Azure IoT Central, The Things Network and Lorawan.

The overall flow for the IoT telemetry:

![flow image][flow]

The device we will be using:

![node image][node] 

The gateway used:

![laird image][laird]


During this hackaton you will complete the following tasks:
- Setup an Arduino Development Environment
- Configure and program a device
- Configure and setup a LoraWAN gateway 
- Setup conncectivity from the gateway to The Things Network
- Decode data using custom code in The Things Network
- Send data from The Things Network to IoT Central via and device bridge running in an Azure Function
- Create and configure IoT Central

### Prerequisites
- Laptop
    - Linux, Mac or Windows
- Azure Subscription
    - [https://portal.azure.com](https://portal.azure.com)
- TTN Node Device
    - _"The Things Node is based off the SparkFun Pro Micro - 3.3V/8Mhz with added Microchip LoRaWAN module and temperature sensor, NXP’s digital accelerometer, a light sensor, button and RGB LED. All this is packaged in a matchbox-sized waterproof (IP54) casing with 3 AAA batteries to power it for months of usage."_
    - [https://www.thethingsnetwork.org/docs/devices/node/](https://www.thethingsnetwork.org/docs/devices/node/)
- Lorawan Gateway
    - [https://connectivity.lairdtech.com/wireless-modules/lorawan-solutions/sentrius-rg1xx-lora-enabled-gateway-wi-fi-bluetooth-ethernet](https://connectivity.lairdtech.com/wireless-modules/lorawan-solutions/sentrius-rg1xx-lora-enabled-gateway-wi-fi-bluetooth-ethernet)
- USB to mini USB cable for programming the device
- Basic understanding of IoT and programming
- Basic understanding of Microsoft Azure
- Optional: 3 x AAA batteries if you want to test the device without being connected to your laptop

[flow]: /resources/flow.png "Flow"
[node]: /resources/ttn-node.png "Node"
[laird]: /resources/laird.png "Laird"
[device-eui]: /resources/dev-eui.png "Device EUI"


### Contents
- [Install Arduino IDE](#arduioneide)
    - [Install Arduino Boards](#arduioneideboards)
    - [Install TTN Arduino Packages](#arduioneidepackages)
- [Configure & Connect Gateway to TTN](#gwttn) 
- [Upload App Code v.1 to TTN Node](#appcode)
- [Create IoT Central Instance](#iotc)
- [Create IoT Device Bridge with an Azure Function](#iotfunc)
- [Add TTN Decoder v.1 to the TTN Application](#ttndecoder)
- [Setup TTN HTTP Integration](#ttnhttp)
- [Associate, Connect & View Device Data in IoT Central](#iotcconnect)
- [Add a Button Pressed Event](#btnpressevent)
    - [TTN Node Code v.2](#ttnnodecodev2)
    - [TTN Decoder v.2](#ttndecoderv2)
    - [Add IoT Central Event](#iotcevent)
- [OPTIONAL: Build a Device Template Dashboard in IoT Central](#iotcdevicetemplate)
- [OPTIONAL: Setup Alert in IoT Central](#iotcalert)
- [OPTIONAL: Setup Continuous Export from IoT Central](#iotccontexport)

## <a name="arduioneide">Install Arduino IDE</a>
- Follow the guide here [https://www.arduino.cc/en/Guide/HomePage](https://www.arduino.cc/en/Guide/HomePage) to install the IDE for your platform

### <a name="arduioneideboards">Install Arduino Boards</a>
- The Pro Micro boards are not developed by Arduino. As a result the boards are not available in a newly installed IDE
- Add the boards in the IDE: 
    - Open 'Preferences' from the 'File' menu
    - In the 'Additional Boards Manager URL' enter https://raw.githubusercontent.com/sparkfun/Arduino_Boards/master/IDE_Board_Manager/package_sparkfun_index.json
    - Close the dialog with 'OK'
- Go to the 'Tools' menu, item 'Board: "Arduino/Genuino Uno"', this will open a sub menu, select 'Board Manager'
    - In the boards manager, type 'sparkfun' in the dialog to show all SparkFun provided board packages
    - Click on 'SparkFun AVR Boards' to select the package and click 'Install' to install the board definitions on your system
- Open the 'Tools' menu and go to the 'Board: "Arduino/Genuino Uno"' again. Now select 'SparkFun Pro Micro' from the list
- __IMPORTANT: Open the 'Tools' menu again, in 'Processor' select 'ATmega32u4 (_3.3V, 8MHz_)'__

### <a name="arduioneidepackages">Install TTN Arduino Packages</a>
- The TTN Node has some specific packages you can install to get started quickly. They are not in the IDE by default so you have to install them.
- Add the packages in the IDE:
    - Open the 'Sketch' menu
    - Click 'Include Library' then 'Manage Libraries'
    - In the Library Manager search for 'thethingsnode' and click 'Install'
    - In the Library Manager search for 'thethingsnetwork' and click 'Install'
    - Click 'Close'
- You can now find 'TheThingsNetwork' and 'TheThingsNode' under 'Contributed Libraries'
    - The easiest way to get started is the 'Basic' sketch, but you will be guided later on the specific code to use

## <a name="gwttn">Configure & Connect Gateway to TTN</a>
- Laird GW (the one tested for this guide)
  - Setup your Laird GW as described in this PDF doc [https://connectivity-staging.s3.us-east-2.amazonaws.com/s3fs-public/2018-10/Sentrius%20RG1xx%20Quick%20Start%20Guide%20v2_1.pdf](https://connectivity-staging.s3.us-east-2.amazonaws.com/s3fs-public/2018-10/Sentrius%20RG1xx%20Quick%20Start%20Guide%20v2_1.pdf) in chapters 3, 4 and 5
    - You can use a wired connection, but you probably want to setup wifi as described in chapter 5.2
  - Configure packet forwarding as describe in chapter 6
    - Important use the "The Things Network Legacy" option!
  - Complete the configuration with TTN as described in chapter 7 
    - This will create both a gateway and an application in TTN
    - Tip: you will need the Device EUI for the application in TTN. You can get this via the Arduino Serial Monitor ![deviceeui image][device-eui] 
- For other supported gateways see here: https://www.thethingsnetwork.org/docs/gateways/ and follow the guidance provided

## <a name="appcode">Upload App Code v.1 to TTN Node</a>
- Use the IDE to upload the following code to the device
    - Remember to insert in your own 'appEui' and 'appKey'
```
#include <TheThingsNode.h>

// Set your AppEUI and AppKey
const char *appEui = "INSERT YOU OWN";
const char *appKey = "INSERT YOU OWN";

#define loraSerial Serial1
#define debugSerial Serial

// Replace REPLACE_ME with TTN_FP_EU868 or TTN_FP_US915
#define freqPlan TTN_FP_EU868

TheThingsNetwork ttn(loraSerial, debugSerial, freqPlan);
TheThingsNode *node;

#define PORT_SETUP 1
#define PORT_INTERVAL 2
#define PORT_MOTION 3
#define PORT_BUTTON 4

void setup()
{
  loraSerial.begin(57600);
  debugSerial.begin(9600);

  // Wait a maximum of 10s for Serial Monitor
  while (!debugSerial && millis() < 10000)
    ;

  // Config Node
  node = TheThingsNode::setup();
  node->configLight(true);
  node->configInterval(true, 60000);
  node->configTemperature(true);
  node->onWake(wake);
  node->onInterval(interval);
  node->onSleep(sleep);
  node->onMotionStart(onMotionStart);
  node->onButtonRelease(onButtonRelease);

  // Test sensors and set LED to GREEN if it works
  node->showStatus();
  node->setColor(TTN_GREEN);

  debugSerial.println("-- TTN: STATUS");
  ttn.showStatus();

  debugSerial.println("-- TTN: JOIN");
  ttn.join(appEui, appKey);

  debugSerial.println("-- SEND: SETUP");
  sendData(PORT_SETUP);
}

void loop()
{
  node->loop();
}

void interval()
{
  node->setColor(TTN_BLUE);

  debugSerial.println("-- SEND: INTERVAL");
  sendData(PORT_INTERVAL);
}

void wake()
{
  node->setColor(TTN_GREEN);
}

void sleep()
{
  node->setColor(TTN_BLACK);
}

void onMotionStart()
{
  node->setColor(TTN_BLUE);

  debugSerial.print("-- SEND: MOTION");
  sendData(PORT_MOTION);
}

void onButtonRelease(unsigned long duration)
{
  node->setColor(TTN_BLUE);

  debugSerial.print("-- SEND: BUTTON");
  debugSerial.println(duration);

  sendData(PORT_BUTTON);
}

void sendData(uint8_t port)
{
  // Wake RN2483
  ttn.wake();

  ttn.showStatus();
  node->showStatus();

  byte *bytes;
  byte payload[6];

  //Battery
  uint16_t battery = node->getBattery();
  bytes = (byte *)&battery;
  payload[0] = bytes[1];
  payload[1] = bytes[0];

  //Light
  uint16_t light = node->getLight();
  bytes = (byte *)&light;
  payload[2] = bytes[1];
  payload[3] = bytes[0];

  // Temperature
  int16_t temperature = round(node->getTemperatureAsFloat() * 100);
  bytes = (byte *)&temperature;
  payload[4] = bytes[1];
  payload[5] = bytes[0];

  ttn.sendBytes(payload, sizeof(payload), port);

  // Set RN2483 to sleep mode
  ttn.sleep(60000);

  // This one is not optionnal, remove it
  // and say bye bye to RN2983 sleep mode
  delay(50);
}
```
- Make sure you have connected the device using USB
- Make sure you have selected the correct COM port in the IDE
- Upload the code to the device 
- View the Serial Monitor to validate that the device connects to gw and sends data
### Validate Data From TTN Node Device to TTN
- On the TTN website go and view the data packages from the device on the TTN Application
- Unless you are really good a decoding a byte array you will probably need a Decoder to understand the data...
- The same goes for IoT Central as it will expect a JSON payload not a byte array

## <a name="iotc">Create IoT Central Instance</a>
- Follow the guide here to deploy you IoT Central: https://docs.microsoft.com/en-us/azure/iot-central/quick-deploy-iot-central
    - In step 1: Choose the 'Pay-As-You-Go' otherwise you will loose everything after 7 days and cost for this setup is minimal
    - In step 3: Choose 'Custom application' 
- For general reference on the UI of IoT Central refer to: https://docs.microsoft.com/en-us/azure/iot-central/overview-iot-central-tour

### Create a Device Template
- In the newly create IoT Central create a device template
- Telemetry:
    - Battery Voltage
        Field: battery
    - Light
        Field: light
    - Temperature
        Field: temperature
- State:
    - Device state
        - Field: event
        - Values: button, setup, interval, motion

## <a name="iotfunc">Create IoT Device Bridge with an Azure Function</a> 
- Follow the guidance on https://github.com/Azure/iotc-device-bridge to deploy and configure the device bridge on an Azure Function
    - Replace the code in the method in line 19 with the following: 
```
module.exports = async function (context, req) {
    try {
        context.log('req: ', req);
        context.log('measurements: ', req.body.measurements);
        context.log('hw serial: ', req.body.hardware_serial.toLowerCase());
        req.body = {
            device: {
                deviceId: req.body.hardware_serial.toLowerCase()
            },
            measurements: req.body.payload_fields
        };

        await handleMessage({ ...parameters, log: context.log, getSecret: getKeyVaultSecret }, req.body.device, req.body.measurements, req.body.timestamp);
    } catch (e) {
        context.log('[ERROR]', e.message);

        context.res = {
            status: e.statusCode ? e.statusCode : 500,
            body: e.message
        };
    }
}
```
## <a name="ttndecoder">Add TTN Decoder v.1 to the TTN Application</a>
- On the application in TTN add a decoder using the code below
    - Application > Payload Formats > Decoder
```
function Decoder(bytes, port) {
  var decoded = {};
  var events = {
    1: 'setup',
    2: 'interval',
    3: 'motion',
    4: 'button'
  };
  decoded.event = events[port];
  decoded.battery = (bytes[0] << 8) + bytes[1];
  decoded.light = (bytes[2] << 8) + bytes[3];
  if (bytes[4] & 0x80)
    decoded.temperature = ((0xffff << 16) + (bytes[4] << 8) + bytes[5]) / 100;
  else {
    decoded.temperature = ((bytes[4] << 8) + bytes[5]) / 100;
  }
  
    return {
        "temperature": JSON.stringify(decoded.temperature),
        "light" : JSON.stringify(decoded.light),
        "battery" : JSON.stringify(decoded.battery),
        "event" : decoded.event
    };
}
```
## <a name="ttnhttp">Setup TTN HTTP Integration</a>
- On the application in TTN add an integration to the Azure Function
    - Application > Integrations > add integration > HTTP Integration
    - Get the endpoint from the Azure Function as describe in step 5 on https://github.com/Azure/iotc-device-bridge

##  <a name="iotcconnect">Associate, Connect & View Device Data in IoT Central</a>
- We are using the approach described here: https://docs.microsoft.com/en-us/azure/iot-central/concepts-connectivity#connect-without-registering-devices 
    - Make sure your device is sending data to TTN and that TTN is sending data to the Azure Function
    - In IoT Central associate and approve the device
    - Wait a couple of minutes and verify that data is being received in IoT Central

## <a name="btnpressevent">Add a Button Pressed Event</a>
In this part you only get hints. By now you are ready to start modifying the code yourself!

### <a name="ttnnodecodev2">TTN Node Code v.2</a>
These are just hint complete the code yourself :-)
```
if(INSERT YOUR OWN LOGIC) {
    payload[6] = (byte *)true; 
}
else {
    payload[6] = (byte *)false;
}
```

### <a name="ttndecoderv2">TTN Decoder v.2</a>
These are just hint complete the code yourself :-)
```
if(INSERT YOUR OWN LOGIC) {
    decoded.buttonpressed = true;
}
else {
    decoded.buttonpressed = false;
}
```
```
if(decoded.buttonpressed)
{
    return {
        "temperature": JSON.stringify(decoded.temperature),
        "light" : JSON.stringify(decoded.light),
        "battery" : JSON.stringify(decoded.battery),
        "event" : decoded.event,
        "buttonpressed" : INSERT YOUR OWN CODE
    };
}
else {
    return {
        INSERT YOUR OWN CODE
    };
}
```

### <a name="iotcevent">Add IoT Central Event</a>
- Add an Event type on the Device Template called 'Button Pressed' using the telemetry field from the device
    - Field: buttonpressed

## <a name="iotcdevicetemplate">OPTIONAL: Build a Device Template Dashboard in IoT Central</a>
- Build a nice looking dashboard on the Device Template showing important information

## <a name="iotcalert">OPTIONAL: Setup Alert in IoT Central</a>
- Setup and an Alert sending an email of the temperature is above a threshold for 5 minutes
https://docs.microsoft.com/en-us/azure/iot-central/tutorial-configure-rules

## <a name="iotccontexport">OPTIONAL: Setup Continuous Export from IoT Central</a>
- Setup Continuous Export from IoT Central to Azure Blob Storage
https://docs.microsoft.com/en-us/azure/iot-central/howto-export-data-blob-storage
