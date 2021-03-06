.. testsetup:: *

    from pytorch_lightning.core.lightning import LightningModule
    from pytorch_lightning.core.datamodule import LightningDataModule
    from pytorch_lightning.trainer.trainer import Trainer
    import os
    import torch
    from torch.nn import functional as F
    from torch.utils.data import DataLoader
    from torch.utils.data import DataLoader
    import pytorch_lightning as pl
    from torch.utils.data import random_split

.. _new_project:

####################
Lightning in 2 steps
####################

**In this guide we'll show you how to organize your PyTorch code into Lightning in 2 steps.**

Organizing your code with PyTorch Lightning makes your code:

* Keep all the flexibility (this is all pure PyTorch), but removes a ton of boilerplate
* More readable by decoupling the research code from the engineering
* Easier to reproduce
* Less error prone by automating most of the training loop and tricky engineering
* Scalable to any hardware without changing your model

----------

Here's a 2 minute conversion guide for PyTorch projects:

.. raw:: html

    <video width="100%" controls autoplay muted playsinline src="https://pl-bolts-doc-images.s3.us-east-2.amazonaws.com/pl_docs/pl_quick_start_full.m4v"></video>

----------

*********************************
Step 0: Install PyTorch Lightning
*********************************


You can install using `pip <https://pypi.org/project/pytorch-lightning/>`_

.. code-block:: bash

    pip install pytorch-lightning

Or with `conda <https://anaconda.org/conda-forge/pytorch-lightning>`_ (see how to install conda `here <https://docs.conda.io/projects/conda/en/latest/user-guide/install/>`_):

.. code-block:: bash

    conda install pytorch-lightning -c conda-forge

You could also use conda environments

.. code-block:: bash

    conda activate my_env
    pip install pytorch-lightning

----------

Import the following:

.. code-block:: python

    import os
    import torch
    from torch import nn
    import torch.nn.functional as F
    from torchvision.datasets import MNIST
    from torchvision import transforms
    from torch.utils.data import DataLoader
    import pytorch_lightning as pl
    from torch.utils.data import random_split

******************************
Step 1: Define LightningModule
******************************

.. code-block::

    class LitAutoEncoder(pl.LightningModule):

        def __init__(self):
            super().__init__()
            self.encoder = nn.Sequential(
                nn.Linear(28*28, 64),
                nn.ReLU(),
                nn.Linear(64, 3)
            )
            self.decoder = nn.Sequential(
                nn.Linear(3, 64),
                nn.ReLU(),
                nn.Linear(64, 28*28)
            )

        def forward(self, x):
            # in lightning, forward defines the prediction/inference actions
            embedding = self.encoder(x)
            return embedding

        def training_step(self, batch, batch_idx):
            # training_step defined the train loop. It is independent of forward
            x, y = batch
            x = x.view(x.size(0), -1)
            z = self.encoder(x)
            x_hat = self.decoder(z)
            loss = F.mse_loss(x_hat, x)
            # Logging to TensorBoard by default
            self.log('train_loss', loss)
            return loss

        def configure_optimizers(self):
            optimizer = torch.optim.Adam(self.parameters(), lr=1e-3)
            return optimizer

A :class:`~pytorch_lightning.core.LightningModule` defines a *system* such as:

- `Autoencoder <https://github.com/PyTorchLightning/pytorch-lightning-bolts/blob/master/pl_bolts/models/autoencoders/basic_ae/basic_ae_module.py>`_
- `BERT <https://colab.research.google.com/drive/1F_RNcHzTfFuQf-LeKvSlud6x7jXYkG31#scrollTo=yr7eaxkF-djf>`_
- `DQN <https://colab.research.google.com/drive/1F_RNcHzTfFuQf-LeKvSlud6x7jXYkG31#scrollTo=IAlT0-75T_Kv>`_
- `GAN <https://github.com/PyTorchLightning/pytorch-lightning-bolts/blob/master/pl_bolts/models/gans/basic/basic_gan_module.py>`_
- `Image classifier <https://colab.research.google.com/drive/1F_RNcHzTfFuQf-LeKvSlud6x7jXYkG31#scrollTo=gEulmrbxwaYL>`_
- Seq2seq 
- `SimCLR <https://github.com/PyTorchLightning/pytorch-lightning-bolts/blob/master/pl_bolts/models/self_supervised/simclr/simclr_module.py>`_
- `VAE <https://github.com/PyTorchLightning/pytorch-lightning-bolts/blob/master/pl_bolts/models/autoencoders/basic_vae/basic_vae_module.py>`_

