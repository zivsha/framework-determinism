# TensorFlow Determinism

## Latest News

From TensorFlow version 2.8 onwards, op-determinism is enabled using
`tf.config.experimental.enable_op_determinism`. For more information, see
the [official documentation][9]. You may also want to refer to the
[detailed status](./tensorflow_status.md) of GPU-determinism in TensorFlow in
this current repository.

## Announcements

### Upcoming Patch Changes

In the next release of this package (version `0.4.0`), the distribution name
will be changed from `tensorflow-determinism` to `framework-determinism` and the
package name will be changed from `tfdeterminism` to `fwd9m`. These changes
reflect an intention going forward for this repo to increasingly support
determinism in multiple deep learning frameworks.

Users of `tfdeterminism.patch` will need to use the to-be-deprecated
`fwd9m.tensorflow.patch` and will be encouraged to migrate to using
`fwd9m.tensorflow.enable_determinism` instead, which is intended to provide
compatibility, indefinitely, with future versions of TensorFlow.

There is no ETA for this release. Resources are currently focused on making
various determinism-related changes to stock TensorFlow. In the meantime, you
may want to clone this repo and see if `fwd9m.tensorflow.enable_determinism` in
it's current, unreleased, state (with patching for the segment reduction ops)
does what you need. Please let me know how that goes.

### In-Progress Stock TensorFlow Changes

[RFC: Enabling Determinism in TensorFlow][506] has been accepted. This
formalizes a plan to replace the `TF_DETERMINISTIC_OPS` environment variable
with a `tf.config.enable_deterministic_ops` function and, when determinsitic
ops are enabled, to throw a `tf.errors.UnimplementedError` exception when an op
is used in a way that will introduce nondeterminism into the functionality of
your model.

