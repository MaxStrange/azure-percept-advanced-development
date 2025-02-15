# Azure Percept DK Advanced Development

**Please note!** The experiences in this repository should be considered to be in **preview/beta**.
Significant portions of these experiences are subject to change without warning. **No part of this code should be considered stable**.

Please consider providing feedback
via this [questionnaire](https://forms.office.com/Pages/ResponsePage.aspx?id=v4j5cvGGr0GRqy180BHbRzoJxrXKT0dEvfQyxsA0h8lUMzE0V0pCTFU4UUVSS0xTRUtNT0hZSEs1Ry4u).
Your feedback will help us continue to fine-tune and improve the advanced tools experience.

## Releases

All the releases are tagged with [release notes here](https://github.com/microsoft/azure-percept-advanced-development/tags).

## Overview

This repository holds all the code and documentation for advanced development using the Azure Percept DK.
In this repository, you will find:

* [azureeyemodule](azureeyemodule/README.md): The code for the azureeyemodule, which is the IoT module responsible for running the AI workload
  on the Percept DK.
* [machine-learning-notebooks](machine-learning-notebooks/README.md): Example Python notebooks which show how to train up a
  few example neural networks from scratch (or using transfer learning) and get them onto your device.
* [Model and Data Protection](secured_locker/secured-locker-overview.md): Azure Percept currently supports AI model and data protection as a preview feature.
* [Bring Your Own Model Pipeline Tutorials](tutorials/README.md): Tutorials for how to bring your own custom AI model (and post-processing pipeline) to the device.

## General Workflow

One of the main things that this repository can be used for is to bring your own custom computer vision pipeline
to your Azure Percept DK. The flow for doing that would be this:

1. Use whatever version of whatever DL framework you want (Tensorflow 2.x or 1.x, PyTorch, etc.) to develop your model.
1. Save it to a
   [format that can be converted to OpenVINO IR](https://docs.openvinotoolkit.org/2021.1/openvino_docs_MO_DG_prepare_model_convert_model_Converting_Model.html).
   However, **make sure your ops/layers are supported by OpenVINO 2021.1**.
   See [here for a compatiblity matrix](https://docs.openvinotoolkit.org/2021.1/openvino_docs_MO_DG_prepare_model_Supported_Frameworks_Layers.html).
1. Use OpenVINO to convert it to IR or blob format.
    * I recommend using the [OpenVINO Workbench](https://docs.openvinotoolkit.org/2021.1/workbench_docs_Workbench_DG_Introduction.html)
      to convert your model to OpenVINO IR (or to download a common, pretrained model from their model zoo).
    * You can use the [scripts/run_workbench.sh](scripts/run_workbench.sh) script on Unix systems to run the workbench, or
      [scripts/run_workbench.ps1](scripts/run_workbench.ps1) script on Windows systems.
    * You can use a Docker container to convert IR to blob for our device. See the [scripts/compile_ir.sh](scripts/compile_ir.sh) script
      and use it as a reference (for Windows users: [scripts/compile_ir.ps1](scripts/compile_ir.ps1)).
      Note that you will need to modify it to adjust for if you have multiple output layers in your network.
1. Develop a C++ subclass, using the examples we already have. The Azure Percept DK's azureeyemodule runs a C++ application which
   needs to know about your model in order for your model to run on the device. See the [tutorials](tutorials/README.md) and
   [azureeyemodule](azureeyemodule/README.md) folder for how to do this.
1. The azureeyemodule is the IoT module running on the device responsible for doing inference. It will need to grab your model somehow. For development,
   you could package your model up with your custom azureeyemodule and then have the custom program run it directly. You could also
   have it pull down a model through the module twin (again, see the azureeyemodule folder and tutorials for more details).

See the [tutorials](tutorials/README.md) for detailed walkthroughs on how to do all of this.

Note the following limitations while developing your own custom model for the device:

* [Only certain ops/layers are supported](https://docs.openvinotoolkit.org/2021.1/openvino_docs_MO_DG_prepare_model_Supported_Frameworks_Layers.html).
* Model inputs and outputs must be either unsigned 8 bit integers or 32-bit floating point values - other values are not currently supported
  by the OpenCV G-API.
* The model should be *small*. This is an embedded device for doing inference at the very edge of the IoT network. Its usual use case would be
  to help inform a larger downstream model or to make simple decisions, like whether something moving is a cat or a person - not to do
  instance segmentation over 200 different classes. In particular, while there is no hard and fast limit to model size, I have personally
  found that models larger than 100 MB (after conversion to OpenVINO .blob format) have serious trouble getting loaded onto the device
  and running. This is just a rule of thumb though; the point is to keep your models small.
* The hardware accelerator on the device is an Intel Myriad X. It does not support quantization - only FP32 and FP16 are allowed weight values.
* The VPU is best suited for accelerating convolutional architectures. Having said that, recurrent architectures are supported (as long
  as your ops are in the compatibility matrix), they just aren't very fast. The Optical Character Recognition example model actually
  makes use of an RNN.
* You are not technically limited to a single model. You can bring cascaded models and stitch them into the G-API, but keep in mind
  that this will typically result in slower inferences, especially if you are doing any processing of the outputs from one network
  before feeding into the next, as these intermediate values will need to be offloaded from the VPU back to the ARM chip for computation.
  For an example of how to do this, see the [OCR example model file](./azureeyemodule/app/model/ocr.cpp).

## Model URLs

The Azure Percept DK's azureeeyemodule supports a few AI models out of the box. The default model that runs is Single Shot Detector (SSD),
trained for general object detection on the COCO dataset. But there are a few others that can run without any hassle. Here are the links
for the models that we officially guarantee (because we host them and test them on every release).

To use these models, you can download them through the Azure Percept Studio, or you can paste the URLs into your Module Twin
as the value for "ModelZipUrl". Not all models are found in the Azure Percept Studio.

| Model            | Source            | License           | URL                    |
|------------------|-------------------|-------------------|------------------------|
| Faster RCNN ResNet 50 | [Intel Open Model Zoo](https://docs.openvinotoolkit.org/latest/omz_models_public_faster_rcnn_resnet50_coco_faster_rcnn_resnet50_coco.html) | Apache 2.0 | https://aedsamples.blob.core.windows.net/vision/aeddevkitnew/faster-rcnn-resnet50.zip |
| Open Pose        | [Intel Open Model Zoo](https://docs.openvinotoolkit.org/latest/omz_models_intel_human_pose_estimation_0001_description_human_pose_estimation_0001.html) | Apache 2.0 | https://aedsamples.blob.core.windows.net/vision/aeddevkitnew/openpose.zip |
| Optical Character Recognition | [Intel Open Model Zoo](https://docs.openvinotoolkit.org/2021.1/omz_models_intel_text_detection_0004_description_text_detection_0004.html) and [Intel Open Model Zoo](https://docs.openvinotoolkit.org/2021.1/omz_models_intel_text_recognition_0012_description_text_recognition_0012.html) | Apache 2.0 | https://aedsamples.blob.core.windows.net/vision/aeddevkitnew/ocr.zip |
| Person Detection | [Intel Open Model Zoo](https://docs.openvinotoolkit.org/latest/omz_models_intel_person_detection_retail_0013_description_person_detection_retail_0013.html) | Apache 2.0 | https://aedsamples.blob.core.windows.net/vision/aeddevkitnew/person-detection-retail-0013.zip |
| Product Detection | Custom Vision | Apache 2.0 | https://aedsamples.blob.core.windows.net/vision/aeddevkitnew/product-detection.zip |
| SSD General      | [Intel Open Model Zoo](https://docs.openvinotoolkit.org/latest/omz_models_public_ssdlite_mobilenet_v2_ssdlite_mobilenet_v2.html) | Apache 2.0 |https://aedsamples.blob.core.windows.net/vision/aeddevkitnew/ssdlite-mobilenet-v2.zip
| Tiny YOLOv2 General | [Intel Open Model Zoo](https://docs.openvinotoolkit.org/latest/omz_models_public_yolo_v2_tiny_tf_yolo_v2_tiny_tf.html) | Apache 2.0 | https://aedsamples.blob.core.windows.net/vision/aeddevkitnew/tiny-yolo-v2.zip |
| Unet for Semantic Segmentation of Bananas (for [this notebook](./machine-learning-notebooks/train-from-scratch/SemanticSegmentationUNet.ipynb)) | Trained from scratch | GPLv3 | https://aedsamples.blob.core.windows.net/vision/aeddevkitnew/binary-unet.zip |
| Vehicle Detection | Custom Vision | Apache 2.0 | https://aedsamples.blob.core.windows.net/vision/aeddevkitnew/vehicle-detection.zip |

## Contributing

This repository follows the [Microsoft Code of Conduct](https://opensource.microsoft.com/codeofconduct).

Please see the [CONTRIBUTING.md file](CONTRIBUTING.md) for instructions on how to contribute to this repository.

## Trademark Notice

**Trademarks** This project may contain trademarks or logos for projects, products, or services. Authorized use of
Microsoft trademarks or logos is subject to and must follow [Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.

## Reporting Security Vulnerabilities

Microsoft takes the security of our software products and services seriously, which includes all source code repositories managed through
our GitHub organizations, which include [Microsoft](https://github.com/Microsoft), [Azure](https://github.com/Azure),
[DotNet](https://github.com/dotnet), [AspNet](https://github.com/aspnet), [Xamarin](https://github.com/xamarin),
and [our GitHub organizations](https://opensource.microsoft.com/).

If you believe you have found a security vulnerability in any Microsoft-owned repository that
meets Microsoft's [Microsoft's definition of a security vulnerability](https://docs.microsoft.com/en-us/previous-versions/tn-archive/cc751383(v=technet.10)),
please report it to us as described below.

**Please do not report security vulnerabilities through public GitHub issues.**

Instead, please report them to the Microsoft Security Response Center (MSRC) at [https://msrc.microsoft.com/create-report](https://msrc.microsoft.com/create-report).

If you prefer to submit without logging in, send email to [secure@microsoft.com](mailto:secure@microsoft.com).
If possible, encrypt your message with our PGP key; please download it from
the [Microsoft Security Response Center PGP Key page](https://www.microsoft.com/en-us/msrc/pgp-key-msrc).

You should receive a response within 24 hours. If for some reason you do not, please follow up via email to ensure
we received your original message. Additional information can be found at [microsoft.com/msrc](https://www.microsoft.com/msrc).

Please include the requested information listed below (as much as you can provide) to help us better understand the nature and scope of the possible issue:

  * Type of issue (e.g. buffer overflow, SQL injection, cross-site scripting, etc.)
  * Full paths of source file(s) related to the manifestation of the issue
  * The location of the affected source code (tag/branch/commit or direct URL)
  * Any special configuration required to reproduce the issue
  * Step-by-step instructions to reproduce the issue
  * Proof-of-concept or exploit code (if possible)
  * Impact of the issue, including how an attacker might exploit the issue

This information will help us triage your report more quickly.

If you are reporting for a bug bounty, more complete reports can contribute to a higher
bounty award. Please visit our [Microsoft Bug Bounty Program](https://microsoft.com/msrc/bounty) page for more details about our active programs.

We prefer all communications to be in English.

Microsoft follows the principle of [Coordinated Vulnerability Disclosure](https://www.microsoft.com/en-us/msrc/cvd).