It is a :class:`torch.nn.Module` that groups all research code into a single file to make it self-contained:

- The Train loop
- The Validation loop
- The Test loop
- The Model + system architecture
- The Optimizer

You can customize any part of training (such as the backward pass) by overriding any
of the 20+ hooks found in :ref:`hooks`

.. code-block:: python

    class LitAutoEncoder(pl.LightningModule):

        def backward(self, trainer, loss, optimizer, optimizer_idx):
            loss.backward()

In Lightning, training_step defines the train loop and is independent of forward. Use forward to define
what happens during inference/predictions

.. code-block:: python

    def forward(...):
        # how you want your model to do inference/predictions

    def training_step(...):
        # the train loop INDEPENDENT of forward.

More details in :ref:`lightning_module` docs.

----------

**********************************
Step 2: Fit with Lightning Trainer
**********************************

First, define the data however you want. Lightning just needs a :class:`~torch.utils.data.DataLoader` for the train/val/test splits.

.. code-block:: python

    dataset = MNIST(os.getcwd(), download=True, transform=transforms.ToTensor())
    train_loader = DataLoader(dataset)
    
Next, init the :class:`~pytorch_lightning.core.LightningModule` and the PyTorch Lightning :class:`~pytorch_lightning.trainer.Trainer`,
then call fit with both the data and model.

.. code-block:: python

    # init model
    autoencoder = LitAutoEncoder()

    # most basic trainer, uses good defaults (auto-tensorboard, checkpoints, logs, and more)
    # trainer = pl.Trainer(gpus=8) (if you have GPUs)
    trainer = pl.Trainer()
    trainer.fit(autoencoder, train_loader)

The :class:`~pytorch_lightning.trainer.Trainer` automates:

* Epoch and batch iteration
* Calling of optimizer.step(), backward, zero_grad()
* Calling of .eval(), enabling/disabling grads
* :ref:`weights_loading`
* Tensorboard (see :ref:`loggers` options)
* :ref:`multi_gpu` support
* :ref:`tpu`
* :ref:`amp` support

-----------

*****************
Predict or Deploy
*****************
When you're done training, you have 3 options to use your LightningModule for predictions.

Option 1: Sub-models
====================
Pull out any model inside your system for predictions.

.. code-block:: python

    # ----------------------------------
    # to use as embedding extractor
    # ----------------------------------
    autoencoder = LitAutoEncoder.load_from_checkpoint('path/to/checkpoint_file.ckpt')
    encoder_model = autoencoder.encoder
    encoder_model.eval()

    # ----------------------------------
    # to use as image generator
    # ----------------------------------
    decoder_model = autoencoder.decoder
    decoder_model.eval()

Option 2: Forward
=================
You can also add a forward method to do predictions however you want.

.. code-block:: python

    # ----------------------------------
    # using the AE to extract embeddings
    # ----------------------------------
    class LitAutoEncoder(pl.LightningModule):
        def forward(self, x):
            embedding = self.encoder(x)
            return embedding

    autoencoder = LitAutoencoder()
    autoencoder = autoencoder(torch.rand(1, 28 * 28))
    
    
