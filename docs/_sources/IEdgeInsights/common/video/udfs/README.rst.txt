.. role:: raw-html-m2r(raw)
   :format: html


**Contents**


* `\ ``UDF Configuration`` <http://localhost:7645/IEdgeInsights/common/video/udfs/#udf-configuration>`_
* `\ ``UDF Writing Guide`` <http://localhost:7645/IEdgeInsights/common/video/udfs/#udf-writing-guide>`_
* `\ ``Sample UDFs`` <http://localhost:7645/IEdgeInsights/common/video/udfs/#sample-udfs>`_

  * `\ ``Native UDFs`` <http://localhost:7645/IEdgeInsights/common/video/udfs/#native-udfs>`_
  * `\ ``Python UDFs`` <http://localhost:7645/IEdgeInsights/common/video/udfs/#python-udfs>`_

* `\ ``Construction of Metadata in UDF`` <http://localhost:7645/IEdgeInsights/common/video/udfs/#construction-of-metadata-in-udf>`_
* `\ ``Chaining of UDFs`` <http://localhost:7645/IEdgeInsights/common/video/udfs/#chaining-of-udfs>`_

  * `\ ``Combination of UDFs with ingestors`` <http://localhost:7645/IEdgeInsights/common/video/udfs/#combination-of-udfs-with-ingestors>`_

``EII Sample UDFs``

EII supports loading and executing of native(c++) and python UDFs. In here,
one can find the sample native and python UDFs(User Defined Functions) to be used with EII components
like VideoIngestion and VideoAnalytics. The UDFs can modify the frame, drop the frame and generate meta-data from the frame.

``UDF Configuration``
-------------------------

Below is the JSON schema for UDF json object configuration:

.. code-block:: javascript

   {
     "type": "object",
     "additionalProperties": false,
     "properties": {
       "max_jobs": {
         "description": "Number of queued UDF jobs",
         "type": "integer",
         "default": 20
       },
       "max_workers": {
         "description": "Number of threads acting on queued jobs",
         "type": "integer",
         "default": 4
       },
       "udfs": {
         "description": "Array of UDF config objects",
         "type": "array",
         "items": [
           {
             "description": "UDF config object",
             "type": "object",
             "properties": {
               "type": {
                 "description": "UDF type",
                 "type": "string",
                 "enum": [
                   "native",
                   "python",
                   "raw_native"
                 ]
               },
               "name": {
                 "description": "Unique UDF name",
                 "type": "string"
               },
               "device": {
                 "description": "Device on which inference occurs",
                 "type": "string",
                 "enum": [
                   "CPU",
                   "GPU",
                   "HDDL",
                   "MYRIAD"
                 ]
               }
             },
             "additionalProperties": true,
             "required": [
               "type",
               "name"
             ]
           }
         ]
       }
     }
   }

One can use `JSON validator tool <https://www.jsonschemavalidator.net/>`_ for
validating the UDF configuration object against the above schema.

Example UDF configuration:

.. code-block:: javascript

   {
     "max_jobs": 20,
     "max_workers": 4,
     "udfs": [ {
         "type": "native",
         "name": "dummy"
       },
       {
         "type": "python",
         "name": "pcb.pcb_filter"
       }]
   }

``UDF Writing Guide``
-------------------------

User can refer to `UDF Writing HOW-TO GUIDE <https://github.com/open-edge-insights/video-common/blob/master/udfs/HOWTO_GUIDE_FOR_WRITING_UDF.md>`_ for an detailed explanation of process to write an custom UDF.

``Sample UDFs``
-------------------

.. note:: \ : The UDF config of these go as json objects in the ``udfs`` key in
   the overall UDF configuration object


``Native UDFs``
^^^^^^^^^^^^^^^^^^^


* 
  **Dummy UDF**

  Accepts the frame and forwards the same without doing any processing. It's a
  do-nothing UDF.

  ``UDF config``\ :

  .. code-block:: javascript

     {
         "name": "dummy",
         "type": "native"
     }

* 
  **Resize UDF**

  Accepts the frame, resizes it based on the ``width`` and ``height`` params.

  ``UDF config``\ :

  .. code-block:: javascript

     {
         "name": "resize",
         "type": "native",
         "width": 600,
         "height": 600
     }

* 
  **FPS UDF**

  FPS udf can be used to measure the total number of frames received every second. It can be used in VideoIngestion and VideoAnalytics
  application by adding the below configuration in the udf configuration. It can also be chained with other udfs in which case the FPS result will be affected depending on the other udfs used.

  ``UDF config``\ :

  .. code-block:: javascript

     {
         "name": "fps",
         "type": "native"
     }

  ``Config for chaining fps udf with other udfs``\ :

  .. code-block:: javascript

     "udfs": [{
             "name": "dummy",
             "type": "native"
         },
         {
             "name": "fps",
             "type": "native"
         }]

  ..

     **Note** The fps results will be logged in ``DEBUG`` LOG_LEVEL, added to the metadata with the AppName as the key and will be
     displayed in the visualizer.


* 
  **Sample Realsense UDF**

  Accepts the color and depth frame, converts to rs2::frame type by using rs2::software_device simulation, enables a color filter on the depth frame using rs2::colorizer.

  ``UDF config``\ :

  .. code-block:: javascript

     {
         "name": "sample_realsense",
         "type": "raw_native",
     }

  ----

  ### ``Python UDFs``

.. note:: \ : Additional properties/keys other than ``name`` and ``type`` in the UDF
   config are the parameters of the python UDF constructor



