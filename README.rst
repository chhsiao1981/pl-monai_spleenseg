Spleen 3D image segmentation (MONAI)
====================================

|Version| |MIT License| |ci|

``pl-monai_spleenseg`` is a `ChRIS <https://chrisproject.org/>`__ *DS*
plugin based off Project MONAI’s spleen segmentation exemplar. This
plugin implements the training and inference phases as two distinct
modes of operation. For training, input files are a set of training
examples (images and segmented images) and output files are training
plots and weight (model) files in ``pth`` or ``ONNX`` format. For
inference, input files are a model file and an input image (or set of
images) and the output is a segmented image (or set of images).

Abstract
--------

Based off Project MONAI’s `spleen segmentation
notebook <https://github.com/Project-MONAI/tutorials/blob/main/3d_segmentation/spleen_segmentation_3d.ipynb>`__,
this plugin implements both the *training* and *inference* phases of the
notebook, using data supplied in the *parent* plugin (see
Implementation).

For the most part, the python notebook code could have been mostly used
*verbatim* in the plugin; however, in this example considerable and
deeper refactoring was performed. Other than of course now being a
complete stand alone implementation, this plugin allows for *continuous*
(or *continued*) training, as well as saving examples of all
training/validation/inference datasets as NIfTI volumes to allow for
better understanding of the process.

For the *training* phase, the parent plugin provides input images
(training and labeled) and the output is a model (``pth`` or ``ONNX``
format). For the *inference* phase, the input is a model file, an image
(or more) to analyze, and the output is a segmented volume file.

An additional remote inference phase is available. If the mode is
specified as ``<modelID>@<URI>`` where ``modelID`` is the name of a
model and ``URI`` denotes a ``pfms`` server’s base URL, the plugin will
transmit input images to the remote endpoint where the ``pfms`` service
will run the inference and return a segmented volume file.

Implementation
--------------

The original notebook is a largely self-contained *monolithic*
application. Exemplar input data is pulled from the web, and the
notebook proceeds from there. In the case of this ChRIS plugin, some
straightforward organizational changes are necessary. The training data
is assumed to already have been downloaded *a priori* and is provided to
this plugin by its *parent*. Outputs of the training are model weight
filesTh.

Installation
------------

``pl-monai_spleenseg`` is a `ChRIS <https://chrisproject.org/>`__
*plugin*, meaning it can run from either within *ChRIS* or the
command-line.

On the metal
~~~~~~~~~~~~

Despite being a ChRIS plugin, *on the metal* development/usage is
supported. Simply check out the repository and do a ``pip install`` to
either a local ``venv`` or an existing one:

.. code:: bash

   pip install -U ./
   spleenseg --help

Apptainer
~~~~~~~~~

To get started with local command-line usage, use
`Apptainer <https://apptainer.org/>`__ (a.k.a. Singularity) to run
``pl-monai_spleenseg`` as a container:

.. code:: shell

   apptainer exec docker://fnndsc/pl-monai_spleenseg spleenseg_train \
               [--args values...] input/ output/

To print its available options, run:

.. code:: shell

   apptainer exec docker://fnndsc/pl-monai_spleenseg spleenseg_train --help

Usage
-----

