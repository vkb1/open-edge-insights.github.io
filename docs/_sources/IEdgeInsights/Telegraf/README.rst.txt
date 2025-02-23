.. role:: raw-html-m2r(raw)
   :format: html


Telegraf
========

Contents:
=========


#. 
   `Telegraf in brief <http://localhost:7645/IEdgeInsights/Telegraf/#Telegraf-in-brief>`_

#. 
   `Telegraf's default configuration <http://localhost:7645/IEdgeInsights/Telegraf/#Telegrafs-default-configuration>`_

#. 
   `MQTT sample configuration and tool to test it <http://localhost:7645/IEdgeInsights/Telegraf/#MQTT-sample-configuration-and-tool-to-test-it>`_

#. 
   `Enabling EII message bus input plugin in Telegraf <http://localhost:7645/IEdgeInsights/Telegraf/#Enabling-EII-message-bus-input-plugin-in-Telegraf>`_

#. 
   `Advanced: Multiple plugin sections of EII message bus input plugin <http://localhost:7645/IEdgeInsights/Telegraf/#Advanced-Multiple-plugin-sections-of-EII-message-bus-input-plugin>`_

#. 
   `Enabling EII message bus Output plugin in Telegraf <http://localhost:7645/IEdgeInsights/Telegraf/#Enabling-EII-message-bus-output-plugin-in-Telegraf>`_

#. 
   `Advanced: Multiple plugin sections of EII message bus output plugin <http://localhost:7645/IEdgeInsights/Telegraf/#Advanced-Multiple-plugin-sections-of-EII-message-bus-output-plugin>`_

#. 
   `Optional: Adding multiple telegraf instances <http://localhost:7645/IEdgeInsights/Telegraf/#Optional-Adding-multiple-telegraf-instances>`_

Telegraf in brief
-----------------

----

