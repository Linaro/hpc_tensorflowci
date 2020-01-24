# HPC SIG TensorFlow CI Recipes

## Overview

These recipes are intended to be used in coordination with [hpc_lab_setup](https://github.com/Linaro/hpc_lab_setup) to setup a CI loop for TensorFlow on AArch64 CentOS7 and CentOS8.
To achieve this the recipes build :
- bazel (rpm, using a slightly altered version of vbatts' from Fedora spec)
- numpy (wheel)
- openblas (rpm, either using OpenHPC's spec for CentOS7 or upstream for CentOS8)
- tensorflow (wheel, 1.15.0 and 2.1.0 are supported)
- dnnl (0.19, also works with fujitsu's DNNL)

Each of those, in addition to the common parts are separated into roles, so they can be executed separately.

Note that DNNL is not yet integrated into TensorFlow since this requires patching the bazelrules to build it.

## How to Use

To use the recipes, refer to [the build_tensorflow Jenkins job of hpc_lab_setup](https://github.com/Linaro/hpc_lab_setup/blob/master/files/build_tensorflow.yml) :
- Run install_python3.yml first to make sure the python requirements are present on the target building machine
- Then run build_tensorflow.yml to build the stack

### Exempli gratia

```bash
$ cat << EOF > hosts
[target]
$IP_OF_THE_MACHINE_YOU_WANT_TO_BUILD_ON
EOF
$ ansible-playbook -i hosts install_python3.yml # Add -v(vv) for verbose output
$ ansible-playbook -i hosts build_tensorflow.yml # You can also override default variables, although default should build everything fine
```

## Overview of the build variables

### Common

- arch : architecture of the target machine (aarch64 by default)
- user : name of the passwordless user that will be created to build everything
- build_id: index of the build in order to have separate builds on the same machine (in separate virtual environments)
- use_openhpc: boolean to toggle the use of OpenHPC's stack (if CentOS7 then this is true by default, else it is false)

We then have multiple variables corresponding to the locations of the various libraries that are dependencies for the build (openblas, fftw, hdf5, openmpi and toolchain/GCC)

Then multiple arrays containing the dependencies to be fetched via OpenHPC or Upstream repositories.

### Bazel

The variables for each package built (that is bazel, dnnl, openblas, numpy, tensorflow) are quite similar :
- build_bazel : boolean that determines if bazel should be built or installed from repositories
- bazel_version : 0.26.1 for 1.15.0 and 0.29.1 for 2.1.0
- bazel_minorversion : used to determine the rpm name
- bazel_rpm : path to the built bazel rpm.
- bazel_dir\* : various directories to be used in the building of bazel
- bazel : dictionary containing release_url, zip name (release_zip), binary name (binary)

### DNNL

Variables are similar to Bazel's, but here are the differences:
- dnnl_prefix : where to install the library (since it is not an package), either in OpenHPC's tree if OpenHPC's toolchain is used; or in a standard path (/usr/local/mkl-dnn) if not.
- dnnl.arch_opt_flags : see [Targeting Specific Architecture section of the upstream documentation](http://intel.github.io/mkl-dnn/dev_guide_build_options.html)
- dnnl.build_type : DEBUG, RELEASE

### NumPy

Same as above, let's keep things DRY
- numpy_patching_required: see [this issue](https://github.com/ansible/ansible/issues/8603) and below
- numpy.patch : name of the "patch" (workaround really) to the below issue

##### NumPy and GCC

This issue : https://github.com/ansible/ansible/issues/8603 details problems with building NumPy with not-so-recent versions of GCC (so standard CentOS toolchains and OpenHPC's)
The workaround or "patch" will be applied if numpy_patching_required is set to true.
The workaround is here : https://github.com/Linaro/hpc_tensorflowci/blob/master/roles/numpy/templates/numpy_centos7_aarch64.patch.j2

##### NumPy and Environment

NumPy hooks up to the environment libraries (i.e. BLAS, LAPACK, FFTW) via the "numpy-site.cfg" file to be put in the HOME_DIR of the user building NumPy.
Note that this also applies to SciPy.

### OpenBLAS

- openblas_target_micro: target microarchitecture for the OpenHPC spec file (upstream takes native)
- openblas_additional_flags: additional flags for compilation

### Tensorflow

- build_tf_mpi: boolean, build tensorflow with mpicc or not
- tensorflow.cxx_optimizations : dictionary with flags to be given to g++ during the build
- tensroflow.c_optimizations : dictionary with flags to be given to gcc during the build

##### TensorFlow v2 and GCC

On CentOS8 aarch64 there is an issue with building v2 using the default gcc toolchain. Latest toolchains work fine.

## Conclusion

There is still work to be done and parts of the stack to be added to this build.
SciPy, Python itself, Keras, maybe even GCC (if one can find a decent enough spec, hopefully OpenHPC v2 will make this a lot easier).

The next step is to add support for MLperf or a standard benchmark suite that would allow for assessment of the relevancy of optimizations used.
Then optimization iterations can be done.

SVE support will first need to focus on OpenBLAS, then NumPy/SciPy, as well as TensorFlow's kernels.

Another task is to make sure that the recipes are able to build "master" branch from TensorFlow github repo. This shouldn't be much of an issue as far as dependencies are concerned (bazel rpm spec should build 1.1.0).

The process of optimizations, which entails a matrix job (as in, various optimizations on various dimensions, each point corresponding to one build and benchmark job).
Analysis of the results of each build might also need a tool to plot them.

## Note To Developers

Thanks to Ansible straightforward and readable syntax, I recommend reading the tasks to get a feel on the flow of the play.
Care as also been taken to make the variables explicit on their function.
