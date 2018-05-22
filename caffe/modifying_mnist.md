# Modifying the Basic MNIST Example

#### Prerequisites

[Basic MNIST Example](https://github.com/cah-icuro/nutshell/blob/master/caffe/basic_mnist.md)

#### Setup

Let's start by cloning the previous project into a new folder to tinker with, so we retain the original as reference.  Enter the new project directory.

#### Solver Parameters

In the previous example, we did 10,000 iterations and ended with a test loss of 0.029.  Did we overtrain, undertrain, or neither?  Change the number of iterations to investigate this.

Changing the max iterations to 20,000 gives me an improved test loss of 0.027, so it looks like this net didn't get enough training initially.

#### Modifying the Network

Find something in the network that you would like to change and check how it affects the network's performance.  For example, a simple question to ask is: How does max pooling compare to average pooling?  If I change layer "pool2" to
```
layer {
  name: "pool2"
  type: "Pooling"
  bottom: "conv2"
  top: "pool2"
  pooling_param {
    pool: AVE
    kernel_size: 2
    stride: 2
  }
}
```

and then retrain, I get slightly worse results:
```
I0522 13:01:52.133872 10273 solver.cpp:418]     Test net output #0: accuracy = 0.9902
I0522 13:01:52.133903 10273 solver.cpp:418]     Test net output #1: loss = 0.0310293
```

If I revert "pool2" to a max pool and set "pool1" to average pooling, I get:
```
I0522 13:09:15.569761 10538 solver.cpp:418]     Test net output #0: accuracy = 0.9908
I0522 13:09:15.569780 10538 solver.cpp:418]     Test net output #1: loss = 0.0308763
```

With both set to `pool: MAX`, I get the best results:
```
I0522 13:10:19.240561 10612 solver.cpp:418]     Test net output #0: accuracy = 0.9906
I0522 13:10:19.240577 10612 solver.cpp:418]     Test net output #1: loss = 0.0278343
```

Thus it looks like max pooling is the best option here.  (Note: training is stochastic so there is some variance in the observed loss).

#### Adding Layers

Currently, we have two fully connected layers at the end of our network.  Let's add one more (also using ReLU as the intermediate nonlinearity), by copying the layers "resu1" and "ip2" and connecting them to the network by modifying the appropriate "top" and "bottom" values.  "bottom" refers to the input (the previous layer), and "top" refers to the output (generally this layer name, except in the case of in-place operations such as ReLU, which feed back the result to the same place it came from).

Currently the first inner product layer has an output size of 500 and the next two have output sized of 10. Let's change the second layer ("ip2") to have an output size of 70.

<details>
  <summary>The changes we have made result in:</summary><p>
  
```
.
.

layer {
  name: "ip1"
  type: "InnerProduct"
  bottom: "pool2"
  top: "ip1"
  param {
    lr_mult: 1
  }
  param {
    lr_mult: 2
  }
  inner_product_param {
    num_output: 500
    weight_filler {
      type: "xavier"
    }
    bias_filler {
      type: "constant"
    }
  }
}
layer {
  name: "relu1"
  type: "ReLU"
  bottom: "ip1"
  top: "ip1"
}
layer {
  name: "ip2"
  type: "InnerProduct"
  bottom: "ip1"
  top: "ip2"
  param {
    lr_mult: 1
  }
  param {
    lr_mult: 2
  }
  inner_product_param {
    num_output: 70
    weight_filler {
      type: "xavier"
    }
    bias_filler {
      type: "constant"
    }
  }
}
layer {
  name: "relu2"
  type: "ReLU"
  bottom: "ip2"
  top: "ip2"
}
layer {
  name: "ip3"
  type: "InnerProduct"
  bottom: "ip2"
  top: "ip3"
  param {
    lr_mult: 1
  }
  param {
    lr_mult: 2
  }
  inner_product_param {
    num_output: 10
    weight_filler {
      type: "xavier"
    }
    bias_filler {
      type: "constant"
    }
  }
}
layer {
  name: "accuracy"
  type: "Accuracy"
  bottom: "ip3"
  bottom: "label"
  top: "accuracy"
  include {
    phase: TEST
  }
}
layer {
  name: "loss"
  type: "SoftmaxWithLoss"
  bottom: "ip3"
  bottom: "label"
  top: "loss"
}
```
</p>
</details>

To verify that you understand the structure of your net, you can now use draw_net on this updated `my_net.prototxt` file.
```
draw_net my_net.prototxt my_net.png
xdg-open my_net.png
```

Note: you can also use this as a syntax check for your network definition, if you have an error, the draw_net script will let you know.

At this point, we can retrain the network with the same script as before:
```bash
bash train_my_net.sh
```
and the output is (running a few separate trials):
```
# First trial
I0522 15:29:16.506413 14020 solver.cpp:418]     Test net output #0: accuracy = 0.9908
I0522 15:29:16.506433 14020 solver.cpp:418]     Test net output #1: loss = 0.0285257

# Second trial
I0522 15:32:33.928989 14171 solver.cpp:418]     Test net output #0: accuracy = 0.991
I0522 15:32:33.929024 14171 solver.cpp:418]     Test net output #1: loss = 0.0273072

# Third trial
I0522 15:34:00.773653 14225 solver.cpp:418]     Test net output #0: accuracy = 0.9903
I0522 15:34:00.773671 14225 solver.cpp:418]     Test net output #1: loss = 0.0298559
```

It seems unlikely that this additional layer provided any benefit in this case, but at least we didn't hard the network's performance.

#### Removing Layers

Just to see how important some layers may be, we could try removing them.  Let's reduce the network so it only has a single inner product layer.  We will experiment to see if having a ReLU layer after the inner product helps or not.

<details>
  <summary>With ReLU</summary><p>
  
```
name: "LeNet"
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
layer {
  name: "conv1"
  type: "Convolution"
  bottom: "data"
  top: "conv1"
  param {
    lr_mult: 1
  }
  param {
    lr_mult: 2
  }
  convolution_param {
    num_output: 20
    kernel_size: 5
    stride: 1
    weight_filler {
      type: "xavier"
    }
    bias_filler {
      type: "constant"
    }
  }
}
layer {
  name: "pool1"
  type: "Pooling"
  bottom: "conv1"
  top: "pool1"
  pooling_param {
    pool: MAX
    kernel_size: 2
    stride: 2
  }
}
layer {
  name: "conv2"
  type: "Convolution"
  bottom: "pool1"
  top: "conv2"
  param {
    lr_mult: 1
  }
  param {
    lr_mult: 2
  }
  convolution_param {
    num_output: 50
    kernel_size: 5
    stride: 1
    weight_filler {
      type: "xavier"
    }
    bias_filler {
      type: "constant"
    }
  }
}
layer {
  name: "pool2"
  type: "Pooling"
  bottom: "conv2"
  top: "pool2"
  pooling_param {
    pool: MAX
    kernel_size: 2
    stride: 2
  }
}
layer {
  name: "ip1"
  type: "InnerProduct"
  bottom: "pool2"
  top: "ip1"
  param {
    lr_mult: 1
  }
  param {
    lr_mult: 2
  }
  inner_product_param {
    num_output: 10
    weight_filler {
      type: "xavier"
    }
    bias_filler {
      type: "constant"
    }
  }
}
layer {
  name: "relu1"
  type: "ReLU"
  bottom: "ip1"
  top: "ip1"
}
layer {
  name: "accuracy"
  type: "Accuracy"
  bottom: "ip1"
  bottom: "label"
  top: "accuracy"
  include {
    phase: TEST
  }
}
layer {
  name: "loss"
  type: "SoftmaxWithLoss"
  bottom: "ip1"
  bottom: "label"
  top: "loss"
}
```
</p>
</details>

Without ReLU: simply delete lines 125-130 containing the definition for the layer "relu1" (no connections need to be modified because ReLUs happen in place)
