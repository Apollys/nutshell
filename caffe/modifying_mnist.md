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
p>
</details>
