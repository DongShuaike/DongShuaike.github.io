---
layout: post
title:  "Tensorflow v2.0 compilation from source code and fuzzing (part 1)"
date:   2020-06-25 22:18:09 +0800
categories: tensorflow
---



Recently I was working on a new project around **deep learning framework fuzzing** which seems very exciting. I choose **Tensorflow** as the target since its complexity and variety of components.

The figure below shows the whole architecture of Tensorflow. From it we know what we usually use in normal days are only a small part of the whole system (mainly **Python client**). To make the fuzzing deeper, I decide to target Tensorflow low-level functions, especially **C APIs** and those **kernel implementations**.

![img](/images/Tensorflow.png)



The first thing to do is to compile its source code, during which I can get familiar with overall structure of Tensorflow and learn how to play with its C-implemented interfaces. However, the journey  was not smooth.

In this post I will record all "holes" I got stuck at and "ad hoc" way to solve them. I hope my solution can help those also compiling Tensorflow. Since I have **no GPUs** on my poor PC, I take the CPU-version instead and  unaware of GPU-related issues.

### Let's start 

#### Tested build configurations

[Tested build configurations on official website ](https://www.tensorflow.org/install/source#linux)

The first and most important reference I should mention is the tested build configuration page on Tensorflow's official website. One of my bad experiences is to copy the procedures on a `gcc-7.5` PC to a `gcc-9.3` laptop.  You are recommended to make your environment consistent to ones tested by the official. For me, it is `bazel 0.26.1`+`Python 3.6` + `tensorflow 2.0.0`. (ubuntu 18.04)

#### bazel workspace.bzl file

Google has changed its default building tool from cmake to their own **bazel**. I spent sometime getting familiarized with its syntax.  Each project built with bazel should have a file called `workspace.bzl` . Inside it listed **how to acquire other projects' sources.**  Below I copies some code snippets of it, which are about the `eigen` library, a very famous C++ template library for linear algebra.

```python
tf_http_archive(
    name = "eigen_archive",
    build_file = clean_dep("//third_party:eigen.BUILD"),
    patch_file = clean_dep("//third_party/eigen3:gpu_packet_math.patch"),
    sha256 = "f3d69ac773ecaf3602cb940040390d4e71a501bb145ca9e01ce5464cf6d4eb68",
    strip_prefix = "eigen-eigen-049af2f56331",
    urls = [
        "https://storage.googleapis.com/mirror.tensorflow.org/bitbucket.org/eigen/eigen/get/049af2f56331.tar.gz",
        "https://bitbucket.org/eigen/eigen/get/049af2f56331.tar.gz",
    ],
)
```

 Although there are materials saying `bazel build` will automatically install all needed external and third-party dependencies listed in `tensorflow/tensorflow/workspace.bzl`, ~~I fail to reach that (due to the unfamiliarity with bazel I think). **will update this part once I fix it.**~~

### Update on Jun 26

***Bazel provides a convenient way to download --> compile --> install needed third-party libraries.*** (I finally figured it out :)

Just modify the `BUILD` file in `tensorflow/third_party`by adding `exports_files` statements.

```python
licenses(["notice"]) # do not change

exports_files(['com_google_absl.BUILD']) # add what you like to install
...
```

According to your configuration, Tensorflow provides a set of third-party library BUILD files for you to install.

```bash
➜  third_party git:(r2.0) ✗ ls *.BUILD
arm_neon_2_x86_sse.BUILD  double_conversion.BUILD  jsoncpp.BUILD     pprof.BUILD      tflite_mobilenet.BUILD
astor.BUILD               eigen.BUILD              libxsmm.BUILD     pybind11.BUILD   tflite_mobilenet_float.BUILD
backports_weakref.BUILD   enum34.BUILD             linenoise.BUILD   rocprim.BUILD    tflite_mobilenet_quant.BUILD
codegen.BUILD             farmhash.BUILD           lmdb.BUILD        six.BUILD        tflite_ovic_testdata.BUILD
com_google_absl.BUILD     functools32.BUILD        nanopb.BUILD      snappy.BUILD     tflite_smartreply.BUILD
cub.BUILD                 gast.BUILD               opt_einsum.BUILD  sqlite.BUILD     wrapt.BUILD
curl.BUILD                gif.BUILD                pcre.BUILD        swig.BUILD       zlib.BUILD
cython.BUILD              googleapis.BUILD         png.BUILD         termcolor.BUILD
```

#### Setup before bazel building

1. Python and Tensorflow package dependencies.

   ```python
   sudo apt install python-dev python-pip # python3-dev and python3-pip
   pip install -U --user pip six numpy wheel setuptools mock 'future>=0.17.1'
   pip install -U --user keras_applications --no-deps
   pip install -U --user keras_preprocessing --no-deps
   ```

   If you are careless and forgot to install **keras\*** related dependencies above, you will get following errors when you are building python `.whl`  files (`bazel build --config=opt //tensorflow/tools/pip_package:build_pip_package`). 

   ```python
   ModuleNotFoundError: No module named 'keras_preprocessing'
   Target //tensorflow/tools/pip_package:build_pip_package failed to build
   Use --verbose_failures to see the command lines of failed build steps
   
   ```

2. Google protocol buffers

   [Google protocol buffers](https://developers.google.com/protocol-buffers)

   are Google's serializing libraries. 

   It is widely-used in Tensorflow project due to its fascinating **code generation** feature. ~~You have to install it before compiling Tensorflow~~. To notice, **you'd better install the version listed in `workspace.bzl` file** . For me, it is

   ```python
    PROTOBUF_URLS = [
           "https://storage.googleapis.com/mirror.tensorflow.org/github.com/protocolbuffers/protobuf/archive/310ba5ee72661c081129eb878c1bbcec936b20f0.tar.gz",
           "https://github.com/protocolbuffers/protobuf/archive/310ba5ee72661c081129eb878c1bbcec936b20f0.tar.gz",
       ]
       PROTOBUF_SHA256 = "b9e92f9af8819bbbc514e2902aec860415b70209f31dfc8c4fa72515a5df9d59"
       PROTOBUF_STRIP_PREFIX = "protobuf-310ba5ee72661c081129eb878c1bbcec936b20f0"
   ```

   Copy the URL and `wget` it, then follow the `ReadMe` file to install it to your system. Finally, the protobuf libraries will be installed to `/usr/local/lib`.

   Although you can use bazel to install it temporarily, I still would like to install it by myself for the future use.

3. Eigen library

   ***(UPDATE on Jun 26, you can also use `exports_files` approach mentioned above to temporarily download and install Eigen library)***

   Different from protocol buffers, we don't need to `make` and `make install` Eigen library. Just download the needed version and decompress it to a folder. Add the path involving `eigen3` to the environment variable `CPLUS_INCLUDE_PATH` like below:

   ```bash
   export CPLUS_INCLUDE_PATH="$HOME/Tools/include/:$CPLUS_INCLUDE_PATH"
   export CPLUS_INCLUDE_PATH="$HOME/Tools/include/eigen3/:$CPLUS_INCLUDE_PATH"
   ```

   The structure of `eigen3` folder is like

   ```bash
   ➜  include ls eigen3           
   bench            COPYING.README        INSTALL
   blas             CTestConfig.cmake     lapack
   cmake            CTestCustom.cmake.in  README.md
   CMakeLists.txt   debug                 scripts
   COPYING.BSD      demos                 signature_of_eigen3_matrix_library
   COPYING.GPL      doc                   test
   COPYING.LGPL     Eigen                 unsupported
   COPYING.MINPACK  eigen3.pc.in
   COPYING.MPL2     failtest
   ```

   If you meet the error `... Partial template specialization is not more specialized thant primary template ...`, please take a look at the solution [xcode 9 falls to build partial template specialization in c](https://stackoverflow.com/questions/46356153/xcode-9-falls-to-build-partial-template-specialization-in-c/46382971). I tested the last answer manually and it worked.

   To conclude, you should do following modifications (Assume you put eigen3 folder in `/usr/include`):

   In the file `/usr/include/eigen3/unsupported/Eigen/CXX11/src/Tensor/TensorStorage.h`, change the old** style to **new** style:

   **Old**:

   ```c
   template<typename T, int Options_, typename FixedDimensions>
   class TensorStorage<T, FixedDimensions, Options_>
   {
   ```

   **New**:

   ```c
   template<typename T, typename FixedDimensions, int Options_>      // swap the templ params to match the declaration
   class TensorStorage                                               // drop the specialization (because it didn't!)
   {
   ```

4. abseil library

   Another library you should have is [abseil-cpp](https://github.com/abseil/abseil-cpp) . ~~`git clone` this project and put it under the root of Tensorflow (where you have another `tensorflow` subfolder, `third_party` and `tools`). Also note to add a soft link to it.~~ ***(This is the old solution, sometimes it works, but sometimes it  brings a lot of trouble, see section "Other Issues" below, use Bazel to install it)***

   ```bash
   $ git clone https://github.com/abseil/abseil-cpp.git
   $ ln -s abseil-cpp/absl ./absl
   ```

   Without abseil library, you may meet the error saying something like `fatal error: absl/strings/string_view.h: No such file or directory`.

   The solution is provided in the following thread:

   [Newly included absl headers are missing from the include path](https://github.com/tensorflow/tensorflow/issues/22007)

### Build it!

To test C implementations in Tensorflow, we need compile some **shared library files**, in my case, **tensorflow.so**, **tensorflow_cc.so** and **tensorflow_framework.so**. All the descriptions of these three libraries are listed in `tensorflow/tensorflow/BUILD` file. Reading the `deps` part you will have a rough idea of what are those libraries for. 

`libtensorflow.so`:

```python
tf_cc_shared_object(
    name = "tensorflow",
    linkopts = select({
        "//tensorflow:macos": [
            "-Wl,-exported_symbols_list,$(location //tensorflow/c:exported_symbols.lds)",
        ],
        "//tensorflow:windows": [
        ],
        "//conditions:default": [
            "-z defs",
            "-Wl,--version-script,$(location //tensorflow/c:version_script.lds)",
        ],
    }),
    per_os_targets = True,
    soversion = VERSION,
    visibility = ["//visibility:public"],
    # add win_def_file for tensorflow
    win_def_file = select({
        # We need this DEF file to properly export symbols on Windows
        "//tensorflow:windows": ":tensorflow_filtered_def_file",
        "//conditions:default": None,
    }),
    deps = [
        "//tensorflow/c:c_api",
        "//tensorflow/c:c_api_experimental",
        "//tensorflow/c:exported_symbols.lds",
        "//tensorflow/c:version_script.lds",
        "//tensorflow/c/eager:c_api",
        "//tensorflow/core:tensorflow",
        "//tensorflow/core/distributed_runtime/rpc:grpc_session",
    ],
)

```

`libtensorflow_cc.so`

```python
tf_cc_shared_object(
    name = "tensorflow_cc",
    linkopts = select({
        "//tensorflow:macos": [
            "-Wl,-exported_symbols_list,$(location //tensorflow:tf_exported_symbols.lds)",
        ],
        "//tensorflow:windows": [],
        "//conditions:default": [
            "-z defs",
            "-Wl,--version-script,$(location //tensorflow:tf_version_script.lds)",
        ],
    }),
    per_os_targets = True,
    soversion = VERSION,
    visibility = ["//visibility:public"],
    # add win_def_file for tensorflow_cc
    win_def_file = select({
        # We need this DEF file to properly export symbols on Windows
        "//tensorflow:windows": ":tensorflow_filtered_def_file",
        "//conditions:default": None,
    }),
    deps = [
        "//tensorflow:tf_exported_symbols.lds",
        "//tensorflow:tf_version_script.lds",
        "//tensorflow/c:c_api",
        "//tensorflow/c/eager:c_api",
        "//tensorflow/cc:cc_ops",
        "//tensorflow/cc:client_session",
        "//tensorflow/cc:scope",
        "//tensorflow/cc/profiler",
        "//tensorflow/core:tensorflow",
    ] + if_ngraph(["@ngraph_tf//:ngraph_tf"]),
)

```

`libtensorflow_framework.so`

```python
tf_cc_shared_object(
    name = "tensorflow_framework",
    framework_so = [],
    linkopts = select({
        "//tensorflow:macos": [],
        "//tensorflow:windows": [],
        "//tensorflow:freebsd": [
            "-Wl,--version-script,$(location //tensorflow:tf_framework_version_script.lds)",
            "-lexecinfo",
        ],
        "//conditions:default": [
            "-Wl,--version-script,$(location //tensorflow:tf_framework_version_script.lds)",
        ],
    }),
    linkstatic = 1,
    per_os_targets = True,
    soversion = VERSION,
    visibility = ["//visibility:public"],
    deps = [
        "//tensorflow/cc/saved_model:loader_lite_impl",
        "//tensorflow/core:core_cpu_impl",
        "//tensorflow/core:framework_internal_impl",
        "//tensorflow/core:gpu_runtime_impl",
        "//tensorflow/core/grappler/optimizers:custom_graph_optimizer_registry_impl",
        "//tensorflow/core:lib_internal_impl",
        "//tensorflow/stream_executor:stream_executor_impl",
        "//tensorflow:tf_framework_version_script.lds",
    ] + tf_additional_binary_deps(),
```

The commands to build them three are :

```
bazel build //tensorflow:tensorflow
bazel build //tensorflow:tensorflow_cc
bazel build //tensorflow:tensorflow_framework
```

After around 1~2 hours (depending on your hardware), you will get these three shared library objects in `tensorflow/bazel-bin/tensorflow`.

### Build python `.whl` binary

If you want to compile the python interfaces as well, you need to run 

```bash
bazel build //tensorflow/tools/pip_package:build_pip_package
# wait for its completion
./bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg
```

If everything is fine, you will find a `*.whl` file in `/tmp/tensorflow_pkg`. Use `pip install *.whl` to install it and enjoy.

### Other weird issues

If you have managed the above, congratulations! Now you can link them and write your own C programs. However, when I "migrated" my above procedures to one of my laptops, I met other issues. I will list them by errors.

1. Annoying inconsistency issue due to the manual installation of abseil-cpp library

   I once met the following errors:

   ```bash
   ERROR: /home/dsk/xxx/tensorflow/tensorflow/core/platform/BUILD:98:1: undeclared inclusion(s) in rule '//tensorflow/core/platform:cpu_info':
   
   this rule is missing dependency declarations for the following files included by 'tensorflow/core/platform/cpu_info.cc':
     'absl/base/log_severity.h'
     'absl/base/attributes.h'
     'absl/base/config.h'
     'absl/base/options.h'
     'absl/base/policy_checks.h'
     'absl/strings/string_view.h'
     'absl/base/internal/throw_delegate.h'
     'absl/base/macros.h'
     'absl/base/optimization.h'
     'absl/base/port.h'
   Target //tensorflow/tools/pip_package:build_pip_package failed to build
   Use --verbose_failures to see the command lines of failed build steps.
   
   ```

   It took me a looooooong time debugging this and I still cannot fix it. However, after I figured out the "bazel way" to install abseil-cpp library, everything becomes fine.  Just add the following line to `tensorflow/third_party/BUILD`:

   ```python
   exports_files(['com_google_absl.BUILD'])
   ```

2. grpc issue

   If you met the following error:

   ```bash
   ERROR: /home/dsk/.cache/bazel/_bazel_dsk/34dcb22fe3c9a03150797364c441c079/external/grpc/BUILD:507:1: C++ compilation of rule '@grpc//:gpr_base' failed (Exit 1)
   external/grpc/src/core/lib/gpr/log_linux.cc:43:13: error: ambiguating new declaration of 'long int gettid()'
      43 | static long gettid(void) { return syscall(__NR_gettid); }
   ```

   Try the following solution, I got this from [Build tensorflow error as "external/grpc/src/core/lib/gpr/log_linux.cc:43:13: error: ambiguating new declaration of 'long int gettid()'"](https://github.com/clearlinux/distribution/issues/1151).

   Step 1. 

   ```bash
   cd tensorflow
   wget https://raw.githubusercontent.com/clearlinux-pkgs/tensorflow/master/Add-grpc-fix-for-gettid.patch
   patch -p1 <Add-grpc-fix-for-gettid.patch
   ```

   Step 2.

   ```bash
   wget https://nomeroff.net.ua/tf/Rename-gettid-functions.patch
   cp ./Rename-gettid-functions.patch ./third_party/Rename-gettid-functions.patch
   ```

3. Errors due to wrong Python version

   I met following errors when executing `bazel build //tensorflow/tools/build_pip_package` with  `Python v3.8`, which is not among the tested configurations on Tensorflow official website.

   ```bash
   ERROR: /home/dsk/xxx/tensorflow/tensorflow/python/BUILD:341:1: C++ compilation of rule '//tensorflow/python:ndarray_tensor_bridge' failed (Exit 1)
   tensorflow/python/lib/core/ndarray_tensor_bridge.cc:108:1: error: cannot convert 'std::nullptr_t' to 'Py_ssize_t {aka long int}' in initialization
    };
    ^
   Target //tensorflow/tools/pip_package:build_pip_package failed to build
   
   ```

   I found the reason in  [vtk fails to build with Python 3.8](https://bugzilla.redhat.com/show_bug.cgi?id=1718837)

   > ```
   > In Python 3.8, the reserved "tp_print" slot was changed from a function pointer to a number, `Py_ssize_t tp_vectorcall_offset`.
   > In C, there is no "nullptr"; either a 0 or NULL casts automatically to both pointers and numbers.
   > 
   > Please use 0 instead of "nullptr" in the slot to be source-compatible both with Python 3.8 and previous versions.
   > 
   > See this as an example PR: https://github.com/YafaRay/Core/pull/114
   > 
   > -- Miro Hrončok 
   > ```

   You should **re-configure** the whole project to switch to an older Python, which means everything will be compiled once again :-).

   

   ### Conclusion

   The compilation is not easy and I really think the introduction section on the Tensorflow official website is somewhat too abbreviate. I hope this post can help you and in the next post, I will introduce some of my practice on fuzzing Tensorflow source code.

   





