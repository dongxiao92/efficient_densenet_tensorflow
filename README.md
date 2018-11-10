# efficient_densenet_tensorflow
A Tensorflow 1.9+ implementation of DenseNet-121, optimized to save GPU memory.

Based on the following repo's code:

https://github.com/Jiankai-Sun/Distributed-TensorFlow-Example/tree/master/CIFAR-10

https://github.com/titu1994/keras-squeeze-excite-network


## Motivation
While DenseNets are fairly easy to implement in deep learning frameworks, most
implementations (such as the [original](https://github.com/liuzhuang13/DenseNet)) tend to be memory-hungry.
In particular, the number of intermediate feature maps generated by batch normalization and concatenation operations
grows quadratically with network depth.

*It is worth emphasizing that this is not a property inherent to DenseNets, but rather to the implementation.*

This implementation uses a new strategy to reduce the memory consumption of DenseNets.
It is based on [efficient_densenet_pytorch](https://github.com/gpleiss/efficient_densenet_pytorch).
It makes use of [checkpointing intermeditate features](https://www.tensorflow.org/versions/r1.5/api_docs/python/tf/contrib/layers/recompute_grad) and
[alternate approach](https://github.com/openai/gradient-checkpointing).

This adds 15-20% of time overhead for training, but **reduces feature map consumption from quadratic to linear.**

For more details, please see the [technical report](https://arxiv.org/pdf/1707.06990.pdf).

## How to checkpoint
Currently all of the dense layers are checkpointed, however you can alter the implementation to trade of speed and memory.
For example by checkpointing earlier layers you remove intermediate checkpoints which are generally larger earlier on due to the
pooling layers.

However more strategies can be found in the [alternate approach](https://github.com/openai/gradient-checkpointing).

## Example setup for a 12gb Nvidia GPU
`python train.py --batch_size 6000 --efficient True`

`python train.py --batch_size 3750`

## Main piece of code:
> models/densenet_creator.py#116
```
        def _x(ip):
            x = batch_normalization(ip, **self.bn_kwargs)
            x = tf.nn.relu(x)

            if self.bottleneck:
                inter_channel = nb_filter * 4

                x = conv2d(x, inter_channel, (1, 1), kernel_initializer='he_normal', padding='same', use_bias=False,
                           **self.conv_kwargs)
                x = batch_normalization(x, **self.bn_kwargs)
                x = tf.nn.relu(x)

            x = conv2d(x, nb_filter, (3, 3), kernel_initializer='he_normal', padding='same', use_bias=False,
                       **self.conv_kwargs)

            if self.dropout_rate:
                x = dropout(x, self.dropout_rate, training=self.training)

            return x

        if self.efficient:
            # Gradient checkpoint the layer
            _x = tf.contrib.layers.recompute_grad(_x)

```

## Requirement
- Tensorflow 1.9+
- Horovod

## Usage
If you care about speed, and memory is not an option, pass the `efficient=False` argument into the `DenseNet` constructor.
Otherwise, pass in `efficient=True`.

Important Options:
- `--batch_size` (int) - The number of images per batch (default 8000)

- `--fp16` (bool) - Whether to run with FP16 or not  (default False)

- `--efficient` (bool) - Whether to run with gradient checkpointing or not (default False)


## Reference

```
@article{pleiss2017memory,
  title={Memory-Efficient Implementation of DenseNets},
  author={Pleiss, Geoff and Chen, Danlu and Huang, Gao and Li, Tongcheng and van der Maaten, Laurens and Weinberger, Kilian Q},
  journal={arXiv preprint arXiv:1707.06990},
  year={2017}
}
```