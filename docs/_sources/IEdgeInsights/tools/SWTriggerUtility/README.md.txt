# Software Trigger Utility for VideoIngestion Module

This utility is used for invoking various software trigger features of VideoIngestion. The currently supported triggers to VideoIngestion module are:
1. START INGESTION - to start the ingestor
2. STOP_INGESTION -  to stop the ingestor
3. SNAPSHOT - to get frame snapshot which feeds one only frame into the video data pipeline.

## Software Trigger Utilily pre-requisites

SWTriggerUtility expects a set of config, interfaces & public private keys to be present in ETCD as a pre-requisite.
* To achieve this, please ensure an entry for SWTriggerUtility with its relative path from [IEdgeInsights](../../) directory is set in any of the .yml files present in [build/usecases](../../build/usecases) directory.
    * An example has been provided below to add the entry in [video-streaming.yml](../../build/usecases/video-streaming.yml)
    ```yml
        AppContexts:
        ---snip---
        - tools/SWTriggerUtility
    ```

## Configuration file:

**config.json** is the configuration file used for sw_trigger_utility.

|       Field      | Meaning |                                       Type of the value                                    |
| :-------------:  | :-----: | ------------------------------------------------------------------------------------ |
| `num_of_cycles`  | `Number of cyles of start-stop ingestions to repeat`   | `integer`                           |
| `dev_mode`       | `dev mode ON or OFF`   | `boolean (true or false)`  |
| `log_level`      | `Log level to view the logs accordingly`   |  `integer [DEBUG=3 (default), ERROR=0, WARN=1, INFO=2]`  |

> **Note**:

> * When working with GigE cameras which requires `network_mode: host`, update the `EndPoint` key of SWTriggerUtility interface in [config.json](./config.json) to have the host system IP instead of the service name of the server.

    Eg. In order to connect to `ia_video_ingestion` service which is configured with GigE camera refer the below `EndPoint` change in the SWTriggerUtility interface:

    ```javascript
    {
        "Clients": [
            {
                "EndPoint": "<HOST_SYSTEM_IP>:64013",
                "Name": "default",
                "ServerAppName": "VideoIngestion",
                "Type": "zmq_tcp"
            }
        ]
    }
    ```

> * In case one needs to change the values in [config.json](./config.json), then ensure to re-run the steps mentioned in [pre-requisites section](#software-trigger-utilily-pre-requisites) to see the updated changes are getting applied `OR` one can choose to update the config key of SWTriggerUtility app via ETCD UI and then restart the application.


**This utility works in both dev & prod mode.**  As a pre-requisite make sure to turn ON the flag corresponding to "dev_mode" to true/false in the config.json file.
## Running Software Trigger Utility

1. EII services can be running in prod or dev mode by setting `DEV_MODE` value accordingly in [build/.env](../../build/.env)

2. Execute builder.py script:
   ```sh
   $ cd [WORKDIR]/IEdgeInsights/build/
   $ python3 builder.py -f usecases/video-streaming.yml
   ```

   **NOTE**: The same yml file to which the **SWTriggerUtility** entry was added in [pre-requisites](#software-trigger-utilily-pre-requisites) has to be selected while running [builder](../../build/builder.py)

3. Run provisioning step as below:

   ```sh
   $ cd [WORKDIR]/IEdgeInsights/build/provision
   $ sudo -E ./provision.sh  ../docker-compose.yml
   ```
## Usage of Software Trigger Utility:

By default the SW Trigger Utility container will not execute anything and one needs to interact with the running container to generate the trigger commands. Make sure VI service is up and ready to process the commands from the utility.

Software trigger utility can be used in following ways:

1: "START INGESTION" -> "Allows ingestion for default time (120 seconds being default)" -> "STOP INGESTION"

    ```sh
    $ cd [WORKDIR]/IEdgeInsights/build
    $ docker exec ia_sw_trigger_utility ./sw_trigger_utility
    ```

2: "START INGESTION" -> "Allows ingestion for user defined time (configurable time in seconds)" -> "STOP INGESTION"

    ```sh
    $ cd [WORKDIR]/IEdgeInsights/build
    $ docker exec ia_sw_trigger_utility ./sw_trigger_utility 300
    ```
    **Note**: In the above example, VideoIngestion starts then does ingestion for 300 seconds then stops ingestion after 300 seconds & cycle repeats for number of cycles configured in the config.json.


3: Selectively send START_INGESTION software trigger:

    ```sh
    $ cd [WORKDIR]/IEdgeInsights/build
    $ docker exec ia_sw_trigger_utility ./sw_trigger_utility START_INGESTION
    ```

4: Selectively send STOP_INGESTION software trigger:

    ```sh
    $ cd [WORKDIR]/IEdgeInsights/build
    $ docker exec ia_sw_trigger_utility ./sw_trigger_utility STOP_INGESTION
    ```

5: Selectively send SNAPSHOT software trigger:

    ```sh
    $ cd [WORKDIR]/IEdgeInsights/build
    $ docker exec ia_sw_trigger_utility ./sw_trigger_utility SNAPSHOT
    ```
> **Note**:  

* If duplicate START_INGESTION or STOP_INGESTION sw_triggers are sent by client by mistake then the VI is capable  of catching these duplicates & responds back to client conveying that duplicate triggers were sent & requets to send proper sw_triggers. 

* In order to send SNAPSHOT trigger, ensure that the ingestion is stopped. In case START_INGESTION trigger is sent previously then stop the ingestion using the STOP_INGESTION trigger.
