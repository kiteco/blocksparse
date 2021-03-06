# Kite usage, read first

## TF 1.15.2
###  Manual build
NOTE:
- commands prefixed by `$` should be run in a shell on the host machine
- commands prefixed by `#` should be run in an interactive shell in the docker container

1) Build image
```
$ sudo docker image build -f Dockerfile --rm -t blocksparse:local .
```

2) Start docker container with an interactive terminal, *Choose the relevant CPU/GPU option below*. 
We use the -u flag to ensure that the build artifacts have the correct host user information/permissions.

CPU
- the tests below will fail if you try to run them without GPU support
- the `ln` command should be run inside the docker container
```
$ sudo docker run -it --privileged -w /working_dir -v ${PWD}:/working_dir -u "$(id -u):$(id -g)" --rm blocksparse:local
# ln -s /usr/local/cuda/compat/libcuda.so /usr/lib/libcuda.so
```

GPU
```
$ sudo docker run -it --gpus all --privileged -w /working_dir -v ${PWD}:/working_dir -u "$(id -u):$(id -g)" --rm blocksparse:local
```

3) Compile (inside the docker container)
```
# make compile
```

## TF 1.13.1
###  Manual build
NOTE:
- commands prefixed by `$` should be run in a shell on the host machine
- commands prefixed by `#` should be run in an interactive shell in the docker container

1) Build image
```
$ sudo docker image build -f Dockerfile-tf-1.13.1 --rm -t blocksparse:local-tf.1.13.1 .
```

2) Start docker container with an interactive terminal, *Choose the relevant CPU/GPU option below*. 
We use the -u flag to ensure that the build artifacts have the correct host user information/permissions.

CPU
- the tests below will fail if you try to run them without GPU support
- the `ln` command should be run inside the docker container
```
$ sudo docker run -it --privileged -w /working_dir -v ${PWD}:/working_dir -u "$(id -u):$(id -g)" --rm blocksparse:local-tf.1.13.1
# ln -s /usr/local/cuda/compat/libcuda.so /usr/lib/libcuda.so
```

GPU
```
$ sudo docker run -it --gpus all --privileged -w /working_dir -v ${PWD}:/working_dir -u "$(id -u):$(id -g)" --rm blocksparse:local-tf.1.13.1
```

3) Compile (inside the docker container)
```
# make compile
```

## Testing

1) Exit the docker container we used to build the package, then restart the docker container with root permissions so that
we can do a pip install.

### TF 1.15.2
```
# exit
$ sudo docker run -it --gpus all --privileged -w /working_dir -v ${PWD}:/working_dir --rm blocksparse:local
# pip3 install dist/*.whl
```

### TF 1.13.1
```
# exit
$ sudo docker run -it --gpus all --privileged -w /working_dir -v ${PWD}:/working_dir --rm blocksparse:local-tf.1.13.1
# pip3 install dist/*.whl
```

2) Test compiled version (inside the docker container)
```
# test/blocksparse_matmul_test.py
```

### Current known test failures
- All of the tests noted below still fail on tf.1.13.1 and tf.1.15.2 builds. The exceptions are `test/nccl_test.py` and `test/quantize_test.py` which
  now pass.

## Original known test failures
As far as I can tell, the following tests have always failed on our builds. I determined this by
checking out commit `3ead98d761cd15095fe8198881490d2acbbd0706` and then building using the instructions for
TF 1.13.1 above.
- `test/blocksparse_conv_test.py` -- originally got an error complaining that the `lut` argument was type int32 intstead of int64, then once I fixed this in conv.py via casting,
  the tests failed with 99.9% of the values mismatched.
- `test/blocksparse_matmul_bench.py` -- fails with "got an unexpected keyword argument 'dw_dtype'".
- `test/blocksparse_reduced_dw_test.py` -- fails with "InvalidArgumentError (see above for traceback): Tensorcore GPU required".
- `test/blocksparse_transformer_test.py` -- fails with "InvalidArgumentError (see above for traceback): Tensorcore GPU required".
- `test/edge_bias_test.py` -- hangs and nothing ever happens.
- `test/nccl_test.py` -- fails with "AttributeError: module 'a9a37b9e0fcca4488628a1751af42d7d' has no attribute 'allreduce_nccl'". This actually makes sense since in this
  commit the compile step has all of the nccl stuff commented out.
- `test/quantize_test.py` -- fails with "FileNotFoundError: [Errno 2] No such file or directory: '/home/scott/quant_log.txt'", changed the logfile to `./quant_log.txt` then it works.

## Publishing TF 1.15.2
NOTE: this is typically done outside the docker container or you will need to place github credentials inside the docker container.

- Build the `.so` file using instructions above and add it to github repo.
- Update `setup.py` with the appropriate version number.
- Install `twine` (`pip3 install twine`) and configure it with your PyPi credentials.
- Release: `make release`