.. code-block:: python

    # ----------------------------------
    # or using the AE to generate images
    # ----------------------------------
    class LitAutoEncoder(pl.LightningModule):
        def forward(self):
            z = torch.rand(1, 3)
            image = self.decoder(z)
            image = image.view(1, 1, 28, 28)
            return image

    autoencoder = LitAutoencoder()
    image_sample = autoencoder(()

Option 3: Production
====================
For production systems onnx or torchscript are much faster. Make sure you have added
a forward method or trace only the sub-models you need.

.. code-block:: python

    # ----------------------------------
    # torchscript
    # ----------------------------------
    autoencoder = LitAutoEncoder()
    torch.jit.save(autoencoder.to_torchscript(), "model.pt")
    os.path.isfile("model.pt")

.. code-block:: python

    # ----------------------------------
    # onnx
    # ----------------------------------
    with tempfile.NamedTemporaryFile(suffix='.onnx', delete=False) as tmpfile:
         autoencoder = LitAutoEncoder()
         input_sample = torch.randn((1, 28 * 28))
         autoencoder.to_onnx(tmpfile.name, input_sample, export_params=True)
         os.path.isfile(tmpfile.name)


********************
Using CPUs/GPUs/TPUs
********************
It's trivial to use CPUs, GPUs or TPUs in Lightning. There's NO NEED to change your code, simply change the :class:`~pytorch_lightning.trainer.Trainer` options.

.. code-block:: python

    # train on CPU
    trainer = pl.Trainer()

.. code-block:: python

    # train on 8 CPUs
    trainer = pl.Trainer(num_processes=8)

.. code-block:: python

    # train on 1024 CPUs across 128 machines
    trainer = pl.Trainer(
        num_processes=8,
        num_nodes=128
    )

.. code-block:: python

    # train on 1 GPU
    trainer = pl.Trainer(gpus=1)
    
.. code-block:: python

    # train on multiple GPUs across nodes (32 gpus here)
    trainer = pl.Trainer(
        gpus=4,
        num_nodes=8
    )
    
.. code-block:: python

    # train on gpu 1, 3, 5 (3 gpus total)
    trainer = pl.Trainer(gpus=[1, 3, 5])

.. code-block:: python

    # Multi GPU with mixed precision
    trainer = pl.Trainer(gpus=2, precision=16)

.. code-block:: python

    # Train on TPUs
    trainer = pl.Trainer(tpu_cores=8)

Without changing a SINGLE line of your code, you can now do the following with the above code:

.. code-block:: python

    # train on TPUs using 16 bit precision with early stopping
    # using only half the training data and checking validation every quarter of a training epoch
    trainer = pl.Trainer(
        tpu_cores=8,
        precision=16,
        early_stop_callback=True,
        limit_train_batches=0.5,
        val_check_interval=0.25
    )
    

***********
Checkpoints
***********
Lightning automatically saves your model. Once you've trained, you can load the checkpoints as follows:

.. code-block:: python

    model = LitModel.load_from_checkpoint(path)

The above checkpoint contains all the arguments needed to init the model and set the state dict.
If you prefer to do it manually, here's the equivalent

.. code-block:: python

    # load the ckpt
    ckpt = torch.load('path/to/checkpoint.ckpt')

    # equivalent to the above
    model = LitModel()
    model.load_state_dict(ckpt['state_dict'])

*****************
Optional features
*****************

Logging
=======
If you want to log to Tensorboard or your favorite logger, and/or the progress bar, use the
:func:`~~pytorch_lightning.core.lightning.LightningModule.log` method. You can call :func:`~~pytorch_lightning.core.lightning.LightningModule.log` from any part of your code, and
have full control on how the logs are aggregated and when.

To enable logging in the training loop:

.. code-block::

    class LitModel(pl.LightningModule):

        def training_step(self, batch, batch_idx):
            ...
            loss = F.mse_loss(x_hat, x)

            # .log sends to tensorboard/logger, prog_bar also sends to the progress bar
            self.log('my_train_loss', loss, prog_bar=True)
            return loss

Lightning can aggregate your logs for each epoch by specifying `on_epoch=True`.

.. code-block::

    class LitModel(pl.LightningModule):

        def training_step(self, batch, batch_idx):
            ...
            loss = F.mse_loss(x_hat, x)

            # Lightning will compute the mean of `my_train_loss` at the end of each epoch
            self.log('my_train_loss', loss, on_epoch=True)
            return loss


Anything you log in the validation loop will by default be logged at the end of each epoch:


.. code-block::

    class LitModel(pl.LightningModule):

        def validation_step(self, batch, batch_idx):
            ...
            loss = F.mse_loss(x_hat, x)

            # Lightning will compute the mean of `my_train_loss` across epoch
            self.log('my_val_loss', loss)

You can always override Lightning deafults to customize any behaviour. If you would like to aggregate manually, you can pass data from
your :func:`~~pytorch_lightning.core.lightning.LightningModule.validation_step` to :func:`~~pytorch_lightning.core.lightning.LightningModule.validation_step_end` or :func:`~~pytorch_lightning.core.lightning.LightningModule.validation_epoch_end` by returning a tensor or a dictionary, and you can manually decide what to log in :func:`~~pytorch_lightning.core.lightning.LightningModule.validation_epoch_end`:


.. code-block::

    class LitModel(pl.LightningModule):

        def validation_step(self, batch, batch_idx):
            ...

	        loss = F.mse_loss(x_hat, x)
	        self.log('val_loss', loss, on_step=False, on_epoch=False)
            # anything you return will be available in validation_step_end and validation_epoch_end
	        return {'a': gpu_idx}

	    def validation_step_end(self, validation_step_output):
	        # {'a': [0, 1, 2, 3]}
	        gpu_0_se = validation_step_output[0]
	        gpu_1_se = validation_step_output[1]
	        gpu_2_se = validation_step_output[2]
	        gpu_3_se = validation_step_output[3]
            # anything you return will be available in validation_epoch_end
	        return gpu_0_se + gpu_1_se + gpu_2_se + gpu_3_se

	    def validation_epoch_end(self, validation_step_outputs):
	        # you can compute your own reduction of metrics or compute anything on values from your validation liip

Read more about :ref:`loggers`.


Callbacks
=========
A callback is an arbitrary self-contained program that can be executed at arbitrary parts of the training loop.

Here's an example adding a not-so-fancy learning rate decay rule:

.. code-block:: python

    class DecayLearningRate(pl.Callback)

        def __init__(self):
            self.old_lrs = []

        def on_train_start(self, trainer, pl_module):
            # track the initial learning rates
            for opt_idx in optimizer in enumerate(trainer.optimizers):
                group = []
                for param_group in optimizer.param_groups:
                    group.append(param_group['lr'])
                self.old_lrs.append(group)

        def on_train_epoch_end(self, trainer, pl_module):
            for opt_idx in optimizer in enumerate(trainer.optimizers):
                old_lr_group = self.old_lrs[opt_idx]
                new_lr_group = []
                for p_idx, param_group in enumerate(optimizer.param_groups):
                    old_lr = old_lr_group[p_idx]
                    new_lr = old_lr * 0.98
                    new_lr_group.append(new_lr)
                    param_group['lr'] = new_lr
                 self.old_lrs[opt_idx] = new_lr_group
                 

Things you can do with a callback:

- Send emails at some point in training
- Grow the model
- Update learning rates
- Visualize gradients
- ...
- You are only limited by your imagination

:ref:`Learn more about custom callbacks <callbacks>`.


Datamodules
===========
DataLoaders and data processing code tends to end up scattered around.
Make your data code reusable by organizing it into a :class:`~pytorch_lightning.core.datamodule.LightningDataModule`.

.. code-block:: python

  class MNISTDataModule(pl.LightningDataModule):

        def __init__(self, batch_size=32):
            super().__init__()
            self.batch_size = batch_size

        # When doing distributed training, Datamodules have two optional arguments for
        # granular control over download/prepare/splitting data:

        # OPTIONAL, called only on 1 GPU/machine
        def prepare_data(self):
            MNIST(os.getcwd(), train=True, download=True)
            MNIST(os.getcwd(), train=False, download=True)

        # OPTIONAL, called for every GPU/machine (assigning state is OK)
        def setup(self, stage):
            # transforms
            transform=transforms.Compose([
                transforms.ToTensor(),
                transforms.Normalize((0.1307,), (0.3081,))
            ])
            # split dataset
            if stage == 'fit':
                mnist_train = MNIST(os.getcwd(), train=True, transform=transform)
                self.mnist_train, self.mnist_val = random_split(mnist_train, [55000, 5000])
            if stage == 'test':
                self.mnist_test = MNIST(os.getcwd(), train=False, transform=transform)

        # return the dataloader for each split
        def train_dataloader(self):
            mnist_train = DataLoader(self.mnist_train, batch_size=self.batch_size)
            return mnist_train

        def val_dataloader(self):
            mnist_val = DataLoader(self.mnist_val, batch_size=self.batch_size)
            return mnist_val

        def test_dataloader(self):
            mnist_test = DataLoader(self.mnist_test, batch_size=self.batch_size)
            return mnist_test

:class:`~pytorch_lightning.core.datamodule.LightningDataModule` is designed to enable sharing and reusing data splits
and transforms across different projects. It encapsulates all the steps needed to process data: downloading,
tokenizing, processing etc.

Now you can simply pass your :class:`~pytorch_lightning.core.datamodule.LightningDataModule` to
the :class:`~pytorch_lightning.trainer.Trainer`:

.. code-block::

    # init model
    model = LitModel()

    # init data
    dm = MNISTDataModule()

    # train
    trainer = pl.Trainer()
    trainer.fit(model, dm)

    # test
    trainer.test(datamodule=dm)

DataModules are specifically useful for building models based on data. Read more on :ref:`datamodules`.

------

*********
Debugging
*********
Lightning has many tools for debugging. Here is an example of just a few of them:

.. code-block:: python

    # use only 10 train batches and 3 val batches
    trainer = pl.Trainer(limit_train_batches=10, limit_val_batches=3)

.. code-block:: python

    # Automatically overfit the sane batch of your model for a sanity test 
    trainer = pl.Trainer(overfit_batches=1)

.. code-block:: python

    # unit test all the code- hits every line of your code once to see if you have bugs,
    # instead of waiting hours to crash on validation
    trainer = pl.Trainer(fast_dev_run=True)

.. code-block:: python
   
   # train only 20% of an epoch
   trainer = pl. Trainer(limit_train_batches=0.2)

.. code-block:: python

    # run validation every 20% of a training epoch
    trainer = pl.Trainer(val_check_interval=0.2)

.. code-block:: python
    
    # Profile your code to find speed/memory bottlenecks
    Trainer(profiler=True)
 
---------------

***************************
Advanced Lightning Features
***************************

Once you define and train your first Lightning model, you might want to try other cool features like

- :ref:`Automatic early stopping <early_stopping>`
- :ref:`Automatic truncated-back-propagation-through-time <trainer:truncated_bptt_steps>`
- :ref:`Automatically scale your batch size <training_tricks:Auto scaling of batch size>`
- :ref:`Automatically find a good learning rate <lr_finder>`
- :ref:`Load checkpoints directly from S3 <weights_loading:Checkpoint Loading>`
- :ref:`Scale to massive compute clusters <slurm>`
- :ref:`Use multiple dataloaders per train/val/test loop <multiple_loaders>`
- :ref:`Use multiple optimizers to do reinforcement learning or even GANs <optimizers:Use multiple optimizers (like GANs)>`

Or read our :ref:`introduction_guide` to learn more!

-------------

**********
Learn more
**********

That's it! Once you build your module, data, and call trainer.fit(), Lightning trainer calls each loop at the correct time as needed.

You can then boot up your logger or tensorboard instance to view training logs

.. code-block:: bash

    tensorboard --logdir ./lightning_logs

Masterclass
===========

Go pro by tunning in to our Masterclass! New episodes every week.

.. image:: _images/general/PTL101_youtube_thumbnail.jpg
    :width: 500
    :align: center
    :alt: Masterclass
    :target: https://www.youtube.com/playlist?list=PLaMu-SDt_RB5NUm67hU2pdE75j6KaIOv2