* 
  **Dummy UDF**

  Accepts the frame and forwards the same without doing any processing. It's a
  do-nothing UDF.

  ``UDF config``\ :

  .. code-block:: javascript

     {
         "name": "dummy",
         "type": "python"
     }

* 
  **Jupyter Connector UDF**

  Accepts the frame and publishes it to the EII JupyterNotebook service which processes the frame and publishes it back to the jupyter_connector UDF.

  ``UDF config``\ :

  .. code-block:: javascript

     {
         "name": "jupyter_connector",
         "type": "python"
     }

* 
  **PCB Filter UDF**

  Accepts the frame and based on if ``pcb`` board is at the center in the frame or not,
  it forwards or drops the frame. It basically sends out only the key frames forward
  for further processing and not all frames it receives.

  ``UDF config``\ :

.. code-block:: javascript

     {
         "name": "pcb.pcb_filter",
         "type": "python",
         "training_mode": "false",
         "scale_ratio": 4,
         "n_total_px": 300000,
         "n_left_px": 1000,
         "n_right_px": 1000
     }

  Refer `python/pcb/README.md <https://github.com/open-edge-insights/video-common/blob/master/udfs/python/pcb/README.md>`_ for more information.


* 
  **PCB Classifier UDF**

  Accepts the frame, uses openvino inference engine APIs to determine whether it's
  a ``good`` pcb with no defects or ``bad`` pcb with defects. Metadata associated with
  the frame is populated accordingly.

  ``UDF config``\ :

  .. code-block:: javascript

     {
         "name": "pcb.pcb_classifier",
         "type": "python",
         "device": "CPU",
         "ref_img": "common/video/udfs/python/pcb/ref/ref.png",
         "ref_config_roi": "common/video/udfs/python/pcb/ref/roi_2.json",
         "model_xml": "common/video/udfs/python/pcb/ref/model_2.xml",
         "model_bin": "common/video/udfs/python/pcb/ref/model_2.bin"
     }

  Refer `python/pcb/README.md <https://github.com/open-edge-insights/video-common/blob/master/udfs/python/pcb/README.md>`_ for more information.

  **NOTE**\ :
  The above config works for both "CPU" and "GPU" devices after setting appropriate ``device`` value.
  Please set the "device" value appropriately based on the device used for inferencing.

* 
  **Sample ONNX UDF**

  This UDF mainly demonstrates the model deployment on edge devices via AzureBridge service only.

  Please follow the below steps:


  * Configure the sample ONNX UDF by following `Sample ONNX UDF configuration guide <https://github.com/open-edge-insights/eii-azure-bridge/blob/master/README.md#sample-eii-onnx-8>`_
  * Follow `Single-Node Azure IOT Edge Deployment <https://github.com/open-edge-insights/eii-azure-bridge/blob/master/README.md#single-node-azure-iot-edge-deployment>`_ to deploy the required modules

  For more details on AzureBridge setup, please refer `AzureBridge README.md <https://github.com/open-edge-insights/eii-azure-bridge/blob/master/README.md>`_

----

``Construction of Metadata in UDF``
---------------------------------------

If EII Visualizer/WebVisualizer clients are used for visualizing the classified frames, then please follow the metadata guidelines mentioned in **\ ``Metadata Structure``\ ** in `Visualizer <https://github.com/open-edge-insights/video-native-visualizer/blob/master/README.md>`_ / `WebVisualizer <https://github.com/open-edge-insights/video-web-visualizer/blob/master/README.md>`_ README respectively.

**Note**\ : User has to make sure that the data with in meta data should be of type list, tuple, dict or primitive data types (int, float, string or bool). Also, data with in list, tuple, dict
must contain only primitive data types.
Eg: Any data is of type "numpy.float" or "numpy.int" should be type-casted to float and int respectively.

``Chaining of UDFs``
------------------------

One can chain multiple native/python UDFs in the ``udfs`` key. The way chaining
works here is the output of the UDF listed first would send the modified frame
and metadata to the subsequent UDF and so on. One such classic example is having
``pcb.pcb_filter`` and ``pcb.pcb_classifier`` in VideoIngestion service config to
do both the pre-processing and the classification logic without the need of
VideoAnalytics service.

``Combination of UDFs with ingestors``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. list-table::
   :header-rows: 1

   * - Ingestor
     - Chaining UDFs for pcb demo usecase
     - Chaining UDFs for worker safety gear usecase
   * - opencv/gstreamer
     - :raw-html-m2r:`<br>`\ :raw-html-m2r:`<br>`\ Combination of resize (native), pcb filter (python) and\ :raw-html-m2r:`<br>`\ pcb classifier (python) can be used as :raw-html-m2r:`<br>`\ per the need.
     - :raw-html-m2r:`<br>`\ :raw-html-m2r:`<br>`\ :raw-html-m2r:`<br>`\ Combination of resize (native) and worker safety gear classifier (native/python) :raw-html-m2r:`<br>` can be used as per the need.
   * - gstreamer with GVA(Gstreamer Video Analytics) elements
     - Not Applicable
     - :raw-html-m2r:`<br>`\ Any post-processing UDFs can be used as all\ :raw-html-m2r:`<br>`\ the pre-processing and classification is :raw-html-m2r:`<br>`\ happening in the gstreamer pipeline itself


.. note:: \ :
   Dummy UDF can also be used for above use cases for testing chaining UDFs
   feature but as such there is no value add as it's a do-nothing UDF.
   In DEV Mode python udfs changes can be tested by restarting containers, no need to rebuild.

