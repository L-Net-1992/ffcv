Parameter Tuning
================

FFCV is a generic library and to be able to provide great performance in many situations, some parameters might be required to tune the system to your particular application.

In order to help users with that task, we consider a couple common use case and provide appropriate recommended setting.

Scenario: Small dataset
-----------------------

If the dataset you are working on is small or if you are lucky enough to have a machine with large amounts of RAM. We recommend the following:

- Use ``os_cache=True`` in ``Loader()``. The first epoch will be a little bit slower as the operating system is not able to pre-fetch the data as well but once the dataset has been completely cached in RAM then it will read directly from there with no overhead.

- You can use ``RANDOM`` or ``QUASI_RANDOM``. They should both perform very similarly (QUASI_RANDOM might be marginally better).


Scenario: Large scale datasets
------------------------------

If your dataset is too large to be cached on the machine we recommend:

- Use ``os_cache=False``. Since the data can't be cached, FFCV will have to read it over and over. Having FFCV take over the operating system for caching is beneficial as it knows in advance the which samples will be needed in the future and can load them ahead of time.
- If you need randomness and perfect uniform sampling isn't mission critical we recommend using ``QUASI_RANDOM`` traversal order. It will optimize the order to minimize the reads on the underlying storage while maintaining very good randomness properties. If you ever experienced with the ``shuffle()`` function of ``webdataset`` and the quality of the randomness wasn't sufficient we still suggest you give ``QUASI_RANDOM`` a try as it should be significantly better.


Scenario: Multi-GPU training (1 model, multiple GPUs)
-----------------------------------------------------

FFCV's ``Loader`` class offer a flag ``distributed`` that will make the loader behave similarly to the Pytorch's ``DistributedSampler`` applied on a ``DataLoader``. If that's what your code is using, switching to FFCV should just be a matter of replacing the data loader.

FFCV should work fine with Pytorch's ``DataParallel`` wrapper but we agree with the developpers and recommend you use ``DistributedDataParallel`` with FFCV's ``distributed`` flag enabled.

The same recommendations mentioned above related to dataset size still apply here but we empaphize on the fact that ``os_cache=True`` is particularly beneficial in this scenario. Indeed, as multiple processes will access the same dataset, having the caching at the OS level allow for data sharing between them, reducing overall memory consumption.

.. note ::
    QUASI_RANDOM isn't currently supported with ``distributed=True``. While this is technically possible to implement the team haven't invested the necessray time yet. We also welcome pull requests.

We encourage users to try different values for the ``num_workers`` parameters. As FFCV is usually very CPU resource efficient it is sometime beneficial to use fewer workers to avoid scheduling and cache inefficiencies.

Scenario: Grid search (1 model per GPU)
---------------------------------------

This use case is similar to the previous. One should still have one process per GPU and if all the models are getting trained using the same dataset, ``os_cache=True`` should be preferred to allow cache sharing between the jobs. Note that if the dataset is bigger than the amount of main memory, ``os_cache=False`` might still perform better and we encourage users to try both.

Scenario: Extreme grid search (2+ models per GPU)
--------------------------------------------------

Unlike other solutions, FFCV is thread based and not processed based. As a result, users are able to train multiple models on a single GPU. This is particularly useful for small models that can't leverage the compute power of powerful GPUs. To do so users have to do the following:

- Run a **single** process per GPU
- The main thread of that process should start one thread for each model that has to be trained concurrently
- Each thread creates his own FFCV `Loader` and model and trains normally
- As for regular Grid search, ``os_cache=True`` is mostly the best choice here but it doesn't hurt to try disabling it for very large scale datasets

.. warning ::
    It is a common mistake to assume that running multiple processes on the same GPU will improve speed. For security reasons and unless Nvidia MPS service is enabled, a GPU can only be used by a single process at a time. If more try, GPU time will be shared between them but they will never run concurrently.

.. note ::
   We have experienced some CUDNN bugs while running multiple odels on the same GPU. It seems to originate from scheduling concurrently multiple BatchNorm layers. If you encounter that issue a simple fix is to put a lock around the forward pass of your models. This will make sure that no two forward pass is scheduled concurrently. This shouldn't impact performance too much as CUDA calls are asynchronous anyway.