Telegraf is a part of TICK stack (https://www.influxdata.com/time-series-platform/). This is a plugin based agent.  Telegraf has many input and output plugins. In EII's basic configuration, it's being used for data ingestion. However EII framework does not restrict Telegraf's any of the features. In EII basic configuration uses Telegraf for data ingestion and sending it to influxdb. 

Telegraf's default configuration
--------------------------------


#. Telegraf starts with the default configuration which is present at `config/Telegraf/Telegraf.conf <https://github.com/open-edge-insights/eii-core/blob/master/config/Telegraf/Telegraf.conf>`_ (for the dev mode the name is 'Telegraf_devmode.conf'). By default the below plugins are enabled.

   * MQTT input plugin (\ **[[inputs.mqtt_consumer]]**\ )
   * EII message bus input plugin (\ **[[inputs.eii_msgbus]]**\ )
   * Influxdb output plugin (\ **[[outputs.influxdb]]**\ )

Telegraf will be started using script 'telegraf_start.py. This script will get the configuration from ETCD first and then it will start the  Telegraf service by picking the right configuration depending on the developer/production mode. By default only single instance of Telegraf container runs (named 'ia_telegraf').

MQTT sample configuration and tool to test it.
----------------------------------------------


* 
  To test with MQTT publisher in k8s helm environment, Please update 'MQTT_BROKER_HOST' Environment Variables in `values.yaml <https://github.com/open-edge-insights/eii-core/blob/master/helm/values.yaml>`_ with HOST IP address of the system where MQTT Broker is running.

* 
  To test with remote mqtt broker in docker environment, Please update 'MQTT_BROKER_HOST' Environment Variables in `docker-compose.yml <https://github.com/open-edge-insights/ts-telegraf/blob/master/docker-compose.yml>`_ with HOST IP address of the system where MQTT Broker is running.

.. code-block::

     ia_telegraf:
       environment:
         ...
         MQTT_BROKER_HOST: '<HOST IP address of the system where MQTT Broker is running>'


* 
  Telegraf Instance can be configured with pressure point data ingestion. In the following example, the MQTT input plugin of Telegraf is configured to read pressure point data and stores into ‘point_pressure_data’ measurement.

  .. code-block::

       # # Read metrics from MQTT topic(s)
       [[inputs.mqtt_consumer]]
       #   ## MQTT broker URLs to be used. The format should be scheme://host:port,
       #   ## schema can be tcp, ssl, or ws.
           servers = ["tcp://localhost:1883"]
       #
       #   ## MQTT QoS, must be 0, 1, or 2
       #   qos = 0
       #   ## Connection timeout for initial connection in seconds
       #   connection_timeout = "30s"
       #
       #   ## Topics to subscribe to
           topics = [
           "pressure/simulated/0",
           ]
           name_override = "point_pressure_data"
           data_format = "json"
       #
       #   # if true, messages that can't be delivered while the subscriber is offline
       #   # will be delivered when it comes back (such as on service restart).
       #   # NOTE: if true, client_id MUST be set
           persistent_session = false
       #   # If empty, a random client ID will be generated.
           client_id = ""
       #
       #   ## username and password to connect MQTT server.
           username = ""
           password = ""

* 
  To start the mqtt-publisher with pressure data,

  .. code-block:: sh

      $ cd ../tools/mqtt/publisher/

   change the command option in `docker-compose.yml <https://github.com/open-edge-insights/eii-core/blob/master/tools/mqtt/publisher/docker-compose.yml>`_ to:

  .. code-block:: sh

      ["--pressure", "10:30"]

   Build and Run mqtt publisher:

  .. code-block:: sh

      $ docker-compose up --build -d

  please refer `tools/mqtt/publisher/README.md <https://github.com/open-edge-insights/eii-tools/blob/master/mqtt/README.md>`_

Enabling EII message bus input plugin in Telegraf
-------------------------------------------------

The purpose of this enablement is to allow telegraf received data from EII message bus and storing it into influxdb that able to be scalable. 

**Overview**

The plugin subscribes to configured topic / topic prefixes. Plugin has component
called subscriber which receives the data from eii message bus.
After receiving the data, depending on configuration, the plugin process
the data, either synchronously or asynchronously. 


* In **synchronous** processing**, the receiver thread (thread which receives the data from EII message bus) is also resposible for the processing of the data (json parsing). After processing the previous data only, the receiver thread process next data available on the EII message bus. 
* In **asynchronous** processing the  receiver thread  receives the data and put it into the queue. There will be pool of threads which will be dequeing the data from the queue and processing it.

**Guidelines for choosing the data processing options are**


* Synchronous option: When the ingestion rate is consistent
* Asynchronous options: There are two options


#. Topic specific queue+threadpool : Frequent spike in ingestion rate for a specific topic
#. Global queue+threadpool : Sometimes spike in ingestion rate for a specific topic

**Configuration of the plugin**

Configuration of the plugin is divided into two parts


#. ETCD configuration
#. Configuration in Telegraf.conf file `config/Telegraf/Telegraf.conf <https://github.com/open-edge-insights/eii-core/blob/master/config/Telegraf/Telegraf.conf>`_

**ETCD configuration**

Since this is eii message bus plugin and so it's part of EII framework,
message bus configuration and plugin's topic specific configuration is kept into etcd.
Below is the sample configuration

.. code-block:: json

   {
       "config": {
           "influxdb": {
               "username": "admin",
               "password": "admin123",
               "dbname": "datain"
           },
           "default": {
               "topics_info": [
                   "topic-pfx1:temperature:10:2",
                   "topic-pfx2:pressure::",
                   "topic-pfx3:humidity"
               ],
               "queue_len": 10,
               "num_worker": 2,
               "profiling": "false"
           }
       },
       "interfaces": {
           "Subscribers": [
               {
                   "Name": "default",
                   "Type": "zmq_tcp",
                   "EndPoint": "ia_zmq_broker:60515",
                   "Topics": [
                       "*"
                   ],
                   "PublisherAppName": "ZmqBroker"
               }
           ]
       }
   }

**Brief description of the configuration**.

Like any other EII service Telegraf has 'config' and 'interfaces'  sections.  "interfaces" are the eii interface details. Let's have more information of "config" section.

config :  Contains the configuration of the influxdb (\ **"influxdb"**\ ) and  eii messagebus input plugin (\ **"default"**\ ). In the above sample configuration, the **"default"** is an instance name. This instance name is referenced from the Telegraf's configuration file `config/Telegraf/Telegraf.conf <https://github.com/open-edge-insights/eii-core/blob/master/config/Telegraf/Telegraf.conf>`_


* 
  topics_info : This is an array of topic prefix configuration, where user specifies, how the data from topic-prefix should be processed. Below is the  way every line in the topic information should be interpreted.


  #. "topic-pfx1:temperature:10:2" : Process data from topic prefix 'topic-pfx1' asynchronously using the dedicated queue of length 10 and dedicated thread pool of size 2. And the processed data will be stored at measurement named 'temperature' in influxdb.
  #. "topic-pfx2:pressure::" : Process data from topic prefix 'topic-pfx2' asynchronously using the global queue and global thread pool. And the processed data will be stored at measurement named 'pressure' in influxdb.
  #. 
     "topic-pfx3:humidity" : Process data synchronously. And the processed data will be stored at measurement named 'humidity' in influxdb.

     *Note : In case, topic specific configuration is not mentioned, then by default, data  gets processed synchronously and measurement name would be, same as topic name.*

* 
  queue_len : Global queue length.

* num_worker : Global thread pool size.
* 
  profiling : This is to enable profiling mode of this plugin (value can be either "true" or "false"). In profiling mode every point will get the following information and will be kept into same measurement as that of point.


  #. Total time spent in plugin (time in ns)
  #. Time spent in queue (in case of asynchronous processing only and time in ns)
  #. Time spend in json processing (time in ns)
  #. 
     The name of the thread pool and the thread id which processed the point.

     *Note: The name of the global thread pool is "GLOBAL". For a topic specific thread pool, the name is "for-$topic-name".*

     .. code-block::

                                                                                                                               |

**Configuration at Telegraf.conf file**

The plugin instance name is an additional key, kept into plugin configuration section. This key is used to fetch the configuration from ETCD. Below is the minimmum, sample configuration with single plugin instance.

.. code-block::

   [[inputs.eii_msgbus]]
   **instance_name = "default"**
   data_format = "json"
   json_strict = true


Here, the value **'default'**  acts as a key in the file **\ `config.json <https://github.com/open-edge-insights/ts-telegraf/blob/master/config.json>`_\ **. For this key, there is configuration in the **'interfaces' and 'config'** sections of the file **\ `config.json <https://github.com/open-edge-insights/ts-telegraf/blob/master/config.json>`_\ **. So the value of\ ** 'instance_name'** acts as a connect/glue between the Telegraf configuration **\ `config.json <https://github.com/open-edge-insights/ts-telegraf/blob/master/config.json>`_\ ** and the **ETCD configuration `config.json <https://github.com/open-edge-insights/ts-telegraf/blob/master/config.json>`_\ **

*Note: 
Since it’s telegraf input plugin, the telegraf’s parser configuration
has to be in Telegraf.conf file. The more information of the telegraf json parser plugin can be be found at https://github.com/influxdata/telegraf/tree/master/plugins/parsers/json.*
In case if there are multiple telegraf instances, then the location of the Telegraf configuration files would be different. For more details please refer the section `Optional: Adding multiple telegraf instance <http://localhost:7645/IEdgeInsights/Telegraf/#Optional-Adding-multiple-telegraf-instances>`_

Advanced: Multiple plugin sections of EII message bus input plugin
------------------------------------------------------------------

Like any other Telegraf plugin user can keep multiple configuration sections of the EII message bus input plugin in the **\ `config/Telegraf/Telegraf.conf <https://github.com/open-edge-insights/eii-core/blob/master/config/Telegraf/Telegraf.conf>`_\ ** file.

Let's have an example for the same. Let's assume there are two EII apps, one with the AppName "EII_APP1" and another with the AppName "EII_APP2", which are publishing the data to eii message bus.
*The Telegraf's ETCD configuration for the same is*

.. code-block:: json

   {
      "config":{
         "subscriber1":{
            "topics_info":[
               "topic-pfx1:temperature:10:2",
               "topic-pfx2:pressure::",
               "topic-pfx3:humidity"
            ],
            "queue_len":100,
            "num_worker":5,
            "profiling":"true"
         },
         "subscriber2":{
            "topics_info":[
               "topic-pfx21:temperature2:10:2",
               "topic-pfx22:pressure2::",
               "topic-pfx23:humidity2"
            ],
            "queue_len":100,
            "num_worker":10,
            "profiling":"true"
         }
      },
      "interfaces":{
         "Subscribers":[
            {
               "Name":"subscriber1",
               "EndPoint":"EII_APP1_container_name:5569",
               "Topics":[
                  "*"
               ],
               "Type":"zmq_tcp",
               "PublisherAppName": "EII_APP1"
            },
            {
               "Name":"subscriber2",
               "EndPoint":"EII_APP2_container_name:5570",
               "Topics":[
                  "topic-pfx21",
                  "topic-pfx22",
                  "topic-pfx23"
               ],
               "Type":"zmq_tcp",
               "PublisherAppName": "EII_APP2"
            }
         ]
      }
   }

*The Telegraf.conf configuration sections for the same is*

.. code-block::

   [[inputs.eii_msgbus]]
   instance_name = "subscriber1"
   data_format = "json"
   json_strict = true

   [[inputs.eii_msgbus]]
   instance_name = "subscriber2"
   data_format = "json"
   json_strict = true


Enabling EII message bus Output plugin in Telegraf
--------------------------------------------------

**Purpose**
Receiving the data from Telegraf Input Plugin and publish data to eii msgbus.

**Configuration of the plugin**
Configuration of the plugin is divided into two parts


#. ETCD configuration
#. Configuration in Telegraf.conf file `config/Telegraf/Telegraf.conf <https://github.com/open-edge-insights/eii-core/blob/master/config/Telegraf/Telegraf.conf>`_

**ETCD configuration**
Since this is eii message bus plugin and so it’s part of EII framework, message bus configuration and plugin’s topic specific configuration is kept into etcd.
Below is the sample configuration

.. code-block:: json

   {
       "config": {
           "publisher": {
               "measurements": ["*"],
               "profiling": "false"
           }
       },
       "interfaces": {
           "Publishers": [
               {
                   "Name": "publisher",
                   "Type": "zmq_tcp",
                   "EndPoint": "0.0.0.0:65077",
                   "Topics": [
                       "*"
                   ],
                   "AllowedClients": [
                       "*"
                   ]
               }
           ]
       }
   }

**Brief description of the configuration**.
Like any other EII service Telegraf has 'config' and 'interfaces'  sections.  "interfaces" are the eii interface details. Let's have more information of "config" section.

config :  Contains eii messagebus output plugin (\ **"publisher"**\ ). In the above sample configuration, the **"publisher"** is an instance name. This instance name is referenced from the Telegraf's configuration file `config/Telegraf/Telegraf.conf <https://github.com/open-edge-insights/eii-core/blob/master/config/Telegraf/Telegraf.conf>`_


* measurements : This is an array of measurements configuration, where user specifies, which measurement data should be published in msgbus.
* profiling : This is to enable profiling mode of this plugin (value can be either "true" or "false").

**Configuration at Telegraf.conf file**

The plugin instance name is an additional key, kept into plugin configuration section. This key is used to fetch the configuration from ETCD. Below is the minimmum, sample configuration with single plugin instance.

.. code-block::

   [[outputs.eii_msgbus]]
   instance_name = "publisher"



Here, the value **'publisher'**  acts as a key in the file **\ `config.json <https://github.com/open-edge-insights/ts-telegraf/blob/master/config.json>`_\ **. For this key, there is configuration in the **'interfaces' and 'config'** sections of the file **\ `config.json <https://github.com/open-edge-insights/ts-telegraf/blob/master/config.json>`_\ **. So the value of\ ** 'instance_name'** acts as a connect/glue between the Telegraf configuration **\ `config.json <https://github.com/open-edge-insights/ts-telegraf/blob/master/config.json>`_\ ** and the **ETCD configuration `config.json <https://github.com/open-edge-insights/ts-telegraf/blob/master/config.json>`_\ **

Advanced: Multiple plugin sections of EII message bus output plugin
-------------------------------------------------------------------

Like any other Telegraf plugin user can keep multiple configuration sections of the EII message bus output plugin in the **\ `config/Telegraf/Telegraf.conf <https://github.com/open-edge-insights/eii-core/blob/master/config/Telegraf/Telegraf.conf>`_\ ** file.

*The Telegraf's ETCD configuration for the same is*

.. code-block:: json

   {
       "config": {
           "publisher1": {
               "measurements": ["*"],
               "profiling": "false"
           },
           "publisher2": {
               "measurements": ["*"],
               "profiling": "false"
           }
       },
       "interfaces": {
           "Publishers": [
               {
                   "Name": "publisher1",
                   "Type": "zmq_tcp",
                   "EndPoint": "0.0.0.0:65077",
                   "Topics": [
                       "*"
                   ],
                   "AllowedClients": [
                       "*"
                   ]
               },
               {
                   "Name": "publisher2",
                   "Type": "zmq_tcp",
                   "EndPoint": "0.0.0.0:65078",
                   "Topics": [
                       "*"
                   ],
                   "AllowedClients": [
                       "*"
                   ]
               }
           ]
       }
   }

*The Telegraf.conf configuration sections for the same is*

.. code-block::

   [[outputs.eii_msgbus]]
   instance_name = "publisher1"

   [[outputs.eii_msgbus]]
   instance_name = "publisher2"


Run Telegraf Input output plugin in IPC mode
--------------------------------------------


* User needs to modify interfaces section of **\ `config.json <https://github.com/open-edge-insights/ts-telegraf/blob/master/config.json>`_\ ** to run in IPC mode as following:

.. code-block::

       "interfaces": {
           "Subscribers": [
               {
                   "Name": "default",
                   "Type": "zmq_ipc",
                   "EndPoint": {
                     "SocketDir": "/EII/sockets",
                     "SocketFile": "backend-socket"
                   },

                   "Topics": [
                       "*"
                   ],
                   "PublisherAppName": "ZmqBroker"
               }
           ],
           "Publishers": [
               {
                   "Name": "publisher",
                   "Type": "zmq_ipc",
                   "EndPoint": {
                     "SocketDir": "/EII/sockets",
                     "SocketFile": "telegraf-out"
                   },
                   "Topics": [
                       "*"
                   ],
                   "AllowedClients": [
                       "*"
                   ]
               }
           ]
       }

Optional: Adding multiple telegraf instances
--------------------------------------------


* User can add multiple instances of Telegarf. For that user needs to add additional environment variable named 'ConfigInstance' in docker-compose.yml file. For  every additional telegraf instance, there has to be additional compose section in the docker-compose.yml file. 
* 
  The configuration for every instance has to be in the telegraf image. The standard to be followed is described as below. 

    For instance named $ConfigInstance the telegraf configuration has to be kept in the repository at 
    `config <https://github.com/open-edge-insights/ts-telegraf/blob/master/config>`_\ /$ConfigInstance/$ConfigInstance.conf (for production mode) and 
    `config <https://github.com/open-edge-insights/ts-telegraf/blob/master/config>`_\ /$ConfigInstance/$ConfigInstance_devmode.conf (for developer mode).

    The same files will be available inside the respective container at
    '/etc/Telegraf/$ConfigInstance/$ConfigInstance.conf' (for production mode)
    '/etc/Telegraf/$ConfigInstance/$ConfigInstance_devmode.conf' (for developer mode)

Let's have some of the examples.

Example1:
For $ConfigInstance = 'Telegraf1'


* The location of the Telegraf configuration would be
  `config <https://github.com/open-edge-insights/ts-telegraf/blob/master/config>`_\ /Telegraf1/Telegraf1.conf (for production mode) and `config <https://github.com/open-edge-insights/ts-telegraf/blob/master/config>`_\ /Telegraf1/Telegraf1_devmode.conf (for developer mode)
* The additional docker compose section which has to be manually added in the file 'docker-compose.yml' would be

.. code-block::

     ia_telegraf1:
       depends_on:
         - ia_common
       build:
         context: $PWD/../Telegraf
         dockerfile: $PWD/../Telegraf/Dockerfile
         args:
           EII_VERSION: ${EII_VERSION}
           EII_UID: ${EII_UID}
           EII_USER_NAME: ${EII_USER_NAME}
           TELEGRAF_SOURCE_TAG: ${TELEGRAF_SOURCE_TAG}
           TELEGRAF_GO_VERSION: ${TELEGRAF_GO_VERSION}
           UBUNTU_IMAGE_VERSION: ${UBUNTU_IMAGE_VERSION}
           CMAKE_INSTALL_PREFIX: ${EII_INSTALL_PATH}
       container_name: ia_telegraf1
       hostname: ia_telegraf1
       image: ${DOCKER_REGISTRY}ia_telegraf:${EII_VERSION}
       restart: unless-stopped
       ipc: "none"
       security_opt:
       - no-new-privileges
       read_only: true
       healthcheck:
         test: ["CMD-SHELL", "exit", "0"]
         interval: 5m
       environment:
         AppName: "Telegraf"
         ConfigInstance: "Telegraf1"
         CertType: "pem,zmq"
         DEV_MODE: ${DEV_MODE}
         no_proxy: "${ETCD_HOST},ia_influxdbconnector"
         NO_PROXY: "${ETCD_HOST},ia_influxdbconnector"
         ETCD_HOST: ${ETCD_HOST}
         ETCD_CLIENT_PORT: ${ETCD_CLIENT_PORT}
         MQTT_BROKER_HOST: 'ia_mqtt_broker'
         INFLUX_SERVER: 'ia_influxdbconnector'
         INFLUXDB_PORT: $INFLUXDB_PORT
         ETCD_PREFIX: ${ETCD_PREFIX}
       networks:
         - eii
       volumes:
         - "vol_temp_telegraf:/tmp/"
         - "vol_eii_socket:${SOCKET_DIR}"
       secrets:
         - ca_etcd
         - etcd_Telegraf_cert
         - etcd_Telegraf_key

.. note:: \ : If user wants to add telegraf output plugin in telegraf instance, modify `docker-compose.yml <https://github.com/open-edge-insights/ts-telegraf/blob/master/docker-compose.yml>`_\ , `docker-compose.yml <https://github.com/open-edge-insights/ts-telegraf/blob/master/docker-compose.yml>`_ and telegraf configuration(.conf) files.


   #. 
      Add publisher configuration in `config.json <https://github.com/open-edge-insights/ts-telegraf/blob/master/config.json>`_\ :

      .. code-block::

         {
            "config": {

                ...,
                "<output plugin instance_name>": {
                    "measurements": ["*"],
                    "profiling": "true"
                }
            },
            "interfaces": {
                ...,
                "Publishers": [
                    ...,
                    {
                        "Name": "<output plugin instance_name>",
                        "Type": "zmq_tcp",
                        "EndPoint": "0.0.0.0:<publisher port>",
                        "Topics": [
                            "*"
                        ],
                        "AllowedClients": [
                            "*"
                        ]
                    }

                ]
            }
         }

      Example:

      .. code-block::

         {
            "config": {

                ...,
                "publisher1": {
                    "measurements": ["*"],
                    "profiling": "true"
                }
            },
            "interfaces": {
                ...,
                "Publishers": [
                    ...,
                    {
                        "Name": "publisher1",
                        "Type": "zmq_tcp",
                        "EndPoint": "0.0.0.0:65078",
                        "Topics": [
                            "*"
                        ],
                        "AllowedClients": [
                            "*"
                        ]
                    }

                ]
            }
         }


      #. 
         Expose "publisher port" in `docker-compose.yml <https://github.com/open-edge-insights/ts-telegraf/blob/master/docker-compose.yml>`_ file:

         .. code-block::

            ia_telegraf<ConfigInstance number>:
            ...
            ports:
              - <publisher port>:<publisher port>

         Example:

         .. code-block::

            ia_telegraf<ConfigInstance number>:
            ...
            ports:
              - 65078:65078

      #. 
         Add eii_msgbus output plugin in telegraf instance config file `config <https://github.com/open-edge-insights/ts-telegraf/blob/master/config>`_\ /$ConfigInstance/$ConfigInstance.conf (for production mode) and 
         `config <https://github.com/open-edge-insights/ts-telegraf/blob/master/config>`_\ /$ConfigInstance/$ConfigInstance_devmode.conf (for developer mode).

         [[outputs.eii_msgbus]]
         instance_name = "\ :raw-html-m2r:`<publisher instance>`\ "

      Example:
      For $ConfigInstance = 'Telegraf1'

      User needs to add following section in `config <https://github.com/open-edge-insights/ts-telegraf/blob/master/config>`_\ /Telegraf1/Telegraf1.conf (for production mode) and `config <https://github.com/open-edge-insights/ts-telegraf/blob/master/config>`_\ /Telegraf1/Telegraf1_devmode.conf (for developer mode) 

       [[outputs.eii_msgbus]]
       instance_name = "publisher1"



* 
  After that, user will need to run the **builder.py** script command to allow the changes of the dockerfile take place.

  .. code-block::

       $ cd [WORK_DIR]/IEdgeInsights/build
       $ python3 builder.py

* 
  User will need to provision, build and bring up all the container again by using below command.

  .. code-block::

       $ cd [WORK_DIR]/IEdgeInsights/build/provision
       $ sudo ./provision.sh ../docker-compose.yml
       $ cd ../
       $ docker-compose -f docker-compose-build.yml build
       $ docker-compose up -d

* 
  Based on above example, user can check the telegraf service will have multiple  container as by using docker ps command.

**Note**\ : It's been practice followed by many users, to keep the configuration in a modular way. One way to achieve the same could be keeping the additional configuration inside 'Telegraf/config/$ConfigInstance/telegraf.d. For example:

create a directory 'telegraf.d' inside 'Telegraf/config/config/$ConfigInstance' :

.. code-block::

      $ mkdir config/$ConfigInstance/telegraf.d
      $ cd config/$ConfigInstance/telegraf.d

keep additional configuration files inside that directory and pass the whole command to start the Telegraf in docker-compose.yml file as following:

.. code-block::

       command: ["telegraf -config=/etc/Telegraf/$ConfigInstance/$ConfigInstance.conf -config-directory=/etc/Telegraf/$ConfigInstance/telegraf.d"]
