..
  Technote content.

  See https://developer.lsst.io/docs/rst_styleguide.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   Notes on running the Stack using Singularity

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).


- :ref:`singularity-intro`
- :ref:`singularity-image`
- :ref:`singularity-proc`
- :ref:`singularity-sfd`
- :ref:`singularity-htcondor-root`

.. _singularity-intro:

Introduction to Singularity
===========================

The `Singularity container project <http://www.sylabs.io/>`_ has commanded attention in the
scientific world and the high performance computing (HPC) community.
Singularity gives users the freedom to execute their customized applications, bundled with required dependencies,
in an isolated context on HPC platforms without impacting the underlying system.
Unprivileged users are able to effectively swap out the operating system on the HPC host for one that they select and control.

Singularity is well suited for HPC platforms because it prevents user context escalation within the container.
The user inside a Singularity container is the same user as outside the container,
and this feature has made it possible to run user supplied containers on shared infrastructures.

.. _singularity-image:

Singularity Image Files
=======================

Singularity utilizes a single file as a container image, providing the
complete representation of all the software/files within the container.
Because Singularity images are single files, they are easy to distribute, copy and manage.
Administering permissions for the container is reduced to standard file system permissions.
The Singularity container images are compressed and consume relatively little disk space.

A local Singularity image file can be created from an LSST
`Science Pipelines <https://pipelines.lsst.io>`_
`docker <https://pipelines.lsst.io/install/docker.html>`_ image on
`dockerhub <https://hub.docker.com/r/lsstsqre/centos/>`_ utilizing ``singularity pull``.
An example of image generation is::

     singularity pull --name  "/home/daues/7-stack-lsst_distrib-v16.0.img" docker://lsstsqre/centos:7-stack-lsst_distrib-v16_0


.. _singularity-proc:

Running processCcd.py with Singularity
======================================

Processing with an LSST weekly or daily tag such as ``lsstsqre/centos:7-stack-lsst_distrib-w_2018_28`` on the `Verification Cluster <https://developer.lsst.io/services/verification.html>`_
can be initiated promptly using Singularity.   A local container image can be created in a few minutes via::

     singularity pull --name  "/home/daues/7-stack-lsst_distrib-w_2018_28.img" docker://lsstsqre/centos:7-stack-lsst_distrib-w_2018_28

This approach is rather expedient in comparison to building a stack from source on a target platform
for each weekly/daily tag or release.
With the container image present, a Slurm job can be submitted to the Verification Cluster that runs a ``singularity exec``
to launch processing.  A simple example running processCcd is:

.. code-block:: shell

     #!/bin/bash -l
     #SBATCH -p debug
     #SBATCH -N 1
     #SBATCH -n 1
     #SBATCH -t 00:30:00
     #SBATCH -J job1

     singularity exec -B /datasets/:/datasets/ -B /project/:/project/ -B /scratch/:/scratch/ -B /home/:/home/  /home/daues/7-stack-lsst_distrib-w_2018_23.img /home/daues/run_ProcessCcd.sh

In this example a wrapper script ``run_ProcessCcd.sh`` is executed within the container;
it initializes the stack under ``/opt/lsst``
within the container and runs the desired command line:

.. code-block:: shell

     #!/bin/bash
     source /opt/lsst/software/stack/loadLSST.bash
     setup lsst_distrib
     processCcd.py /datasets/hsc/repo --calib /datasets/hsc/repo/CALIB/ --output /scratch/daues/output1  --id visit=6320 ccd=10


The ``singularity exec`` command line utilizes user-bind path specifiers such as ``-B /datasets/:/datasets/`` to map file spaces
into the Singulariy container. The format of the specifications is ``-B src:dest``,  where ``src`` and ``dest`` are outside
and inside paths, respectively.

.. _singularity-sfd:

SingleFrameDriver in SMP mode with Singularity
==============================================

In this section we show one approach to use Singularity for processing at a more significant scale,
sufficient to fully utilize a single multi-core node.
This example runs ``singleFrameDriver``, a ``ctrl_pool``-style pipeline driver
(`DMTN-023 <https://dmtn-023.lsst.io>`_), in SMP mode within the context of
a ``singularity exec``.  We write a Slurm job description to launch the ``singularity exec``
on a compute node of the Verification Cluster:

.. code-block:: shell


     #!/bin/bash -l
     #SBATCH -p debug
     #SBATCH -N 1
     #SBATCH -n 1
     #SBATCH -t 00:30:00
     #SBATCH -J job2

     singularity exec -B /datasets/:/datasets/ -B /project/:/project/ -B /scratch/:/scratch/ -B /home/:/home/  /home/daues/singularity/images/7-stack-lsst_distrib-w_2018_23.img /home/daues/run_singleFrameDriver_192.sh

A wrapper script :file:`run_singleFrameDriver_192.sh` is executed within the container, initializing the stack and running the
``singleFrameDriver`` command line:

.. code-block:: text

     #!/bin/bash
     source /opt/lsst/software/stack/loadLSST.bash
     setup lsst_distrib
     singleFrameDriver.py /datasets/hsc/repo --calib /datasets/hsc/repo/CALIB/ --output /scratch/daues/output2 --batch-type smp --job test --id visit=6320 ccd=10..33 --cores 48  --mpiexec "-launcher ssh"


We note that an option ``--mpiexec "-launcher ssh"`` is included on the ``singleFrameDriver`` command line.
Without this option, we observe the internal ``mpiexec`` of the ``singleFrameDriver --batch-type smp`` to attempt to use
the slurm launcher, but this fails because the slurm commands such as ``srun`` are not present within the container.
We work around this by designating an alternative launcher for the ``mpiexec``.

.. _singularity-htcondor-root:

Configuring HTCondor to run Singularity Jobs
============================================

In this section we consider the context of an HTCondor pool with admin/root level installations.
An HTCondor worker node (i.e., running a ``startd`` daemon) can be configured to run jobs within
individual slots that are comprised
of a Singularity container.  Sample additions to the HTCondor configuration file (typically located at
``/etc/condor/condor_config.local``) are:

.. code-block:: text

    SINGULARITY = /usr/local/bin/singularity
    SINGULARITY_JOB = !isUndefined(TARGET.SingularityImage)
    SINGULARITY_IMAGE_EXPR = TARGET.SingularityImage

    # Maps $_CONDOR_SCRATCH_DIR on the host to /srv inside the image.
    SINGULARITY_TARGET_DIR = /srv
    SINGULARITY_BIND_EXPR = "/home,/scratch,/data"

When the HTCondor ``startd`` runs with these settings a machine ad ``HasSingularity``
will be displayed on submit nodes:


.. code-block:: text

    % condor_status -long slot1@worker01.ncsa.illinois.edu
      ...
      HasSingularity = true
      ...

This ClassAd will allow jobs that require Singularity to match against the resource.
In addition, this configuration allows the submission of Singularity jobs, but does not force all jobs
to be so.  If an HTCondor job description has an expression such as::

    +SingularityImage = "7-stack-lsst_distrib-w_2018_28.img"

then the job will run as a Singularity container and use the provided image.
If no such expression exists, a conventional job can be submitted for execution.

An alternative configuration with greater administrative control could specify:

.. code-block:: text

    # Forces _all_ jobs to run inside singularity.
    SINGULARITY_JOB = true

    # Forces all jobs to use this image.
    SINGULARITY_IMAGE_EXPR = "/cvmfs/cernvm-prod.cern.ch/cvm3"

In this case all jobs run as Singularity containers and utilize the designated image (i.e., the user does not
have the ability to specify the image.

Returning to the original flexible configuration, a example HTCondor job description could specify:

.. code-block:: text

    universe = vanilla

    output = out.$(Cluster).$(Process)
    error = err.$(Cluster).$(Process)

    executable = setupLSSTStack.sh

    log = test.log

    should_transfer_files = YES
    when_to_transfer_output = ON_EXIT
    notification=Error
    transfer_executable = True

    request_cpus = 1

    Requirements = HAS_SINGULARITY == TRUE

    +SingularityImage = "7-stack-lsst_distrib-w_2018_28.img"

    queue

In this example a Singularity image ``7-stack-lsst_distrib-w_2018_28.img``
generated from an LSST docker release
image (i.e., a ``singularity pull`` thereof) is used.
In order to guarantee that this job runs on a worker node that supports Singularity,
we include the Requirements clause for HAS_SINGULARITY.
Serving as the executable of this job, an example wrapper script
:file:`setupLSSTStack.sh` is used to run commands inside the container:

.. code-block:: shell

   #!/bin/bash
   source  /opt/lsst/software/stack/loadLSST.bash
   setup lsst_distrib
   eups list | grep lsst_distrib






.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
