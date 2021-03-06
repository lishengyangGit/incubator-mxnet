# Importing an ONNX model into MXNet

In this tutorial we will:

- learn how to load a pre-trained ONNX model file into MXNet.
- run inference in MXNet.

## Prerequisites
This example assumes that the following python packages are installed:
- [mxnet](http://mxnet.incubator.apache.org/install/index.html)
- [onnx](https://github.com/onnx/onnx) (follow the install guide)
- Pillow - A Python Image Processing package and is required for input pre-processing. It can be installed with ```pip install Pillow```.
- matplotlib


```python
from PIL import Image
import numpy as np
import mxnet as mx
import mxnet.contrib.onnx as onnx_mxnet
from mxnet.test_utils import download
from matplotlib.pyplot import imshow
```

### Fetching the required files


```python
img_url = 'https://s3.amazonaws.com/onnx-mxnet/examples/super_res_input.jpg'
download(img_url, 'super_res_input.jpg')
model_url = 'https://s3.amazonaws.com/onnx-mxnet/examples/super_resolution.onnx'
onnx_model_file = download(model_url, 'super_resolution.onnx')
```

## Loading the model into MXNet

To completely describe a pre-trained model in MXNet, we need two elements: a symbolic graph, containing the model's network definition, and a binary file containing the model weights. You can import the ONNX model and get the symbol and parameters objects using ``import_model`` API. The paameter object is split into argument parameters and auxilliary parameters.


```python
sym, arg, aux = onnx_mxnet.import_model(onnx_model_file)
```

We can now visualize the imported model (graphviz needs to be installed)


```python
mx.viz.plot_network(sym, node_attrs={"shape":"oval","fixedsize":"false"})
```




![svg](https://s3.amazonaws.com/onnx-mxnet/examples/super_res_mxnet_model.png)



## Input Pre-processing

We will transform the previously downloaded input image into an input tensor.


```python
img = Image.open('super_res_input.jpg').resize((224, 224))
img_ycbcr = img.convert("YCbCr")
img_y, img_cb, img_cr = img_ycbcr.split()
test_image = np.array(img_y)[np.newaxis, np.newaxis, :, :]
```

## Run Inference using MXNet's Module API

We will use MXNet's Module API to run the inference. For this we will need to create the module, bind it to the input data and assign the loaded weights from the two parameter objects - argument parameters and auxilliary parameters.


```python
mod = mx.mod.Module(symbol=sym, data_names=['input_0'], context=mx.cpu(), label_names=None)
mod.bind(for_training=False, data_shapes=[('input_0',test_image.shape)], label_shapes=None)
mod.set_params(arg_params=arg, aux_params=aux, allow_missing=True, allow_extra=True)
```

Module API's forward method requires batch of data as input. We will prepare the data in that format and feed it to the forward method.


```python
from collections import namedtuple
Batch = namedtuple('Batch', ['data'])

# forward on the provided data batch
mod.forward(Batch([mx.nd.array(test_image)]))
```

To get the output of previous forward computation, you use ``module.get_outputs()`` method.
It returns an ``ndarray`` that we convert to a ``numpy`` array and then to Pillow's image format


```python
output = mod.get_outputs()[0][0][0]
img_out_y = Image.fromarray(np.uint8((output.asnumpy().clip(0, 255)), mode='L'))
result_img = Image.merge(
"YCbCr", [
                img_out_y,
                img_cb.resize(img_out_y.size, Image.BICUBIC),
                img_cr.resize(img_out_y.size, Image.BICUBIC)
]).convert("RGB")
result_img.save("super_res_output.jpg")
```

Here's the input image and the resulting output images compared. As you can see, the model was able to increase the spatial resolution from ``256x256`` to ``672x672``.

| Input Image | Output Image |
| ----------- | ------------ |
| ![input](https://raw.githubusercontent.com/dmlc/web-data/master/mxnet/doc/tutorials/onnx/images/super_res_input.jpg?raw=true) | ![output](https://raw.githubusercontent.com/dmlc/web-data/master/mxnet/doc/tutorials/onnx/images/super_res_output.jpg?raw=true) |

<!-- INSERT SOURCE DOWNLOAD BUTTONS -->