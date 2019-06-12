# Sense Energy Monitor API for Node (unofficial)

This is an unofficial API for the Sense Energy Monitor. It was forked from blandman's unofficial-sense implementation. The key difference is this implementation does not automatically start up the websocket stream. It also contains additional error handling and promises.

This version is used by the [SmartThings-Sense app](https://github.com/brbeaird/SmartThings_SenseMonitor).

## Important Note on Real-time websocket streaming
After opening the websocket stream, it is strongly recommended that you immediately close the stream after receiving a packet of data. If you leave the stream open, you will likely run into rate limiting from Sense servers. Sense is known to have limits on concurrent websocket streams per account. If you leave a stream open and then try to open the Sense mobile or web app, the real-time view in those apps likely will not function. Open streams also generate unnecessary work on Sense servers.

## Installation

`npm install sense-energy-node`

## Exposed objects and methods
- authData: gets account details after logging in
- getDevices: gets list of all devices associated with the account
- getMonitorInfo: gets information on the monitor device itself
- getTimeline: gets recent events from the Sense timeline
- getAuth: triggers re-authentication (used in case token expires)
- openStream/closeStream: opens/closes the websocket stream
- events.on('data'): subscribes to websocket data

## Usage Example

```javascript
const sense = require('node-sense-monitor')
const email = 'email';
const password = 'password';

//Global Variables
var mySense;                        //Main Sense API object
var currentlyProcessing = false;    //Flag to ensure only one websocket packet is handled at a time
var websocketPollingInterval = 60;  //Number of seconds between opening/closing the websocket

startSense();

async function startSense(){
    try {
        mySense = await sense({email: email, password: password, verbose: false})   //Set up Sense API object and authenticate

        //Get devices
        await mySense.getDevices().then(devices => {
            for (let dev of devices) {
                console.log(dev);   //Process devices here
            }
        });

        //Get monitor info
        await mySense.getMonitorInfo().then(monitor => {
            console.log(monitor);   //Process monitor details here
        })

        //Handle websocket data updates (one at a time)
        mySense.events.on('data', (data) => {

            //Check for loss of authorization. If detected, try to reauth
            if (data.payload.authorized == false){
                console.log('Authentication failed. Trying to reauth...');
                refreshAuth();
            }

            //Set processing flag so we only send through and process one at a time
            if (data.type === "realtime_update" && data.payload && data.payload.devices) {
                mySense.closeStream();
                if (currentlyProcessing){
                    return 0;
                }
                currentlyProcessing = true;
                console.log(`Fresh websocket device data incoming...`)

                if (data.payload && data.payload.devices) {
                    console.log(data.payload.devices[0]);   //Handle websocket stream devices here
                }
            }
            return 0;
        });

        //Handle closures and errors
        mySense.events.on('close', (data) => {
            console.log(`Sense WebSocket Closed | Reason: ${data.wasClean ? 'Normal' : data.reason}`);
            let interval = websocketPollingInterval && websocketPollingInterval > 10 ? websocketPollingInterval : 60;

            //On clean close, set up the next scheduled check
            console.log(`New poll scheduled for ${interval * 1000} ms from now.`);
            setTimeout(() => {
                mySense.openStream();
            },  interval * 1000);
        });
        mySense.events.on('error', (data) => {
            console.log('Error: Sense WebSocket Closed | Reason: ' + data.msg);
        });

        //Open websocket flow (and re-open again on an interval)
        mySense.openStream();

    } catch (error) {
        console.log(`FATAL ERROR: ${error}`);
        if (error.stack){
            console.log(`FATAL ERROR: ${error.stack}`);
        }
        process.exit();
    }
}

//Attempt to refresh auth
function refreshAuth(){
    try {
        mySense.getAuth();
    } catch (error) {
        console.log(`Re-auth failed: ${error}. Exiting.`);
        process.exit();
    }
}

```

## API Response Reference

### Websocket - Authenticated
```json
{
    "status": "Authenticated",
    "data": {
        "authorized": true,
        "account_id": "xxxxx",
        "user_id": "xxxxx",
        "access_token": "token",
        "monitors":
        [ { "id": "xxxxx",
            "time_zone": "America/Los_Angeles",
            "solar_connected": false,
            "solar_configured": false,
            "online": false,
            "attributes": Object,
            "signal_check_completed_time": "2017-10-17T23:51:23.000Z" } ],
        "bridge_server": "wss://mb1.home.sense.com",
        "partner_id": null,
        "date_created": "2017-10-17T23:52:37.000Z"
    }
}
```
### Websocket - Recevied
```json
{
    "status": "Received",
    "data": {
    "type": "realtime_update",
    "payload": {
        "hz": 60.003795623779,
        "c": 122,
        "channels": [
        5870.3833007812,
        5293.4794921875
        ],
        "devices": [
        {
            "c": 61,
            "w": 5584.701171875,
            "name": "Dryer",
            "icon": "washer",
            "id": "11654c89",
            "tags": {
            "AlwaysOn": "false",
            "UserEditable": "true",
            "DateFirstUsage": "2017-10-19",
            "name_useredit": "false",
            "OriginalName": "Dryer",
            "DeviceListAllowed": "true",
            "DefaultUserDeviceType": "Dryer",
            "TimelineAllowed": "true",
            "Type": "ElectricDryer",
            "UserDeviceType": "Dryer",
            "DeployToMonitor": "true",
            "Revoked": "false",
            "PeerNames": [
                {
                "UserDeviceType": "Dryer",
                "Percent": 97,
                "UserDeviceTypeDisplayString": "Dryer",
                "Icon": "washer",
                "Name": "Dryer"
                },
                {
                "UserDeviceType": "Washer",
                "Percent": 1,
                "UserDeviceTypeDisplayString": "Washer",
                "Icon": "dishes",
                "Name": "Washer"
                },
                {
                "UserDeviceType": "StoveTop",
                "Percent": 1,
                "UserDeviceTypeDisplayString": "Stove Top",
                "Icon": "stove",
                "Name": "Stove"
                },
                {
                "UserDeviceType": "WaterHeater",
                "Percent": 1,
                "UserDeviceTypeDisplayString": "Water Heater",
                "Icon": "heat",
                "Name": "Water heater"
                }
            ],
            "PreselectionIndex": 0,
            "UserDeviceTypeDisplayString": "Dryer",
            "Mature": "true",
            "Alertable": "true",
            "ModelCreatedVersion": "10",
            "TimelineDefault": "true",
            "Pending": "false",
            "user_editable": "true"
            }
        },
        {
            "c": 47,
            "w": 4340.5068359375,
            "name": "Other",
            "icon": "home",
            "id": "unknown",
            "tags": {
            "DefaultUserDeviceType": "Unknown",
            "TimelineAllowed": "false",
            "UserDeviceType": "Unknown",
            "UserEditable": "false",
            "UserDeviceTypeDisplayString": "Unknown",
            "DeviceListAllowed": "true",
            "user_editable": "false"
            }
        },
        {
            "c": 7,
            "w": 726,
            "name": "Always On",
            "icon": "alwayson",
            "id": "always_on",
            "tags": {
            "DefaultUserDeviceType": "AlwaysOn",
            "TimelineAllowed": "false",
            "UserDeviceType": "AlwaysOn",
            "UserEditable": "false",
            "UserDeviceTypeDisplayString": "AlwaysOn",
            "DeviceListAllowed": "true",
            "user_editable": "false"
            }
        },
        {
            "c": 5,
            "w": 512.65478515625,
            "name": "Washer",
            "icon": "dishes",
            "id": "ccd27606",
            "tags": {
            "AlwaysOn": "false",
            "UserEditable": "true",
            "DateFirstUsage": "2017-10-19",
            "name_useredit": "false",
            "OriginalName": "Washer",
            "DeviceListAllowed": "true",
            "DefaultUserDeviceType": "Washer",
            "TimelineAllowed": "true",
            "Type": "WashingMachine",
            "UserDeviceType": "Washer",
            "DeployToMonitor": "true",
            "Revoked": "false",
            "PeerNames": [
                {
                "UserDeviceType": "Washer",
                "Percent": 96,
                "UserDeviceTypeDisplayString": "Washer",
                "Icon": "dishes",
                "Name": "Washer"
                }
            ],
            "PreselectionIndex": 0,
            "UserDeviceTypeDisplayString": "Washer",
            "Mature": "true",
            "Alertable": "true",
            "ModelCreatedVersion": "8",
            "TimelineDefault": "true",
            "Pending": "false",
            "user_editable": "true"
            }
        }
        ],
        "w": 11163.86328125,
        "_stats": {
        "mrcv": 1528234116.29,
        "brcv": 1528234116.1969,
        "msnd": 1528234116.29
        },
        "epoch": 1528234102,
        "deltas": [

        ],
        "voltage": [
        121.58633422852,
        122.09526824951
        ],
        "frame": 18657330
    }
    }
}
```

### *getDevices()*

```json
[
    {
      "id": "xxxxxxx",
      "name": "Microwave",
      "icon": "microwave",
      "tags": {
        "Alertable": "true",
        "AlwaysOn": "false",
        "DateFirstUsage": "2017-10-18",
        "DefaultUserDeviceType": "Microwave",
        "DeployToMonitor": "true",
        "DeviceListAllowed": "true",
        "Mature": "true",
        "ModelCreatedVersion": "10",
        "name_useredit": "false",
        "OriginalName": "Microwave",
        "PeerNames": [
          {
            "Name": "Microwave",
            "UserDeviceType": "Microwave",
            "Percent": 100,
            "Icon": "microwave",
            "UserDeviceTypeDisplayString": "Microwave"
          }
        ],
        "Pending": "false",
        "PreselectionIndex": 0,
        "Revoked": "false",
        "TimelineAllowed": "true",
        "TimelineDefault": "true",
        "Type": "Microwave",
        "user_editable": "true",
        "UserDeviceTypeDisplayString": "Microwave",
        "UserEditable": "true"
      }
    },
    {
      "id": "xxxxxxx",
      "name": "Stove 2",
      "icon": "stove",
      "tags": {
        "Alertable": "true",
        "AlwaysOn": "false",
        "DateFirstUsage": "2017-10-19",
        "DefaultUserDeviceType": "StoveTop",
        "DeployToMonitor": "true",
        "DeviceListAllowed": "false",
        "Mature": "true",
        "ModelCreatedVersion": "12",
        "name_useredit": "false",
        "OriginalName": "Stove 2",
        "PeerNames": [
          {
            "Name": "Stove",
            "UserDeviceType": "StoveTop",
            "Percent": 9000,
            "Icon": "stove",
            "UserDeviceTypeDisplayString": "Stove Top"
          }
        ],
        "Pending": "false",
        "PreselectionIndex": 0,
        "Revoked": "true",
        "TimelineAllowed": "false",
        "TimelineDefault": "true",
        "Type": "Stove",
        "user_editable": "false",
        "UserDeviceTypeDisplayString": "Stove Top",
        "UserEditable": "false"
      }
    },
    {
      "id": "xxxxxxxx",
      "name": "Washer",
      "icon": "dishes",
      "tags": {
        "Alertable": "true",
        "AlwaysOn": "false",
        "DateFirstUsage": "2017-10-19",
        "DefaultUserDeviceType": "Washer",
        "DeployToMonitor": "true",
        "DeviceListAllowed": "true",
        "Mature": "true",
        "ModelCreatedVersion": "8",
        "name_useredit": "false",
        "OriginalName": "Washer",
        "PeerNames": [
          {
            "Name": "Washer",
            "UserDeviceType": "Washer",
            "Percent": 96,
            "Icon": "dishes",
            "UserDeviceTypeDisplayString": "Washer"
          }
        ],
        "Pending": "false",
        "PreselectionIndex": 0,
        "Revoked": "false",
        "TimelineAllowed": "true",
        "TimelineDefault": "true",
        "Type": "WashingMachine",
        "user_editable": "true",
        "UserDeviceTypeDisplayString": "Washer",
        "UserEditable": "true"
      }
    }
]
```
### *getMonitorInfo()*

```json
{
  "signals": {
    "progress": 100,
    "status": "OK"
  },
  "device_detection": {
    "in_progress": [
      {
        "icon": "cup",
        "name": "Possible Coffee Maker",
        "progress": 16
      },
      {
        "icon": "dishes",
        "name": "Possible Dishwasher",
        "progress": 5
      },
      {
        "icon": "stove",
        "name": "Possible Oven",
        "progress": 10
      }
    ],
    "found": [
      {
        "icon": "stove",
        "name": "Oven",
        "progress": 48
      },
      {
        "icon": "dishes",
        "name": "Washer",
        "progress": 50
      },
      {
        "icon": "home",
        "name": "Motor",
        "progress": 96
      }
    ],
    "num_detected": 18
  },
  "monitor_info": {
    "serial": "XXXXXXXXXX",
    "ndt_enabled": true,
    "online": true,
    "version": "1.11.1870-2d186a5-master",
    "ssid": "For the Horde",
    "signal": "-44 dBm",
    "mac": "xx:xx:xx:xx:xx:xx"
  }
}
```

### *getTimeline()*
```json
{
  "more_items": true,
  "sticky_items": [
    {
      "time": "2018-06-01T05:56:24.350Z",
      "type": "Info",
      "subtype": "NewDeviceFound",
      "icon": "heat",
      "body": "Sense found a new device and named it 'Heat 3'.",
      "destination": "device:fbcff7e5",
      "device_id": "fbcff7e5",
      "device_name": "Heat 3",
      "show_action": false,
      "allow_sticky": true,
      "guid": "f733cb4f-5250-479b-8fc6-64d2dc6baedc",
      "user_device_type": "MysteryHeat"
    },
    {
      "time": "2018-06-01T05:56:24.215Z",
      "type": "Info",
      "subtype": "NewDeviceFound",
      "icon": "settings",
      "body": "Sense found a new device and named it 'Motor 2'.",
      "destination": "device:cad5432c",
      "device_id": "cad5432c",
      "device_name": "Motor 2",
      "show_action": false,
      "allow_sticky": true,
      "guid": "dc21dd5f-1211-4e34-9766-ecd2d1944727",
      "user_device_type": "MysteryMotor"
    }
  ],
  "user_id": xxx,
  "items": [
    {
      "time": "2018-06-05T21:57:43.086Z",
      "type": "DeviceOn",
      "icon": "heat",
      "body": "{device.name} turned on",
      "destination": "device:qtdquGC6",
      "device_id": "qtdquGC6",
      "device_name": "Heat 4",
      "show_action": false,
      "allow_sticky": false,
      "guid": "b4ac6c21-2767-48b7-97f3-abe0dca1d0d2",
      "user_device_type": "MysteryHeat"
    },
    {
      "time": "2018-06-05T21:57:18.669Z",
      "type": "DeviceOn",
      "icon": "stove",
      "body": "{device.name} turned on",
      "destination": "device:6aa95d8f",
      "device_id": "6aa95d8f",
      "device_name": "Rear left burner",
      "show_action": false,
      "allow_sticky": false,
      "guid": "63952534-e661-480c-8d67-e54badd809cb",
      "user_device_type": "Oven"
    },
    {
      "time": "2018-06-05T21:46:04.276Z",
      "type": "DeviceWasOn",
      "icon": "heat",
      "body": "{device.name} was on for 7 minutes",
      "destination": "device:bdee9e9a",
      "start_time": "2018-06-05T21:38:43.452Z",
      "device_id": "bdee9e9a",
      "device_name": "Water heater",
      "show_action": false,
      "children": [
        {
          "time": "2018-06-05T21:46:04.276Z",
          "type": "DeviceOff",
          "icon": "heat",
          "body": "{device.name} turned off",
          "destination": "device:bdee9e9a",
          "device_id": "bdee9e9a",
          "device_name": "Water heater",
          "show_action": false,
          "allow_sticky": false,
          "location": "Laundry room",
          "guid": "e6173b29-995b-4d20-b7d7-5c3f6d329bd6",
          "user_device_type": "WaterHeater"
        },
        {
          "time": "2018-06-05T21:38:43.452Z",
          "type": "DeviceOn",
          "icon": "heat",
          "body": "{device.name} turned on",
          "destination": "device:bdee9e9a",
          "device_id": "bdee9e9a",
          "device_name": "Water heater",
          "show_action": false,
          "allow_sticky": false,
          "location": "Laundry room",
          "guid": "9bdcadf3-cde7-4803-9a34-9948dca37e10",
          "user_device_type": "WaterHeater"
        }
      ],
      "allow_sticky": false,
      "location": "Laundry room",
      "guid": "9bdcadf3-cde7-4803-9a34-9948dca37e10",
      "user_device_type": "WaterHeater"
    },
    {
      "time": "2018-06-05T21:26:35.914Z",
      "type": "DeviceOff",
      "icon": "heat",
      "body": "{device.name} turned off",
      "destination": "device:gea4Jysu",
      "device_id": "gea4Jysu",
      "device_name": "Baby Bottle Warmer",
      "show_action": false,
      "allow_sticky": false,
      "location": "Bathroom",
      "guid": "49c415cc-abc5-4f7f-b01f-3d8ba0408548",
      "user_device_type": "MysteryHeat"
    },
    {
      "time": "2018-06-05T21:25:06.805Z",
      "type": "DeviceOn",
      "icon": "dishes",
      "body": "{device.name} turned on",
      "destination": "device:ccd27606",
      "device_id": "ccd27606",
      "device_name": "Washer",
      "show_action": false,
      "allow_sticky": false,
      "guid": "88cc8138-1779-40af-be2e-4d3da341319e",
      "user_device_type": "Washer"
    },
    {
      "time": "2018-06-05T21:21:11.167Z",
      "type": "DeviceOn",
      "icon": "washer",
      "body": "{device.name} turned on",
      "destination": "device:11654c89",
      "device_id": "11654c89",
      "device_name": "Dryer",
      "show_action": false,
      "allow_sticky": false,
      "guid": "725f6c18-05f5-422c-b562-eb439501015c",
      "user_device_type": "Dryer"
    },
    {
      "time": "2018-06-05T21:11:37.193Z",
      "type": "DeviceOn",
      "icon": "heat",
      "body": "{device.name} turned on",
      "destination": "device:gea4Jysu",
      "device_id": "gea4Jysu",
      "device_name": "Baby Bottle Warmer",
      "show_action": false,
      "allow_sticky": false,
      "location": "Bathroom",
      "guid": "b7c01d78-5276-41ea-9413-cdbac06e5647",
      "user_device_type": "MysteryHeat"
    }
  ]
}
```

# Disclaimer

This module was developed without the consent of the Sense company, and makes use of an undocumented and unsupported API. Use at your own risk, and be aware that Sense may change the API at any time and break this repository perminantly.