Determinism-unimplemented exception-throwing (enabled, for now, by
`TF_DETERMINISTIC_OPS`) is being added to some ops, and other ops are receiving
deterministic GPU implementations (enabled, for now, by `TF_DETERMINISTIC_OPS`).
To keep track of what is happening, you may wish to refer to the list of
[pull reqests](#tensorflow-pull-requests) and/or the list of
[confirmed sources and solutions](#confirmed-current-gpu-specific-sources-of-non-determinism-with-solutions)
and associated notes.

## Installation

Note that, currently, you only need to install and use this package if you're
using a version of TensorFlow for which there is a determinism patch available.
There is currently no (officially released) patch available for TensorFlow
versions 2.1 or greater because the effect of the patch that was developed for
earlier versions has been upstreamed into these newer versions.

The next release of this package (version 0.4.0) will include an
`enable_determinism` function that can be applied to any version of TensorFlow
to obtain the latest and best solutions, including any new patches (including
for earlier versions of TensorFlow).

Use `pip` to install:

```
pip install tensorflow-determinism
```

This will install a package that can be imported as `tfdeterminism`. The
installation of `tensorflow-determinism` will not automatically install
TensorFlow. The intention of this is to allow you to install your chosen
version of TensorFlow. You will need to install your chosen version of
TensorFlow before you can import and use `tfdeterminism`.

## Deterministic TensorFlow Solutions

There are currently several ways to access GPU-deterministic op functionality in
TensorFlow for most deep learning applications:

1. Use the lastest version of stock TensorFlow (currently version 2.4), which
   implements most of the currently-available deterministic op solutions. It
   does not require any officially released determinism patches.
2. Clone this repo and call `fwd9m.tensorflow.enable_determinism` to apply the,
   as-yet unreleased, patch for the segment reduction ops to your chosen
   version of TensorFlow. This code may not be fully regression tested; your
   results may vary.
3. Use an NVIDIA NGC TensorFlow Docker image (version >= 19.06).
4. Use version 1.14, 1.15, or 2.0 of stock TensorFlow with GPU support, plus the
   application of `tfdeterminism.patch`. Version 2.1 of stock TensorFlow
   does not have a patch avaliable (and does not require earlier patching) and
   includes many of the deterministic op solutions.

For the latest status of GPU-determinism for different versions of TensorFlow,
see the [tables](#confirmed-current-gpu-specific-sources-of-non-determinism-with-solutions)
below. Although in the short-term, solutions may be deployed as patches for
stock TensorFlow or via the NGC TensorFlow container images, the long-term
intention and plan is to continue upstreaming all solutions into stock
TensorFlow.

### Stock TensorFlow Version 2.4

Stock TensorFlow version 2.4 implements most of the currently-available
GPU-deterministic op solutions. It is missing deterministic
`tf.sparse.sparse_dense_matmul`, which is provided by
[NGC TF Docker image](#nvidia-gpu-cloud-ngc-tensorflow-docker-images) version
`21.04`+.

The following Python code is running on a machine in which `pip` package
`tensorflow=2.4.1` has been installed correctly.

```
import tensorflow as tf
import os
os.environ['TF_DETERMINISTIC_OPS'] = '1'
# Now build your graph and train it
```

Stock TensorFlow version 2.4 with GPU support can be installed as follows:

```
pip install tensorflow=2.4.1
```

The TensorFlow project includes [detailed instructions][3] for installing
TensorFlow with GPU support.

### NVIDIA GPU Cloud (NGC) TensorFlow Docker Images

NGC TensorFlow Docker images, starting with version `19.06`, implement
GPU-deterministic op functionality. Version `19.12` (and beyond) also implements
[multi-algorithm deterministic cuDNN convolutions][1003], which solves the
problem of some layer configurations causing an exception to be thrown with the
message "No algorithm worked!". Version `20.03` (and beyond) also implements
deterministic backprop for bilinear resizing. Version `21.04` (and beyond)
implements deterministic `tf.sparse.sparse_dense_matmul`.

In Python code running inside the container, deterministic ops can be enabled
as follows:

```
import tensorflow as tf
import os
os.environ['TF_DETERMINISTIC_OPS'] = '1'
# Now build your graph and train it
```

The following table shows which version of TensorFlow each NGC Docker image
version is based on:

 NGC TF Image Version | TensorFlow Version |
:---------------------|:-------------------|
 19.06                | 1.13               |
 19.07 - 19.10        | 1.14               |
 19.11 - 20.01        | 1.15 / 2.0         |
 20.02 - 20.03        | 1.15 / 2.1         |
 20.06 - 20.08        | 1.15 / 2.2         |
 20.09 - 20.12        | 1.15 / 2.3         |
 21.02                | 1.15 / 2.4         |

Note that, for now, the NGC TensorFlow container images continue to support
a GPU-performance-optimized TensorFlow API version 1 variant (using a `-tf1`
docker image repository tag), for those who have not yet migrated to TensorFlow
API version 2. The source code for this can be found at
[GitHub/NVIDIA/TensorFlow](https://github.com/NVIDIA/tensorflow).

For information about pulling and running the NVIDIA NGC Docker images, see
[these instructions][2].

### Stock TensorFlow Version < 2.1

Versions 1.14, 1.15, and 2.0 of stock TensorFlow implement a reduced form of
GPU-deterministic op functionality, which must be supplemented with a patch
provided in this repo. The following Python code is running on a machine in
which `pip` package `tensorflow-gpu=2.0.0` has been installed correctly and on
which `tensorflow-determinism` has also been installed (as shown in the
[installation](#installation) section above).

```
import tensorflow as tf
from tfdeterminism import patch
patch()
# use tf as normal
```

Stock TensorFlow with GPU support can be installed as follows:

```
pip install tensorflow-gpu=2.0.4
```

The TensorFlow project includes [detailed instructions][3] for installing
TensorFlow with GPU support.

### Additional Ingredients in the Determinism Recipe

Deterministic op functionality, such as that enabled by
`TF_DETERMINISTIC_OPS=1`, can only contribute to fully-deterministic operation
of a model or training regime in the context of a deterministic system. The
following are notes on various other items that may need to be addressed in
order to ensure that your model trains or infers with prefect reproducibility.

Guidance on obtaining reproducible operation from TensorFlow, as well as the
status as of 2021-08-06, can be found in [these slides][8].

#### Seeds ####

You'll also need to set any and all appropriate random seeds:

```
SEED = 123
random.seed(SEED)
np.random.seed(SEED)
tf.random.set_seed(SEED)
```

You also may need to set the environment variable `PYTHONHASHSEED` to a
reproducible value before you start the python process.

In the TensorFlow version 1 API, `tf.random.set_seed` was `tf.set_random_seed`.

In most models, the effect of setting `tf.random.set_seed` is to ensure that the
trainable variables (the weights and biases) in your model are pseudorandomly
initialized the same way each time. Every time `tf.random.set_seed` is called,
with a particular seed value, the pseudorandom number generator that TensorFlow
uses to initialize the trainable variables is reset ("seeded") deterministically
according to that seed.

If your model is not training deterministically, a good starting point is to
confirm that the trainable variables prior to training are set the same way on
every run, this can be done by calling the following function after calling
`model.compile` and before calling `model.fit`:

```
def summarize_keras_trainable_variables(model, message):
  s = sum(map(lambda x: x.sum(), model.get_weights()))
  print("summary of trainable variables %s: %.13f" % (message, s))
  return s
```

Assuming the the trainable variable are being reproducibly reset, the function
can also be used after training has completed to confirm that the training was
deterministic:

```
model.compile(..)
summarize_keras_trainable_variables(model, "before training")
model.fit(...)
summarize_keras_trainable_variables(model, "after training")
```

An equivalent function can be used for non-keras models. It might also be
preferable to use a hash function rather than sum.

If you're using dropout, which introduces a pseudorandom dropout sequence during
training, then to achieve deterministic results you will need to reset
the pseudorandom number generator that is used to produce those dropout
sequences. The pseudorandom dropout sequence introduced by `tf.keras.Dropout`
layers _will_ be reset by `tf.random.set_seed`.

If you would like to control the seed used for dropout independently of the seed
used for trainable variable initialization, then you can call
`tf.random.set_seed` just before training (e.g. just before calling `model.fit`
but after constructing the model and running `model.compile`). However, if you
would like to explicitly control the seed used for the dropout sequence, then
you can specify it using the `seed` argument of the `tf.keras.layers.Dropout`
constructor.

Note that the `shuffle` argument of the Keras model `fit` method defaults to
`True` (enabled). This pseudorandom shuffling is controlled by a pseudorandom
number generator that is also reset reproducibly by `tf.random.set_seed`,
probably the same pseudorandom number generator that TensorFlow uses throughout.

For more information, including about the relationship between the global seed
(set via `tf.random.set_seed`) and the `seed` parameter that some ops have, see
the documentation for
[`tf.random.set_seed`](https://www.tensorflow.org/api_docs/python/tf/random/set_seed).
This documentation suggests that, for determinism, if `tf.random.set_seed` has
been called then the `seed` parameter of other ops does not need to be set.
This did not used to be true and there may still be instances where the seed
parameter of a given op must be set, to obtain determinism, even after
`tf.random.set_seed` has been called.

#### Dataset Sharding ####

~~If you're using `tf.data.Dataset`, you should not shard the dataset. This
is achieved by either not calling the `shard()` method, or by setting its
`num_shards` parameter to 1.~~

From at least TensorFlow 2.2 onward, if not earlier, `tf.data.Dataset::shard`
appears to operate deterministically.

#### Data-Loader Parallelism ####

When the data-loader pipeline is stateful and is replicated into multiple
asynchronous threads, the threads can interact with each other, resulting in
non-deterministic operation of the overall data-loader functionality. The
most common example of this is when pseudorandom number generation is used in
the data-loader pipeline, such as for data augmentation. When the same
underlying pseudorandom number generator state is used in all the threads, it
will result in non-deterministic functionality. This happens when the default
pseudorandom number generator (e.g. from numpy) is used in the data-loader.
There are three solutions to this:

  1. Make sure that the instances of the objects operating in the parallel
     threads have their own pseudorandom number generator state. For example,
     see [numpy.random.Generator][5].
  2. Generate the pseudorandom augmentation parameters, each set associated with
     each example, in a single thread and then pass them into a stateless,
     parallelized augmentation process.
  3. Run all the data-loader code in only one thread.

How you run all the data-loader code in only one thread depends on how you're
running the data-loader. If you're using `tf.keras.Model::fit()` or
`tf.keras.Model::fit_generator()` then `workers` should be set to not more than
`1`.

Since the validation process runs in the main thread, if the validation
process uses the same stateful data pipeline as the training data-loader then
these two processes will also run in separate threads and you'll wind up with
the same problem. In this case, you need to set `workers` to `0` (zero), which
will cause the data-loader to be run in the main thread.

If you're using `tf.data.Dataset`, it may not be possible to instantiate
different pseudorandom number generator state for each of the parallel calls in
methods such as `map` (controlled by `num_parallel_calls`). A deterministic
solution is to serialize the pseudorandom generation of the data augmentation
parameters (i.e. `num_parallel_calls=1`) and then feed these into a subsequent,
parallelized augmentation stage (or stages). For more information, and example
code, see github/NVIDIA/framework-determinism
[issue 36](https://github.com/NVIDIA/framework-determinism/issues/36).

When deterministic ops are expected, version 2.7 of TensorFlow will force
`num_parallel_calls` to `1` for `map` and `interleave` stages where a `map_func`
contains stateful ops. This may reduce the performance of data-loaders built
using `tf.data.Dataset` that have not been restructured as described above. A
future version of TensorFlow may perform the required restructuring
automatically.

The methods of `tf.data.Dataset` (such as both map and interleave) that have a
`num_parallel_calls` parameter also have a `deterministic` parameter. When set
to `True`, `deterministic` forces the elements to be produced in a repeatable
order when `num_parallel_calls` is greater than 1; when set to `False`, the
constraint is relaxed; if not set, then
`tf.data.Options.experimental_deterministic` (which defaults to `True`) controls
the behavior. Therefore, for determinism, don't set the `deterministic`
parameter in any of the methods and also don't change
`tf.data.Options.experimental_deterministic` from the default.

`tf.image` ops that use pseudorandom number generators (and therefore produce a
different result every time they are executed), such as
`tf.image.sample_distorted_bounding_box`, have stateless versions
(e.g. `tf.image.stateless_sample_distorted_bounding_box`) which will
always produce the same result every time they are executed with the same `seed`
parameter. These ops may be useful in the creation of a deterministic
data-loader. A random number per example could be created and added in a
`tf.data.Dataset` stage with no parallelism (`num_parallel_calls=1`) and then
this random number could be passed as the `seed` parameter of multiple,
different stateless random image ops in later, parallelized
(`num_parallel_calls` > 1) stages.

[`tf.data.experimental.enable_debug_mode`](https://www.tensorflow.org/api_docs/python/tf/data/experimental/enable_debug_mode)
can be called to quickly disable
all potential sources of nondeterminism in subsequently defined `tf.data` input
pipelines when running in eager mode, which makes ruling-out nondeterminism
from a `tf.data` input pipeline easy and fast.

Update for stock TF versions 2.7 and 2.8: functionality has been, or is being,
added to a new `make_deterministic` grappler pass that attempts to modify
instances of `tf.data.Dataloaders` to guarantee deterministic operation when
op-determinism is expected/enabled. Please refer to the
[notes](https://github.com/tensorflow/tensorflow/blob/f888d93b2e1d59432cb22900cb93ea443e8a0c81/tensorflow/core/grappler/optimizers/data/make_deterministic.h#L24-L54)
for the `MakDeterministic` class declaration for more information. The link
provided is for a specific commit point, so reference the notes at the commit
point that is relevant to you.

#### While Loop Parallelism ####

In TF1, the use of `tf.while_loop` when `parallel_iterations` is greater than 1
(note that 10 is the default) may introduce nondeterminism into model
functionality. Additionally, the [AutoGraph Transformations][6], that operate
while compiling code into a graph when (TF2 API) `tf.function` (or the use of
the @tf.function decorator) is used, may lead to [loops][7] being implemented
using `tf.while_loop` and, therefore, parallelized.

The current work-around, to prevent this nondeterminism, is to use
`tf.autograph.experimental.set_loop_options` inside the `for` loop, with
`parallel_iterations=1`.

It has not yet been determined if this non-determinism is specific to
operation on a GPU or if it is a general issue in TensorFlow.

In TF2, the configuration of `parallel_iterations` in a `tf.while_loop` does
*not* affect the order of stateful operations, and therefore `tf.while_loop` can
be used without setting `parallel_iterations` to 1.

#### Gradient Gating ####

In some TensorFlow API interfaces, it is possible to limit the amount of
paralellism that is allowed during back-propagation calculations.

If used, `tf.gradients` (not supported in eager execution) should have its
`gate_gradients` parameter set to `True` (the default is `False`).

The non-Keras, TF1 API optimizers, based on the `tf.compat.v1.train.Optimizer`,
such as `tf.compat.v1.train.AdamOptimizer`, accept a `gate_gradients` parameter
in their `minimize` and `compute_gradient` methods. If this is set to
`tf.compat.v1.train.Optimizer.GATE_NONE` then there is an increased probability
of introducing nondeterminism in the backprop. If not specified,`gate_gradients`
defaults to `tf.compat.v1.train.Optimizer.GATE_OP`, which theoretically could
lead to the introduction of nondeterminism, although I have not yet seen this
happen in a real application. The setting of this parameter that minimizes
parallelism in the backprop calculation (and leads to the lowest performance)
is `tf.compat.v1.train.Optimizer.GATE_GRAPH`. If you've removed all other
sources of nondeterminism and nondeterminism is still being introducted
somewhere, then you could try this setting, if it's available to you. I don't
recommend changing `gate_gradients` to `GATE_GRAPH` as a standard practice.

The Keras optimizers, such as `tf.keras.optimizers.SGD`, which are now the
standard optimizers in the TF2 API, do not offer a `gate_gradients` parameter
in the `minimize` or `get_gradients` methods. There is also no ability to
control gradient gating on `tf.GradientTape` for calculation of gradients in
eager execution.

The reduced availablilty of control of gradient gating in TF2, with eager
execution and an increased reliance on the (often high-level) Keras interface,
doesn't seem to be a real problem with respect to GPU-determinism.

#### Protobufs ####

When serializing a graph into a protobuf, you should pass `True` to the
[`SetSerializationDeterministic`](https://github.com/protocolbuffers/protobuf/blob/a1bb147e96b6f74db6cdf3c3fcb00492472dbbfa/src/google/protobuf/io/coded_stream.h#L834-L860)
method of the `CodedOutputStream` class.

#### Multi-GPU using Horovod ####

If you're using Horovod for multi-GPU training, you may need to disable Tensor
Fusion (assuming that the non-determinism associated with Tensor Fusion has not
yet been resolved, see Horovod [PR 1130][503]):

```
os.environ['HOROVOD_FUSION_THRESHOLD']='0'
```

#### Multi-GPU using tf.distribute Strategies ####

Prior to TensorFlow version 2.3, when using `tf.data.Dataset::shuffle` with
`tf.distribute.MirroredStrategy` (or perhaps any `tf.distribute` strategy),
setting `reshuffle_each_iteration=True` introduces nondeterminism. This
appears to have been fixed in TensorFlow version 2.3. See TF issue
[38197](https://github.com/tensorflow/tensorflow/issues/38197) for more
information.

The last time I checked, the `seed` parameter of the `shuffle` method needed to
be set in order to obtain determinism, although the latest
[documentation](https://www.tensorflow.org/api_docs/python/tf/random/set_seed)
for `tf.random.set_seed` suggests that this should not be necessary as long as
`tf.random.set_seed` has been called.

Creating a `tf.random.Generator` under a `tf.distribute.Strategy::scope()` will
result in different replicas getting different random number streams.

#### CPU ####

If you want to obtain determinism when your ops are running on the CPU, you may
need to limit the number of CPU threads used. In the TF1 API, this can be
acheived as follows:

```
config = tf.compat.v1.ConfigProto(intra_op_parallelism_threads=1,
                                  inter_op_parallelism_threads=1)
with tf.compat.v1.Session(config=config):
  ...
```

In the TF2 API, it can be achieved like this:

```
tf.config.threading.set_intra_op_parallelism_threads(1)
tf.config.threading.set_inter_op_parallelism_threads(1)
```

It should not be necessary to limit the number of CPU threads used when your
ops are not running on the CPU (e.g. when they're running on a GPU).

## Detailed Status of Determinism in TensorFlow and Beyond

Confirmed and likely sources of non-determinism, along with any existing
solutions, are being tracked here.

### GPU-Specific Sources of Non-Determinism

#### Historic GPU-Specific Sources of Non-Determinism

In the past, `tf.math.reduce_sum` and `tf.math.reduce_mean` operated
nondeterministically when running on a GPU. This was resolved before
TensorFlow version 1.12. These ops now function deterministically
by default when running on a GPU.

#### Confirmed Current GPU-Specific Sources of Non-Determinism (With Solutions)

The information in this section has been moved to a separate
[**Status of GPU-Determinism in TensorFlow**](./tensorflow_status.md) page, and
expanded.

#### Other Possible GPU-Specific Sources of Non-Determinism

Going beyond the above-mentioned sources, in the TensorFlow master branch on
2021-02-26, afer release 2.4, the following files call CUDA `atomicAdd` either
directly or indirectly. This makes them candidates for the injection of
nondeterminism.

* `dilation_ops_gpu.cu.cc`
* `maxpooling_op_gpu.cu.cc`
* `multinomial_op_gpu.cu.cc`
* `scatter_functor_gpu.cu.h`
* `stateful_random_ops_gpu.cu.cc`
* `svd_op_gpu.cu.cc`

Unless you are using TensorFlow ops that depend on these files (i.e. ops with
similar names), then your model will not be affected by these potential sources
of non-determinism.

Beyond `atomicAdd`, there are ten other CUDA [atomic functions][4] whose use
could lead to the injection of non-determinism, such as `atomicCAS` (the most
generic, atomic compare and swap). Note also that the word 'atomic' was present
in 167 files in the TensorFlow repo and some of these may be related to the use
of CUDA atomic operations. It's important to remember that it's possible to use
CUDA atomic operations without injecting non-determinism, and that, therefore,
when CUDA atomic operations are present in op code, it doesn't guarantee that
the op injects non-determinism into the computation.

### Sources of Non-Determinism in TensorFlow Unrelated to GPU

* [Issue 29101](https://github.com/tensorflow/tensorflow/issues/29101): Random
  seed not set in graph context of `Dataset#map`. This may have been resolved
  in version 1.14 of TensorFlow.
* `tf.while_loop` with `parallel_iterations` > 1 (default is 10). See
  [While Loop Parallelism](#while-loop-parallelism)

### Sources of Non-Determinism Beyond TensorFlow

* TensorRT timing-based kernel schedule. Each time an inference engine is
  generated, it could be slightly different, particularly if there is varying
  load on the machine used to run TensorRT. There is a solution planned for
  this.
* Horovod Tensor Fusion. Work-around: disable Tensor Fusion by setting the
  environment variable `HOROVOD_FUSION_THRESHOLD` to '0'. See Horovod
  [PR 1130][503].
* Before this project started (in 2018), PyTorch was widely considered to have a
  more complete and coherent GPU determinism story than TensorFlow. At the time
  of writing (2020-02-25), it is no longer clear that one framework is superior
  to the other in this regard. For more information about determinism in
  PyTorch, see the
  [reproducibility documentation](http://bit.ly/pytorch-determinism) in the
  PyTorch repo.

## Relevant Links

This section catalogs relevant links.

### TensorFlow Issues

GitHub issues in the TensorFlow project:

Number                                                         | Title                                                                                    | Date Opened | Status |
--------------------------------------------------------------:|:-----------------------------------------------------------------------------------------|:------------|:-------|
 [2652](https://github.com/tensorflow/tensorflow/issues/2652)  | Backward pass of broadcasting on GPU is non-deterministic                                | 2016-06-03  | Closed |
 [2732](https://github.com/tensorflow/tensorflow/issues/2732)  | Mention that GPU reductions are nondeterministic in docs                                 | 2016-06-08  | Closed |
[13932](https://github.com/tensorflow/tensorflow/issues/13932) | Non-determinism from `tf.data.Dataset.map` with random ops                               | 2017-10-23  | Closed |
[16889](https://github.com/tensorflow/tensorflow/issues/16889) | Problems Getting TensorFlow to behave Deterministically                                  | 2018-02-09  | Open   |
[18037](https://github.com/tensorflow/tensorflow/issues/18037) | tf.sparse_tensor_dense_matmul makes small errors with<br>tf.float32 matrices on GPU      | 2018-03-27  | Closed |
[18096](https://github.com/tensorflow/tensorflow/issues/18096) | Feature Request: Support for configuring deterministic<br>options of cuDNN conv ...      | 2018-03-29  | Open   |
[22398](https://github.com/tensorflow/tensorflow/issues/22398) | CUDA implementation of BiasAddGrad op is non-determinstic                                | 2018-09-19  | Closed |
[29101](https://github.com/tensorflow/tensorflow/issues/29101) | Random seed not set in graph context of `Dataset#map`                                    | 2019-05-28  | Open   |
[38151](https://github.com/tensorflow/tensorflow/issues/38151) | Test deterministic cuDNN CTC loss                                                        | 2020-04-01  | Open   |
[38185](https://github.com/tensorflow/tensorflow/issues/38185) | Add GPU-deterministic back-prop for fused<br>softmax/cross-entropy ops                   | 2020-04-02  | Open   |
[38197](https://github.com/tensorflow/tensorflow/issues/38197) | Model not deterministic, even though<br>os.environ['TF_DETERMINISTIC_OPS'] = '1' set     | 2020-04-03  | Closed |
[39751](https://github.com/tensorflow/tensorflow/issues/39751) | Non-deterministic behaviour: tf.math.unsorted_segment_sum<br>uses CUDA Atomic Operations | 2020-05-21  | Open   |
[40514](https://github.com/tensorflow/tensorflow/issues/40514) | TFBertForSequenceClassification: Non-deterministic when<br>training on GPU ...           | 2020-06-16  | Closed |
[42033](https://github.com/tensorflow/tensorflow/issues/42033) | Add deterministic tf.image.crop_and_resize backprop                                      | 2020-08-04  | Open   |
[47174](https://github.com/tensorflow/tensorflow/issues/47174) | EfficientNet models from TensorFlow.Keras not being<br>reproducible on GPU               | 2021-02-15  | Open   |
[51978](https://github.com/tensorflow/tensorflow/issues/51978) | D9m unimplemented exception for AUC metric and<br>TF_DETERMINISTIC_OPS                   | 2021-09-13  | Closed |
[53771](https://github.com/tensorflow/tensorflow/issues/53771) | Deterministic selection of deterministic cuDNN<br>convolution algos removed in TF 2.5    | 2022-01-14  | Open   |
[53792](https://github.com/tensorflow/tensorflow/issues/53792) | Use enable_op_determinism + Fixed seed + same<br>hardware still get diff results in 2.8. | 2022-01-17  | Open   |
[53846](https://github.com/tensorflow/tensorflow/issues/53846) | tf.data.experimental.sample_from_datasets<br>non-deterministic in multi-GPU              | 2022-01-20  | Open   |
[54259](https://github.com/tensorflow/tensorflow/issues/54259) | Possible issue with tf.data.Dataset in 2.7<br>(but probably Petastorm-related)           | 2022-02-03  | Closed |
[54276](https://github.com/tensorflow/tensorflow/issues/54276) | Deterministic GPU impl of unsorted segment<br>reduction op not available on Windows      | 2022-02-04  | Open   |
[54442](https://github.com/tensorflow/tensorflow/issues/54442) | Reproducible init of trainable variables (TVs)<br>even when the number of TVs changes    | 2022-02-17  | Open   |

### Related Project Issues

GitHub issues in dependent or related projects:

 Project      | Number                                                          | Title                                                                 | Date Opened | Status |
:-------------|----------------------------------------------------------------:|:----------------------------------------------------------------------|:------------|:-------|
 Keras        | [12800](https://github.com/keras-team/keras/issues/12800)       | Unable to get reproducible results using Keras / TF on GPU            | 2019-05-07  | Closed |
 Tensorpack   | [902](https://github.com/tensorpack/tensorpack/issues/902)      | How to run Tensorpack training with deterministic behavior            | 2018-09-20  | Closed |
 transformers | [5603](https://github.com/huggingface/transformers/issues/5063) | Non-deterministic training issue on GPU: TF-BERT                      | 2020-06-16  | Open   |
 ONNX Runtime | [4611](https://github.com/microsoft/onnxruntime/issues/4611)    | Inference on GPU is not deterministic                                 | 2020-07-24  | Closed |

### TensorFlow Pull Requests

The following pull requests (and some inidividual commits) are those in the
TensorFlow GitHub repo (`github.com/tensorflow/tensorflow`) that are directly
related to this project. As we have
[discovered](../scripts/README.md#find-tensorflow-commits), 1.8% of all commits
seem to reference, or have some relationship with, "determinism" or
"deterministic". As of 2020-01-30, that was 1,391 commits.

ID                                                           | Title                                                                  | Status | Date Merged | Version |
------------------------------------------------------------:|:-----------------------------------------------------------------------|:-------|:------------|:--------|
[24747](https://github.com/tensorflow/tensorflow/pull/24747) | Add cuDNN deterministic env variable (only<br>for convolution).        | merged | 2019-01-15  | 1.14    |
[25269](https://github.com/tensorflow/tensorflow/pull/25269) | Add deterministic cuDNN max-pooling                                    | merged | 2019-01-30  | 1.14    |
[25796](https://github.com/tensorflow/tensorflow/pull/25796) | Added tests for `TF_CUDNN_DETERMINISTIC`                               | merged | 2019-02-22  | 1.14    |
[c2790][1001]<sup>1</sup>                                    | Add a decorator to disable autotuning during<br>test executions.       | merged | 2019-03-13  | 1.14    |
[29667](https://github.com/tensorflow/tensorflow/pull/29667) | Add release note about `TF_CUDNN_DETERMINISTIC`                        | merged | 2019-08-06  | 1.14    |
[31389](https://github.com/tensorflow/tensorflow/pull/31389) | Enhance release notes related to<br>`TF_CUDNN_DETERMINISTIC`           | merged | 2019-08-07  | 1.14    |
[31465](https://github.com/tensorflow/tensorflow/pull/31465) | Add GPU-deterministic `tf.nn.bias_add`                                 | merged | 2019-10-17  | 2.1     |
[32979](https://github.com/tensorflow/tensorflow/pull/32979) | Fix typo in release note                                               | closed |             |         |
[33483](https://github.com/tensorflow/tensorflow/pull/33483) | Fix small typo in v2.0.0 release note                                  | merged | 2019-10-25  | 2.1     |
[33803](https://github.com/tensorflow/tensorflow/pull/33803) | Enable tf.nn.bias_add python op tests<br>to work in eager mode         | merged | 2020-02-12  | 2.2     |
[33900](https://github.com/tensorflow/tensorflow/pull/33900) | Address problems with use_deterministic_cudnn<br>test decorator        | merged | 2020-01-09  | 2.2     |
[34887](https://github.com/tensorflow/tensorflow/pull/34887) | Add info about `TF_DETERMINISTIC_OPS` to v2.1<br>release notes         | merged | 2019-12-09  | 2.1     |
[34951][1003]                                                | Add multi-algorithm deterministic cuDNN<br>convolutions                | merged | 2020-01-27  | 2.2     |
[35006](https://github.com/tensorflow/tensorflow/pull/35006) | Fix version 2.1 release note regarding<br>TF_DETERMINISTIC_OPS         | merged | 2019-12-20  | 2.1     |
[e3195][1002]<sup>1</sup>                                    | [XLA/GPU] Convert reduction into tree reduction<br>using padding       | merged | 2020-01-07  | 2.2     |
[8b7a3][1004]<sup>1</sup>                                    | [XLA] Respect TF_DETERMINISTIC_OPS env variable<br>for reductions      | merged | 2020-02-19  | 2.2     |
[37377](https://github.com/tensorflow/tensorflow/pull/37377) | [XLA] follow-up on GPU-deterministic reductions                        | merged | 2020-03-09  | 2.3     |
[9e096][1005]<sup>1</sup>                                    | Use the CUDNN_CTC_LOSS_ALGO_DETERMINISTIC<br>algorithm ...             | merged | 2020-03-10  | 2.3     |
[38089](https://github.com/tensorflow/tensorflow/pull/38089) | Add reminder to test deterministic cuDNN CTC loss                      | closed |             |         |
[38509](https://github.com/tensorflow/tensorflow/pull/38509) | List deterministic op func bug fixes in v2.2<br>release notes          | merged | 2020-04-15  | 2.2     |
[39243](https://github.com/tensorflow/tensorflow/pull/39243) | GPU-deterministic tf.image.resize (bilinear)                           | merged | 2020-09-22  | 2.4     |
[44717](https://github.com/tensorflow/tensorflow/pull/44717) | Add to rel notes: deterministic<br>tf.image.resize (bilinear)          | merged | 2020-11-13  | 2.4     |
[47419](https://github.com/tensorflow/tensorflow/pull/47419) | Support all fp types in GPU SparseTensorDenseMatMul                    | merged | 2021-03-08  | 2.5     |
[47749](https://github.com/tensorflow/tensorflow/pull/47749) | Add GPU determinisim for fp types in GPU<br>SparseTensorDenseMatMul    | closed |             |         |
[47772](https://github.com/tensorflow/tensorflow/pull/47772) | Add segment reduction op exceptions for<br>GPU determinism             | merged | 2021-03-18  | 2.5     |
[47925](https://github.com/tensorflow/tensorflow/pull/47925) | Add softmax/cross-entropy op exceptions for<br>GPU determinism         | merged | 2021-04-05  | 2.6     |
[47974](https://github.com/tensorflow/tensorflow/pull/47974) | Add GPU implem of sparse segment reduction<br>ops                      | merged | 2021-05-05  | 2.6     |
[48581](https://github.com/tensorflow/tensorflow/pull/48581) | Update release notes in branch r2.5                                    | closed |             |         |
[48688](https://github.com/tensorflow/tensorflow/pull/48688) | Add CPU-focused tests for fused<br>softmax/cross-entropy ops           | merged | 2021-04-26  | 2.6     |
[48905](https://github.com/tensorflow/tensorflow/pull/48905) | Add GPU excepts, CPU d9m, and tests to<br>crop_and_resize              | merged | 2021-05-13  | 2.6     |
[49178](https://github.com/tensorflow/tensorflow/pull/49178) | Add non-sparse softmax/xent GPU-determinism                            | merged | 2021-06-04  | 2.6     |
[50070](https://github.com/tensorflow/tensorflow/pull/50070) | Add sparse softmax/xent GPU-determinism                                | merged | 2021-09-10  | 2.7     |
[50135](https://github.com/tensorflow/tensorflow/pull/50135) | Factor core/kernels RequireDeterminism() into<br>library               | merged | 2021-06-09  | 2.6     |
[50355](https://github.com/tensorflow/tensorflow/pull/50355) | Add d9m-unimplemented exceptions to sparse/sparse<br>matmul            | merged | 2021-06-23  | 2.6     |
[50505](https://github.com/tensorflow/tensorflow/pull/50505) | Add d9m-unimplemented exception-throwing to fused<br>batch-norm        | merged | 2021-07-08  | 2.7     |
[50640](https://github.com/tensorflow/tensorflow/pull/50640) | Enhance r2.6 release notes                                             | merged | 2021-07-08  | 2.6     |
[0f7b1][1006]<sup>1</sup>                                    | Add internal function to enable/disable op determinism                 | merged | 2021-07-26  | 2.7     |
[51023](https://github.com/tensorflow/tensorflow/pull/51023) | Add unimplemented exception to nearest-neighbor<br>resizing            | merged | 2021-08-02  | 2.7     |
[a4b53][1008]<sup>1</sup>                                    | Raise error if random ops used with determinism<br>without seed        | merged | 2021-08-10  | 2.7     |
[51140](https://github.com/tensorflow/tensorflow/pull/51140) | Add unimplemented exception to<br>tf.image.adjust_contrast             | merged | 2021-08-19  | 2.7     |
[51392](https://github.com/tensorflow/tensorflow/pull/51392) | Add GPU-deterministic segment reductions                               | closed |             |         |
[5a51f][1012]<sup>1</sup>                                    | Add determinism checks & tests for<br>DebugNumericSummaryV2            | merged | 2021-08-31  | 2.7     |
[fc91e][1007]<sup>1</sup>                                    | Add make_deterministic grappler pass                                   | merged | 2021-09-03  | 2.7     |
[51861](https://github.com/tensorflow/tensorflow/pull/51861) | Replacement for 51392 (w/ deterministic kernels<br>optionally enabled) | merged | 2021-09-07  | 2.7     |
[eb95f][1013]<sup>1</sup>                                    | Add d9m checks+tests for ScatterNd,<br>ScatterNdUpdate, & TensorScatter| merged | 2021-09-09  | 2.7     |
[51920](https://github.com/tensorflow/tensorflow/pull/51920) | Add d9m-unimplemented exception for<br>tf.nn.depthwise_conv2d          | merged | 2021-09-15  | 2.7     |
[03ba3][1014]<sup>1</sup>                                    | Make GPU scatter ND ops deterministic by running them on CPU           | merged | 2021-09-17  | 2.7     |
[c0e2e][1009]<sup>1</sup>                                    | Handle MapAndBatch in make_deterministic<br>grappler pass              | merged | 2021-09-10  | 2.7     |
[f0e6c][1011]<sup>1</sup>                                    | Add determinism exception to DenseBincount                             | merged | 2021-09-22  | 2.7     |
[52227](https://github.com/tensorflow/tensorflow/pull/52227) | Add determinism tests for tf.nn.ctc_loss                               | merged | 2021-10-05  | 2.7     |
[52971](https://github.com/tensorflow/tensorflow/pull/52971) | Add op-determinism info to version 2.7<br>release notes                | merged | 2021-11-10  | 2.7     |
[4dc6d][1015]<sup>1</sup>                                    | Have GpuScatterExpander always rewrite<br>Scatter when op-d9m enabled  | merged | 2021-12-16  | 2.8     |
[53465](https://github.com/tensorflow/tensorflow/pull/53465) | Add v2.8 release notes                                                 | merged | 2021-12-22  | 2.8     |
[ced76][1010]<sup>1</sup>                                    | Fix issue where convolutions were not<br>deterministic                 | merged | 2022-01-19  | 2.8     |
[53826](https://github.com/tensorflow/tensorflow/pull/53826) | r2.8 cherry-pick req: Fix issue where<br>convs were not deterministic  | merged | 2022-01-20  | 2.8     |
[53847](https://github.com/tensorflow/tensorflow/pull/53847) | Add release note about deterministic<br>selection of conv algos        | merged | 2022-01-21  | 2.8     |
[54119](https://github.com/tensorflow/tensorflow/pull/54119) | Add disable for depthwise-conv d9m-unimplemented<br>exception          | merged | 2022-02-01  | 2.9     |
[55657](https://github.com/tensorflow/tensorflow/pull/55657) | Add GPU-determinism to tf.nn.depthwise_conv2d                          | merged | 2022-04-26  | 2.10    |

Notes:
  1. These are individual commits.

[1001]: https://github.com/tensorflow/tensorflow/commit/c27909ea80e8823dbf4f7176ab69991a630356a1
[1002]: https://github.com/tensorflow/tensorflow/commit/e31955d9fb34ae7273354dc2347ba99eea8c5280
[1003]: https://github.com/tensorflow/tensorflow/pull/34951
[1004]: https://github.com/tensorflow/tensorflow/commit/8b7a3db0b6e09415b5640be4986fb4d7c6e5209a
[1005]: https://github.com/tensorflow/tensorflow/commit/9e096debc4a0909deb69970f38bee7b77e5e5f7d
[1006]: https://github.com/tensorflow/tensorflow/commit/0f7b1c29cd24727992f355b196f827e6d4235684
[1007]: https://github.com/tensorflow/tensorflow/commit/fc91e5caf3e9c807de4b0312ef456d7b57e1a876
[1008]: https://github.com/tensorflow/tensorflow/commit/a4b53af710f1e1dc41279e2de7cf6ec8b092f28b
[1009]: https://github.com/tensorflow/tensorflow/commit/c0e2e65e155fabd9bdfd5c41b4b816d9efac8e1f
[1010]: https://github.com/tensorflow/tensorflow/commit/ced762bd4986dfad83ccf6d57e4e4fa3e47bd3fe
[1011]: https://github.com/tensorflow/tensorflow/commit/f0e6c2ff82d9892861572dd0802ff926a99b6320
[1012]: https://github.com/tensorflow/tensorflow/commit/5a51fa12e54fd61ca7520fa82ad0cdd1c14591ee
[1013]: https://github.com/tensorflow/tensorflow/commit/eb95f6157f949cd848910cb63f8cd5b1767af5d2
[1014]: https://github.com/tensorflow/tensorflow/commit/03ba364effe173d0b185977f0c14a48863d1f277
[1015]: https://github.com/tensorflow/tensorflow/commit/4dc6d4d59008b4558040afa1a5a5993f583bb48e

### Other TensorFlow Organization Pull Requests

These are relevant pull requests against repositories in
`github.com/tensorflow` other than `github.com/tensorflow/tensorflow`

 Repository | Number                                                   | Title                                                                 | Date Opened | Status |
:-----------|---------------------------------------------------------:|:----------------------------------------------------------------------|:------------|:-------|
 community  | [346](https://github.com/tensorflow/community/pull/346)  | RFC: Enhancing determinism in TF                                      | 2021-01-19  | merged |
 community  | [370](https://github.com/tensorflow/community/pull/370)  | RFC: [determinism] Improve list of ops in                             | 2021-03-19  | merged |
 community  | [386](https://github.com/tensorflow/community/pull/386)  | RFC: [determinism] Add tf.nn.depthwise_conv2d to op list in           | 2021-04-28  | open   |

### PyTorch Pull Requests

ID                                                     | Title                                                         | Status | Date Merged | Version |
------------------------------------------------------:|:--------------------------------------------------------------|:-------|:------------|:--------|
[33795](https://github.com/pytorch/pytorch/pull/33795) | Enhance reproducibility documentation                         | merged | 2020-03-06  | 1.5     |

### Horovod Pull Requests

ID                                                     | Title                                                         | Status | Date Merged | Version |
------------------------------------------------------:|:--------------------------------------------------------------|:-------|:------------|:--------|
 [1130][503]                                           | Add grouped allreduce feature                                 | Open   |             |         |

### Miscellaneous

* TensorFlow [RFC: Enabling Determinism in TensorFlow][506]
* [Gradient injection][505] in the testing of op backprop determinism in
  TensorFlow tests.
* Two Sigma: [A Workaround for Non-Determinism in
  TensorFlow](http://bit.ly/two-sigma-determinism)
* Chainer [PR 2710](https://github.com/chainer/chainer/pull/2710): cuDNN
  Deterministic mode
* SE / Stack Overflow: [Tensorflow: Different results with the same random seed][501]
* SE / Stack Overflow: [Are tensorflow random values guaranteed to be the same inside a single run? (comment)][502]
* SE / Data Science: [Making Keras + Tensorflow code execution deterministic on a GPU][504]

[501]: https://stackoverflow.com/questions/54047654/tensorflow-different-results-with-the-same-random-seed
[502]: https://stackoverflow.com/questions/52213325/are-tensorflow-random-values-guaranteed-to-be-the-same-inside-a-single-run#comment91376212_52213325
[503]: https://github.com/horovod/horovod/pull/1130
[504]: https://datascience.stackexchange.com/questions/14812/making-keras-tensorflow-code-execution-deterministic-on-a-gpu/
[505]: ../test/gradient_injection.md
[506]: https://github.com/tensorflow/community/blob/master/rfcs/20210119-determinism.md

[1]: http://bit.ly/determinism-in-deep-learning
[2]: https://ngc.nvidia.com/catalog/containers/nvidia:tensorflow
[3]: https://www.tensorflow.org/install/gpu
[4]: https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#atomic-functions
[5]: https://numpy.org/doc/stable/reference/random/generator.html
[6]: https://www.tensorflow.org/guide/function#autograph_transformations
[7]: https://www.tensorflow.org/guide/function#loops
[8]: https://bit.ly/dl-determinism-slides-v3
[9]: https://www.tensorflow.org/api_docs/python/tf/config/experimental/enable_op_determinism
