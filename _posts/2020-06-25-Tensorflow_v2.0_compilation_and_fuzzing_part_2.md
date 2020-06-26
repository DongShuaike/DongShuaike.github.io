---

layout: post
title:  "Tensorflow v2.0 compilation from source code and fuzzing (part 2)"
date:   2020-06-26 19:32:09 +0800
categories: tensorflow

---



In the last post I introduced how to build Tensorflow from source code. I then searched a lot on the Internet to locate any useful materials about fuzzing Tensorflow.

Surprisingly, an article in 2017 attracted my eyes -- [Finding Bugs in TensorFlow with LibFuzzer](https://da-data.blogspot.com/2017/01/finding-bugs-in-tensorflow-with.html). The blog is interesting and tells how the author found vulnerabilities with the help of a popular tool -- LibFuzzer. Although no obvious clues (demos) are presented, it shows a right direction.

[LibFuzzer](https://llvm.org/docs/LibFuzzer.html) is a built-in module of LLVM. It provides a series of concise APIs to help construct your own fuzzer. The easiest case is to wrap the interface you want to fuzz with `LLVMFuzerTestOneInput` , like the code snippet below:

```c
// fuzz_target.cc
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {
  DoSomethingInterestingWithMyAPI(Data, Size);
  return 0;  // Non-zero return values are reserved for future use.
}
```

After compiling `fuzz_target.cc` with an option `-fsanitize=address,fuzzer`, you will get a executable. When you run it, LibFuzzer will keep generating byte streams and feed them into the parameter `const uint8_t *Data`,  then the data is handled by the interface you want to fuzz. LibFuzzer will stop running once it finds a crash. Fuzzing has become a popular and effective software testing approach since its birth, I hope to give a more detailed introduction of it in the future.

A casual `grep -r 'fuzz' ./` leads to an exhilarated finding. There is a built-in fuzzing module lying in the deep Tensorflow source code, which is `tensorflow/core/kernels/fuzzing`.  Looking into it, I see following files:

```bash
➜  fuzzing git:(r2.0) ✗ pwd
/home/dsk/Alipay/tensorflow/tensorflow/core/kernels/fuzzing
➜  fuzzing git:(r2.0) ✗ ls
BUILD                        encode_jpeg_fuzz.cc
check_numerics_fuzz.cc       example_proto_fast_parsing_fuzz.cc
corpus                       fuzz_session.h
decode_base64_fuzz.cc        identity_fuzz.cc
decode_bmp_fuzz.cc           one_hot_fuzz.cc
decode_compressed_fuzz.cc    parse_tensor_op_fuzz.cc
decode_json_example_fuzz.cc  scatter_nd_fuzz.cc
decode_png_fuzz.cc           string_split_fuzz.cc
decode_wav_fuzz.cc           string_split_v2_fuzz.cc
dictionaries                 string_to_number_fuzz.cc
encode_base64_fuzz.cc        tf_ops_fuzz_target_lib.bzl

```