.. code:: html

       ARGS
           [--mode <inference|training>]
           The mode of operation. If the actual text "training" text has any additional characters,
           then the epoch training is skipped and only the post-training logic is executed --
           this allows the system to operate as if in training mode, but only perform the post
           training steps. For example, "--mode trainingPost" would invoke this mode.

           If the mode is specified as <model>@<remote> (e.g. splsegv1@http://pfms.org/api/v1/)
           then the pfms server at (in this example) http://pfms.org with base route /api/v1/
           is used to perform remote inference. Here, the model called "splsegv1" on the remote
           pfms server will be used to run inference. The return from this call is a NIfTI
           segmented volume saved in the <outputDir>.

           [--man]
           If specified, print this help page and quit.

           [--version]
           If specified, print the version and quit.

           [--logTransformVols]
           If specified, log a set of intermediary NIfTI volumes as used for training,
           validation, spacing, and inference.

           [--useModel <modelFile>]
           If specified, use <modelFile> for inference or continued training.

           [--trainImageDir <train> --trainLabelsDir <label>]
           In the <inputDir>, the name of the directory containing files for training
           with their corresponding label targets.

           [--testImageDir]
           In the <inputDir> the name of the directory containing images for inference.

           [--device <device>]
           The device to use, typically "cpu" or "cuda:0".

           [--determinismSeed <seed>]
           Set the training seed.

           [--maxEpochs <count>]
           The max number of training epochs.

           [--validateSize <size>]
           In the training space, the number of images that should be used for validation
           and not training.

Examples
--------

``spleenseg_train`` requires two positional arguments: a directory
containing input data, and a directory containing output data (graphs
and “model” files). In this plugin, data is downloaded from
`medicaldecathelon <http://medicaldecathelon.com>`__. To get this data,
first set an environment variable pointing at the directory to contain
the pulled and unpacked data:

.. code:: bash

   export MONAI_DATA_DIR=/some/dir

now, you can pull the data with this python snippet:

.. code:: python

   # You probably will need to
   #   pip install -q "monai-weekly[gdown, nibabel, tqdm, ignite]"
   from monai.apps import download_and_extract

   directory = os.environ.get("MONAI_DATA_DIRECTORY")
   root_dir = tempfile.mkdtemp() if directory is None else directory
   print(root_dir)


   resource = "https://msd-for-monai.s3-us-west-2.amazonaws.com/Task09_Spleen.tar"
   md5 = "410d4a301da4e5b2f6f86ec3ddba524e"
   compressed_file = os.path.join(root_dir, "Task09_Spleen.tar")
   data_dir = os.path.join(root_dir, "Task09_Spleen")
   if not os.path.exists(data_dir):
       download_and_extract(resource, compressed_file, root_dir, md5)

Or simply run the supplied ``trainingDataPull.py`` script (which is
essentially the above code):

.. code:: bash

   python trainingDataPull.py

Create some ``output`` directory, and using our ``$MONAI_DATA_DIR``, we
can run the plugin:

.. code:: shell

   mkdir outgoing/
   apptainer exec docker://fnndsc/pl-monai_spleenseg:latest spleenseg \
           [--args] $MONAI_DATA_DIR outgoing/

Development
-----------

Instructions for developers.

Building
~~~~~~~~

Build a local container image:

.. code:: shell

   docker build -t localhost/fnndsc/pl-monai_spleenseg .

Running
~~~~~~~

Mount the source code ``spleenseg_train.py`` into a container to try out
changes without rebuild.

.. code:: shell

   docker run --rm -it --userns=host -u $(id -u):$(id -g) \
       -v $PWD/spleenseg/spleenseg.py:/usr/local/lib/python3.12/site-packages/spleenseg/spleenseg.py:ro \
       -v $PWD/in:/incoming:ro -v $PWD/out:/outgoing:rw -w /outgoing \
       localhost/fnndsc/pl-monai_spleenseg spleenseg /incoming /outgoing

Testing
~~~~~~~

Run unit tests using ``pytest``. It’s recommended to rebuild the image
to ensure that sources are up-to-date. Use the option
``--build-arg extras_require=dev`` to install extra dependencies for
testing.

.. code:: shell

   docker build -t localhost/fnndsc/pl-monai_spleenseg:dev --build-arg extras_require=dev .
   docker run --rm -it localhost/fnndsc/pl-monai_spleenseg:dev pytest

Release
-------

Steps for release can be automated by `Github
Actions <.github/workflows/ci.yml>`__. This section is about how to do
those steps manually.

Increase Version Number
~~~~~~~~~~~~~~~~~~~~~~~

Increase the version number in ``setup.py`` and commit this file.

Push Container Image
~~~~~~~~~~~~~~~~~~~~

Build and push an image tagged by the version. For example, for version
``1.2.3``:

::

   docker build -t docker.io/fnndsc/pl-monai_spleenseg:1.2.3 .
   docker push docker.io/fnndsc/pl-monai_spleenseg:1.2.3

Get JSON Representation
~~~~~~~~~~~~~~~~~~~~~~~

Run
```chris_plugin_info`` <https://github.com/FNNDSC/chris_plugin#usage>`__
to produce a JSON description of this plugin, which can be uploaded to
*ChRIS*.

.. code:: shell

   docker run --rm docker.io/fnndsc/pl-monai_spleenseg:1.2.3 chris_plugin_info \
               -d docker.io/fnndsc/pl-monai_spleenseg:1.2.3 > chris_plugin_info.json

Intructions on how to upload the plugin to *ChRIS* can be found here:
https://chrisproject.org/docs/tutorials/upload_plugin

*-30-*

.. |Version| image:: https://img.shields.io/docker/v/fnndsc/pl-monai_spleenseg?sort=semver
   :target: https://hub.docker.com/r/fnndsc/pl-monai_spleenseg
.. |MIT License| image:: https://img.shields.io/github/license/fnndsc/pl-monai_spleenseg
   :target: https://github.com/FNNDSC/pl-monai_spleenseg/blob/main/LICENSE
.. |ci| image:: https://github.com/FNNDSC/pl-monai_spleenseg/actions/workflows/ci.yml/badge.svg
   :target: https://github.com/FNNDSC/pl-monai_spleenseg/actions/workflows/ci.yml
