<div>
  <h1>AI Sample Depth Estimation</h1>
  <p align="center">
  </p>
</div>

![depth result](./resource/depth_result.gif)

---

## 👋 Overview

The depth estimation demo is a Python-based ROS 2 pipeline that runs the Depth Anything V2 model on the Qualcomm AI accelerator through QNN.

- It takes an RGB image as input, either from a local file such as `input_image.jpg` published by the `image_publisher` node, or from the ROS topic `cam0_stream1` published by the QRB ROS Camera (`qrb_ros_camera`).
- It preprocesses the image, sends it to the QRB ROS NN inference node, and postprocesses the output tensor.
- The result is published on the `depth_map` ROS topic as an image containing per-pixel depth values.

![architecture](./resource/depth_estimation_architecture.jpg)

| Node Name | Function |
| --------- | -------- |
| `qrb_ros_camera` | Qualcomm ROS 2 package that captures images with configurable parameters and publishes them to ROS topics. |
| `image_publisher` | Publishes image data from a local file to a ROS topic, used when no camera is connected. |
| `sample_depth_estimation` | Subscribes to input images for preprocessing, then performs postprocessing on the output tensor published by the QRB ROS NN inference node. |
| `qrb_ros_nn_inference` | Loads a trained AI model, receives preprocessed tensors, performs inference, and publishes the results. |

## 🔎 Table of contents

- [👋 Overview](#-overview)
- [🔎 Table of contents](#-table-of-contents)
- [⚓ Used ROS Topics](#-used-ros-topics)
- [🚀 Out-of-Box Usage](#-out-of-box-usage)
  - [Prerequisites](#prerequisites)
  - [Run on device](#run-on-device)
- [👨‍💻Visualization:](#visualization)
- [👨‍💻 Build from source](#-build-from-source)
- [❔ FAQs](#-faqs)
- [📜 License](#-license)

## ⚓ Used ROS Topics

All pipeline nodes run in the `sample_container` namespace, except `image_publisher`, which publishes on the global `/image_raw` topic.

| ROS Topic | Type | Description |
| --------- | ---- | ----------- |
| `/image_raw` | `<sensor_msgs/msg/Image>` | Input image published by the `image_publisher` node |
| `/sample_container/cam0_stream1` | `<sensor_msgs/msg/Image>` | Input image stream published by `qrb_ros_camera` |
| `/sample_container/qrb_inference_input_tensor` | `<qrb_ros_tensor_list_msgs/msg/TensorList>` | Preprocessed input tensor sent to the NN inference node |
| `/sample_container/qrb_inference_output_tensor` | `<qrb_ros_tensor_list_msgs/msg/TensorList>` | Raw model output tensor published by the NN inference node |
| `/sample_container/depth_map` | `<sensor_msgs/msg/Image>` | Depth map result |


## 🚀 Out-of-Box Usage

### Prerequisites

- The AI model is installed to `/opt/model/Depth-Anything-V2.bin` when the Debian package is installed. If that file is missing, download the `Depth-Anything-V2.bin` model to `/opt/model/` manually before running the sample.

  ```bash
  sudo mkdir -p /opt/model && cd /opt/model
  sudo wget https://huggingface.co/qualcomm/Depth-Anything-V2/resolve/19ce3645e11de17eed7e869eebcc07dd352834f3/Depth-Anything-V2.bin?download=true -O Depth-Anything-V2.bin
  ```

- Export the NN inference required variables.

  ```bash
  export ADSP_LIBRARY_PATH="/usr/lib/rfsa/adsp;/usr/lib/rfsa/adsp/hexagon-v81"
  export CDSP_LIBRARY_PATH="/vendor/dsp/cdsp0;/usr/lib/rfsa/adsp/hexagon-v81"
  ```

### Run on device

1. Run the sample with a local image file:

   ```bash
   source /opt/ros/jazzy/setup.bash
   ros2 launch sample_depth_estimation launch_with_image_publisher.py
   ```

   With the default parameters, this launch script publishes the built-in `input_image.jpg` at 10 Hz.

2. You can override the image file and the model path:

   ```bash
   ros2 launch sample_depth_estimation launch_with_image_publisher.py image_path:=<your local image path> model_path:=<your local model path>
   ```

3. If a GMSL camera is connected, you can launch the sample with `qrb_ros_camera` instead:

   ```bash
   source /opt/ros/jazzy/setup.bash
   ros2 launch sample_depth_estimation launch_with_qrb_ros_camera_args.py
   ```

## 👨‍💻Visualization: 

You can then check the ROS topic `/sample_container/depth_map` in `rqt`. Please refer to the ROS 2 Jazzy official documentation to install rqt.

## 👨‍💻 Build from source

Source code is located at `sources/quic-qrb-ros/qrb_ros_samples/sample_depth_estimation/` in the downstream Ubuntu workspace.

1. Build the package:

   ```bash
   cd build-utils/ubuntu/
   python3 build.py --gen-debians --package ros-jazzy-sample-depth-estimation
   ```

   Built `.deb` files are output to:

   ```text
   <workspace>/debian_packages/oss/ros-jazzy-sample-depth-estimation/
   ```

2. Copy the `.deb` file to the target device:

   ```bash
   scp <workspace>/debian_packages/oss/ros-jazzy-sample-depth-estimation/ros-jazzy-sample-depth-estimation_*.deb <user>@<device-ip>:~
   ```

3. Install the `.deb` package on the target device:

   ```bash
   sudo apt install ./ros-jazzy-sample-depth-estimation_*.deb
   ```

Refer to [Usage](#-usage) to run the depth estimation sample.

## ❔ FAQs

<details>
<summary>How can I change the image publishing rate?</summary><br>
The input rate is set by the <code>rate</code> parameter of the <code>image_publisher</code> node in <code>launch/launch_with_image_publisher.py</code>. Change the <code>10.0</code> value to publish at a different frequency:

```python
image_path_arg = DeclareLaunchArgument(
    'image_path',
    default_value=os.path.join(package_path, "resource", "input_image.jpg"),
    description='Path to the input image file'
)

# Node for image_publisher
image_publish_node = Node(
    package='image_publisher',
    executable='image_publisher_node',
    name='image_publisher_node',
    output='screen',
    parameters=[
        {'filename': image_path},
        {'rate': 10.0},  # Set the publishing rate to 10 Hz
    ],
)
```

When launching with `qrb_ros_camera`, set the `fps` value of `stream1` in `launch/launch_with_qrb_ros_camera.py` instead.
</details>

<details>
<summary>How can I get the raw output of the QNN inference node?</summary><br>
Comment out the following code in <code>depth_estimation_node.py</code> to get the raw output of the QNN inference node:

```python
# Normalize to [0,255]
normalized = cv2.normalize(output_image, None, 0, 255, cv2.NORM_MINMAX)
colored = cv2.applyColorMap(normalized.astype(np.uint8), cv2.COLORMAP_INFERNO)
```
</details>

<details>
<summary>The sample starts but no depth map is published.</summary><br>
Check that the model file exists at <code>/opt/model/Depth-Anything-V2.bin</code>, or pass a valid path with the <code>model_path</code> launch argument. Also confirm the input topic is active with <code>ros2 topic hz /image_raw</code> (local image) or <code>ros2 topic hz /sample_container/cam0_stream1</code> (camera).
</details>

## 📜 License

Project is licensed under the BSD-3-Clause-Clear License. See [LICENSE](../../LICENSE) for the full license text.
