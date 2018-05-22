# Basic Caffe Example with MNIST

#### Prerequisites

To make things simpler, locate your top-level caffe directory and export it to $CAFFE_ROOT (in .bashrc or just the current terminal), for example:
```
export CAFFE_ROOT=/home/username/caffe
```

This sample will be a more hands-on extension of the [official](http://caffe.berkeleyvision.org/gathered/examples/mnist.html) introductory tutorial.  As in that tutorial, we must start by obtaining the MNIST dataset as follows:

```bash
cd $CAFFE_ROOT
./data/mnist/get_mnist.sh
./examples/mnist/create_mnist.sh
```

This creates two data folders:
```
$CAFFE_ROOT/examples/mnist/mnist_test_lmdb
$CAFFE_ROOT/examples/mnist/mnist_train_lmdb
```

We now create a directory entirely outside of caffe and enter that directory.

#### Get Data

We will copy the data from the caffe directory to a `data` subdirectory of our project location:
```
mkdir data
cp -r $CAFFE_ROOT/examples/mnist/mnist_train_lmdb data/
cp -r $CAFFE_ROOT/examples/mnist/mnist_test_lmdb data/
```


#### The Network

We will start by creating the network definition [my_net.prototxt](https://raw.githubusercontent.com/cah-icuro/nutshell/master/caffe/my_net.prototxt).  The structure of this network is explained in more detail [here](http://caffe.berkeleyvision.org/gathered/examples/mnist.html).  Note lines `14` and `31` reference our train and test data locations, respectively:
```
    source: "data/mnist_train_lmdb"
.
.
    source: "data/mnist_test_lmdb"
```

Next, we need to create a solver definition, [my_solver.prototxt](https://raw.githubusercontent.com/cah-icuro/nutshell/master/caffe/my_solver.prototxt).  There are two important lines referencing file locations in the solver definition, on lines 2 and 29:
```
net: "my_net.prototxt"
.
.
snapshot_prefix: "snapshots/my_net"
```

We have decided to save snapshots in a subdirectory `snapshots` prefixed by `my_net`.  Be aware that if the path doesn't exist, caffe will exit with an error rather than creating the directory!  So we need to do:
```
mkdir snapshots
```

#### Training

Now we are ready to train.  For convenience, you can save the training command to a shell script by running
```
echo "set -e
$CAFFE_ROOT/build/tools/caffe train --solver=my_solver.prototxt $@" >> train_my_net.sh
```

Once you're ready to train, simply run:
```
bash train_my_net.sh
```

The terminal should show output as caffe first initializes the solver according to `my_solver.prototxt` and then initializes the net according to `my_net.prototxt`.  Once initialized, it will go through the stochastic gradient descent optimization process.  Note the learning rate goes down throughout the optimization process.  Hopefully the loss goes down as well.
