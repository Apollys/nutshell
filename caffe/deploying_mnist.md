# Deploying the MNIST Neural Network

If you've trained a neural net on the MNIST dataset, and now you're looking to see it make some new classifications, this should help guide you.

#### Prerequisites

You need to have trained the sample caffe network on MNIST.
* If you did so within the caffe install directory as described [here](http://caffe.berkeleyvision.org/gathered/examples/mnist.html), then you will have to pay close attention to directories and paths to ensure they match your system's structure.
* If you followed [this](https://github.com/cah-icuro/nutshell/blob/master/caffe/basic_mnist.md) procedure, then the steps presented below should apply to your system exactly

#### Prepare Network for Deployment

The structure of inferencing is a little different than that of deployment.  At inference-time, the network won't be given the ground truth labels as you were during training, and thus the network also won't be comparing your inference to the ground truth and computing the resulting loss.  In fact, things are slightly simpler, so we can create a slightly simpler net for deployment.

We will create our deploy network file from a copy of the original net:
```
cp my_net.prototxt my_net_deploy.prototxt
```

Afer opening `my_net_deploy.prototxt`, we first remove the two `type: "Data"` layers at the top of our network file:

```
layer {
  name: "mnist"
  type: "Data"
  top: "data"
  top: "label"
  include {
    phase: TRAIN
  }
  transform_param {
    scale: 0.00390625
  }
  data_param {
    source: "data/mnist_train_lmdb"
    batch_size: 64
    backend: LMDB
  }
}
layer {
  name: "mnist"
  type: "Data"
  top: "data"
  top: "label"
  include {
    phase: TEST
  }
  transform_param {
    scale: 0.00390625
  }
  data_param {
    source: "data/mnist_test_lmdb"
    batch_size: 100
    backend: LMDB
  }
}
```

In their place, we add a `type: "Input"` layer.
```
layer {
    name: "data"
    type: "Input"
    top: "data"
    input_param { shape: { dim: 1 dim: 1 dim: 28 dim: 28 } }
}
```

Notice that although our images are 28 by 28 pixels, we have given the network a 4-dimensional input array.  This will need to match with the input image we pass in from the python script at the end.  The reason four dimensions is a standard for image inference is that two cover the spatial dimensions, one covers the color space, and one covers batch inference.  In our case, the color dimension is 1 since our image is black and white, and our batch size is 1 since we will simply process one image at a time.

Next, we go to the end of the `my_net_deploy.prototxt` and remove the `type: "Accuracy"` layer.  Since this is a comparison to the ground truth label, we certainly won't be able to compute this at inference time.
```
layer {
  name: "accuracy"
  type: "Accuracy"
  bottom: "ip2"
  bottom: "label"
  top: "accuracy"
  include {
    phase: TEST
  }
}
```

Finally, we need to update our `"SoftmaxWithLoss"` layer to simply a `"Softmax"` layer, since we cannot compute loss at inference time.  The new final layer will look like:
```
layer {
  name: "loss"
  type: "Softmax"
  bottom: "ip2"
  top: "loss"
}
```

To more concretely see these changes, you can now run `draw_net` on `my_net_deploy.prototxt` and `my_net.prototxt`, tand then compare the two network diagrams.

#### Making Inferences

Now that your network is ready to deploy, you can make inferences on new images.  Create a folder named `digits` in your current working directory.
* If you wish to try the network on your own image files, make sure they are of size `28x28` pixels
* Alternatively, download the digits I hand-drew in GIMP from [here](https://github.com/cah-icuro/nutshell/tree/master/caffe/digits)

Look over the following python script and save it in your working directory (a level above the `digits` folder) as 'classify_digits.py'.
```python
from __future__ import print_function

import numpy as np
import caffe
import cv2
import os

NETWORK_DEFINITION = 'my_net_deploy.prototxt'
PRETRAINED_WEIGHTS = 'snapshots/my_net_iter_10000.caffemodel'

caffe.set_mode_gpu()
caffe.set_device(0)

# Third parameter either caffe.TRAIN or caffe.TEST
net = caffe.Net(NETWORK_DEFINITION, PRETRAINED_WEIGHTS, caffe.TEST)
print()
print('***********************************************************************')
print('*                           Loaded network                            *')
print('***********************************************************************')
print()

def list_files_of_type(dir, ext):
  filenames = sorted(f for f in os.listdir(dir) if f.endswith('.' + ext))
  return [os.path.join(dir, f) for f in filenames]


IMAGE_DIR = 'digits'
IMAGE_EXT = 'png' # extension without the '.'
image_files = list_files_of_type(IMAGE_DIR, IMAGE_EXT)

for image_path in image_files:
  image = cv2.imread(image_path, cv2.IMREAD_GRAYSCALE)
  if (image.shape != (28, 28)):
    print('An image of incorrect shape was encountered, skipping...')
    print('(Shape must be 28x28)')
    continue
  # Original MNIST data used zero as background during training, for details see:
  # https://stats.stackexchange.com/questions/220164/impact-of-inverting-grayscale-values-on-mnist-dataset
  image = 255 - image
  # Need a 4D array to input into the net
  image_reshaped = image[np.newaxis, np.newaxis, :, :]
  # Set network's input data shape, then pass in input image
  net.blobs['data'].reshape(*image_reshaped.shape)
  net.blobs['data'].data[...] = image_reshaped
  # Inference
  prob = net.forward()
  print('Image:', image_path, 'Classification: ', np.argmax(prob['loss'][0]))

print()
```

Important notes:
* Lines 8, 9: `NETWORK_DEFINITION = ...`, `PRETRAINED_WEIGHTS = ...` - Ensure your network definition and weights filepaths are correct
* Line 28: `IMAGE_EXT = 'png'` - If your test images are not `*.png'` files, update the `IMAGE_EXT` variable accordingly
* Line 39: `image = 255 - image` - If your images are of white digits on a black background, comment out this line.  This line inverts images which start as black digits on white background so that they match MNIST format.  ([More info](https://stats.stackexchange.com/questions/220164/impact-of-inverting-grayscale-values-on-mnist-dataset))

Run `python classify_digits.py` to see your neural net in action!

(Or to be buried under an avalanche of indiscernible errors...  In which case, good luck.)
