# Cirrus-CI

Similar to other integrated github CI/CD services, Cirrus utilizes a simple
YAML-based configuration/description file: ``.cirrus.yml``.  Ref: https://cirrus-ci.org/


## Workflow

All tasks execute in parallel, unless there are conditions or dependencies
which alter this behavior.  Within each task, each script executes in sequence,
so long as any previous script exited successfully.  The overall state of each
task (pass or fail) is set based on the exit status of the last script to execute.


### ``integration`` Task

1. After `gating` passes, spin up one VM per
   `matrix: image_name` item. Once accessible, ``ssh``
   into each VM as the `root` user.

2. ``setup_environment.sh``: Configure root's `.bash_profile`
    for all subsequent scripts (each run in a new shell).  Any
    distribution-specific environment variables are also defined
    here.  For example, setting tags/flags to use compiling.

5. ``integration_test.sh``: Execute integration-testing.  This is
   much more involved, and relies on access to external
   resources like container images and code from other repositories.
   Total execution time is capped at 2-hours (includes all the above)
   but this script normally completes in less than an hour.


### ``cache_images`` Task

1. When a PR is merged (``$CIRRUS_BRANCH`` == ``main``), run another
   round of the ``integration`` task (above).

2. After confirming the tests all pass post-merge, cirrus will
   spin up a special VM capable of communicating with the GCE API.
   Once accessible, it will ``ssh`` into the special VM and run
   the following scripts.

3. ``setup_environment.sh``: Same as the ``integration``
   Task (above).

4. ``build_vm_images.sh``: Examine the merged PR's description on github.
   If it contains the magic string ``***CIRRUS: REBUILD IMAGES***``, then
   continue, otherwise display a message, take no further action, and
   exit successfully.  This prevents production of new VM images unless
   they are called for, thereby saving the cost of needlessly storing them.

5. If the magic string was found, utilize [the packer tool](http://packer.io/docs/)
   to produce new VM images.  Create new VMs starting from *base-images* (see below),
   connect to them with ``ssh``, and perform the steps defined by the
   ``conmon_images.yml`` file.

    1. Copy the current state of the repository into ``/tmp/conmon``.
    2. Execute distribution-specific scripts to prepare the image for
       use by the ``integration`` task (above).  For example,
       ``fedora_setup.sh``.
    3. If successful, shut down each VM and create a new GCE Image
       named with the base image, and the commit sha of the merge.

***Note:*** The ``.cirrus.yml`` file must be manually updated with the new
images names, then the change sent in via a secondary pull-request.  This
ensures that all the ``integration`` tasks can pass with the new images,
before subjecting all future PRs to them.  A workflow to automate this
process is described in comments at the end of the ``.cirrus.yml`` file.


### Base-images

Base-images are VM disk-images specially prepared for executing as GCE VMs.
In particular, they run services on startup similar in purpose/function
as the standard 'cloud-init' services.

*  The google services are required for full support of ssh-key management
   and GCE OAuth capabilities.  Google provides native images in GCE
   with services pre-installed, for many platforms. For example,
   RHEL, CentOS, and Ubuntu.

*  Google does ***not*** provide any images for Fedora or Fedora Atomic
   Host (as of 11/2018), nor do they provide a base-image prepared to
   run packer for creating other images in the ``build_vm_images`` Task
   (above).

*  Base images do not need to be produced often, but doing so completely
   manually would be time-consuming and error-prone.  Therefore a special
   semi-automatic *Makefile* target is provided to assist with producing
   all the base-images: ``conmon_base_images``

To produce new base-images, including an `image-builder-image` (used by
the ``cache_images`` Task) some input parameters are required:

    *  ``GCP_PROJECT_ID``: The complete GCP project ID string e.g. foobar-12345
       identifying where the images will be stored.

    *  ``GOOGLE_APPLICATION_CREDENTIALS``: A *JSON* file containing
       credentials for a GCE service account.  This can be [a service
       account](https://cloud.google.com/docs/authentication/production#obtaining_and_providing_service_account_credentials_manually)
       or [end-user
       credentials](https://cloud.google.com/docs/authentication/end-user#creating_your_client_credentials]

    *  CSV's of builders to use must be specified to ``PACKER_BUILDS``
       to limit the base-images produced.  For example,
       ``PACKER_BUILDS=fedora,image-builder-image``.

The following process should be performed on a bare-metal CentOS 7 machine
with network access to GCE.  Software dependencies can be obtained from
the ``packer/image-builder-image_base_setup.sh`` script.

Alternatively, an existing image-builder-image may be used from within GCE.
However it must be created with elevated cloud privileges.  For example,

```
$ alias pgcloud='sudo podman run -it --rm -e AS_ID=$UID
    -e AS_USER=$USER -v /home/$USER:/home/$USER:z cevich/gcloud_centos:latest'

$ URL=https://www.googleapis.com/auth
$ SCOPES=$URL/userinfo.email,$URL/compute,$URL/devstorage.full_control

$ pgcloud compute instances create $USER-making-images \
    --image-family image-builder-image \
    --boot-disk-size "200GB" \
    --min-cpu-platform "Intel Haswell" \
    --machine-type n1-standard-2 \
    --scopes $SCOPES

$ pgcloud compute ssh centos@$USER-making-images
...
```

When ready, change to the ``packer`` sub-directory, and run:

```
$ make conmon_base_images GCP_PROJECT_ID=<VALUE> \
    GOOGLE_APPLICATION_CREDENTIALS=<VALUE> \
    PACKER_BUILDS=<OPTIONAL>
```

Assuming this is successful (hence the semi-automatic part), packer will
produce a ``packer-manifest.json`` output file.  This contains the base-image
names suitable for updating in ``.cirrus.yml``, `env` keys ``*_BASE_IMAGE``.

On failure, it should be possible to determine the problem from the packer
output.  The only exception is for the Fedora and FAH builds, which utilize
local qemu-kvm virtualisation.  To observe the serial-port output from those
builds, set the ``TTYDEV`` parameter to your current device.  For example:

```
$ make conmon_base_images ... TTYDEV=$(tty)
  ...
```