# Original README.md below

**Status:** Active (under active development, breaking changes may occur)

# Blocksparse

The `blocksparse` package contains TensorFlow Ops and corresponding GPU kernels for block-sparse matrix multiplication.  Also included are related ops like edge bias, sparse weight norm and layer norm.

To learn more, see [the launch post on the OpenAI blog](https://blog.openai.com/block-sparse-gpu-kernels/).

## Prerequisites

First, you need at least one Nvidia GPU. For best performance, we recommend using a Pascal or Maxwell generation GPU -- this is the full list of features by GPU type:

| GPU Family | BSMatMul-ASM | BSMatMul-CudaC | BSConv |
|------------|------------------------|----------------|--------|
| Kepler | - | X | - |
| Maxwell | X (fastest) | X | X |
| Pascal | X (fastest) | X | X |
| Volta | - | X (fastest) | - |

Note that BSMatMul-CudaC **only supports `feature_axis=0`**, while BSMatMul-ASM only supports `feature_axis=1`.

Additionally, you need:

- A working Linux installation (we run Ubuntu 16.04) with the Nvidia drivers for your GPU.
- CUDA 8 (in `/usr/local/cuda`)
- Python 3.5 or newer, or 2.7 or newer
- TensorFlow 1.4.0 or newer, [with GPU support](https://www.tensorflow.org/install/install_linux#install_tensorflow) (e.g. `pip install tensorflow-gpu`)
- CUDA 9 and Volta will work if you update the build targets (-gencode=arch=compute_70,code=sm_70) and also build tenorflow from source.

## Installation

```
pip install blocksparse
```

## Usage

This example performs a block-sparse matrix multiplication:
```
from blocksparse.matmul import BlocksparseMatMul
import tensorflow as tf
import numpy as np

hidden_size = 4096
block_size = 32
minibatch_size = 64

# Create a (random) sparsity pattern
sparsity = np.random.randint(2, size=(hidden_size//block_size,hidden_size//block_size))

# Initialize the sparse matrix multiplication object
bsmm = BlocksparseMatMul(sparsity, block_size=block_size)

# Input to graph
x = tf.placeholder(tf.float32, shape=[None, hidden_size])

# Initialize block-sparse weights
w = tf.get_variable("w", bsmm.w_shape, dtype=tf.float32)

# Block-sparse matrix multiplication
y = bsmm(x, w)

# Run
sess = tf.InteractiveSession()
sess.run(tf.global_variables_initializer())
result = sess.run([y], feed_dict = {x: np.ones((minibatch_size,hidden_size), dtype='float32')})
print(result)
```

For a more involved example using block-sparse ops to train a language model, see [`examples/`](./examples/).

## Development

If you're interested in hacking on the ops and kernels, go ahead and build from source:

    git clone git@github.com:openai/blocksparse.git
    cd blocksparse

    make compile
    pip install dist/*.whl

    # test it if you like
    test/blocksparse_matmul_test.py
    test/blocksparse_conv_test.py

If your CUDA is not in `/usr/local/cuda` or you have several versions, e.g. both `/usr/local/cuda-8.0` and `/usr/local/cuda-9.0`, set `CUDA_HOME` to the base path to use when compiling `make compile`.


## API Documentation:


### blocksparse.matmul

    class BlocksparseMatMul(object)

        def __init__(self, layout, block_size=32, feature_axis=1)
        """
        layout: a 2d array of ones and zeros specifying the block layout
        block_size: values 32, 16, 8 supported
        feature_axis: when block_size is less than 32 memory access becomes far more efficient
                      with a (C,N) activation layout
        """

        # shape helpers for generating tensors (N=minibatch)
        self.w_shape
        def i_shape(self, N)
        def o_shape(self, N)

        # return the coordinates (c,k) in the layout that corresponds to a given block id
        def block_coord(self, block)

        # experimental ortho init
        def ortho_init(self)

        # in practice, identity_init + layernorm is all you need for initialization
        # with gpu=True the init is performed by kernel on the device
        def identity_init(self, gpu=False)

        # To implement weight normalization.  In practice, layernorm works much better.
        def l2_normalize(self, W, gain=None, epsilon=1e-6, dtype=np.float32)

        def __call__(self, I, W, dw_dtype=tf.float32)
        """
        Execute the op.  Note that the weight variable is independant from the bsmm object.
        This allows multiple weights to be tied to the same bsmm layout.

        dw_dtype: allows control over dw precision format.
        """


    def group_param_grads(param_grad, group_size=8, cast32=True)
    """
    param_grad: the tensorflow parameter gradient for a give bsmm weight variable (returned from tf.gradients)
    group_size: desired group size, up to 8 supported

    This causes the tf graph to be rewritten so that weight grad matmuls from different time steps
    (and shared weights across time) are combined into a more efficient single matmul.
    """


    class SparseProj(object):
        def __init__(self, nhidden, nproj=None, proj_stride=None, block_size=32, gather_lut=None)
        """
        Experimental class to support dense-to-sparse and sparse-to-dense projections.
        Basically the same as the tensorflow ops but faster and support alternate precision formats.
        They assume a unique 1 to 1 mapping so atomics need not be used on backward ops.
        """

        def gather(self, x)
        def scatter(self, x)
        def scatter_add(self, x, y)
        def scatter_mul(self, x, y)



### blocksparse.conv

    class BlocksparseConv(object):
        def __init__(self, BCK, TRS, DHW, MPQ=None, strides=(1,1,1), dilates=(1,1,1), padding="SAME", edge_bias=False)
        """
        BCK: (                                             # block(B)/input(C)/output(K) feature dims
                 ( (c0, c1, c2, ...), (k0, k1, k2, ...) ), # block 0 c,k are indeces into C,K dims
                 ( (c0, c1, c2, ...), (k0, k1, k2, ...) ), # block 1
                 ( (c0, c1, c2, ...), (k0, k1, k2, ...) ), # block 2 ...
             )
        TRS: (T,R,S) or (R,S) or (S,)         - filter spatial size dims
        DHW: (D,H,W) or (H,W) or (W,)         - input image spatial size dims
        MPQ: (M,P,Q) or (P,Q) or (Q,) or None - output image spatial size dims (used for ambiguous dims in strided transpose conv)
        strides: (1,1,1) or (1,1) or (1,)
        dilates: (1,1,1) or (1,1) or (1,)
        padding: (1,1,1) or (1,1) or (1,) or "SAME" or "VALID"
        edge_bias: True/False
        """

        # shape helpers for setting up variables or test tensors
        def edge_bias_shape(self)
        def f_shape(self, block=None)
        def i_shape(self, N)
        def o_shape(self, N)

        # execute op passing in param variables and input
        def __call__(self, F, I, edge_bias=None):

        # for implementing weight norm
        def l2_normalize(self, F, gain=None, epsilon=1e-6, dtype=np.float32):

    class BlocksparseDeconv(BlocksparseConv)
        def __init__(self, BCK, TRS, DHW, MPQ=None, strides=(1,1,1), dilates=(1,1,1), padding="SAME", edge_bias=False)
        """
        Deconvolution.  Same params as above.
        """

    def cwise_linear(x, a=None, b=None)
    """
    In the NCHW tensor format, tensorflow is extremely slow at implementing simple broadcasting ops on the middle C dim.
    This lets you do:
        y = a*x + b
        y = a*x
        y = x + b

    Where a and b are of shape (1,C,1,1)
    This is useful for ops like weight norm.

### blocksparse.ew

    # same as tf ops but generally more efficient and allow custom precision formats
    def        add(x, y, name=None)
    def   multiply(x, y, name=None)
    def   subtract(x, y, name=None)
    def     divide(x, y, name=None)
    def    maximum(x, y, name=None)
    def    minimum(x, y, name=None)

    def   negative(x,    name=None)
    def reciprocal(x,    name=None)
    def     square(x,    name=None)
    def       sqrt(x,    name=None)
    def        exp(x,    name=None)
    def        log(x,    name=None)
    def    sigmoid(x,    name=None)
    def       tanh(x,    name=None)
    def       relu(x,    name=None)
    def       elu (x, alpha=1.0, name=None)

    # here args can be the 4 independant gate tensors or
    # a single merged gate tensor (which gets split in 4 internally)
    def fused_lstm_gates(c, *args, name=None)

    def split4(x)
    def concat4(x0, x1, x2, x3)

    # A custom cast op to help explore novel precision formats
    def float_cast(x, dtype, dx_dtype=None)

    # a much faster (and non-deterministic) dropout op
    # also supports novel precision formats
    def dropout(x, keep_prob=0.8, mask=None)

    # an op to be used in tf.gradients when adding together multiple contributions of a gradient.
    # note that only 8 inputs are supported as you'd never want a single op to consume all possible inputs
    # before it starts executing in the graph (and hence reducing the memory footprint)
    def add_n8(xs, name=None)



### blocksparse.norms

    def layer_norm(x, g, b, axis=1, epsilon=1e-6, relu=False)
    """
    Very fast layernorm to support both bsmm feature_axis activation layouts.
    Also inlcludes optional integrated relu (applied to end)
    """

    # basic batch norm ops for the NCHW layout
    def batch_norm(x, g, b, epsilon=1e-6)
    def batch_norm_inference(x, g, b, m, v, epsilon=1e-6)


