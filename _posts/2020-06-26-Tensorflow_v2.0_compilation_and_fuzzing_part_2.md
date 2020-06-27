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

After compiling `fuzz_target.cc` with an option `-fsanitize=address,fuzzer`, you will get an executable. Running it will LibFuzzer keep generating byte streams and feeding them into the parameter `const uint8_t *Data`,  then the data is handled by the interface you want to fuzz. LibFuzzer will stop once it finds a crash. Fuzzing has become a popular and effective software testing approach since its birth, I hope to give a more detailed introduction of it in the future.

A casual `grep -r 'fuzz' ./` leads to an exhilarated finding. There is a built-in fuzzing module lying in the deep Tensorflow source code, which is `tensorflow/core/kernels/fuzzing`.  Looking into it, I see following files:

```bash
➜  fuzzing git:(r2.0) ✗ pwd
/home/dsk/xxx/tensorflow/tensorflow/core/kernels/fuzzing
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

Take a closer look at `fuzz_session.h`. In it a class `FuzzSession` is defined

```c++
class FuzzSession {
 public:
  FuzzSession() : initialized_(false) {}
  virtual ~FuzzSession() {}

  // Constructs a Graph using the supplied Scope.
  // By convention, the graph should have inputs named "input1", ...
  // "inputN", and one output node, named "output".
  // Users of FuzzSession should override this method to create their graph.
  virtual void BuildGraph(const Scope& scope) = 0;

  virtual void FuzzImpl(const uint8_t* data, size_t size) = 0;

  // Implements the logic that converts an opaque byte buffer
  // from the fuzzer to Tensor inputs to the graph.  Users must override.
  //
  //virtual void SINGLE_INPUT_OP_BUILDER(const uint8_t* data, size_t size) = 0;

  // Initializes the FuzzSession.  Not safe for multithreading.
  // Separate init function because the call to virtual BuildGraphDef
  // can't be put into the constructor.
  Status InitIfNeeded() {
    if (initialized_) {
      return Status::OK();
    }
    initialized_ = true;

    Scope root = Scope::DisabledShapeInferenceScope().ExitOnError();
    SessionOptions options;
    session_ = std::unique_ptr<Session>(NewSession(options));

    BuildGraph(root);

    GraphDef graph_def;
    TF_CHECK_OK(root.ToGraphDef(&graph_def));

    Status status = session_->Create(graph_def);
    if (!status.ok()) {
      // This is FATAL, because this code is designed to fuzz an op
      // within a session.  Failure to create the session means we
      // can't send any data to the op.
      LOG(FATAL) << "Could not create session: " << status.error_message();
    }
    return status;
  }

  // Runs the TF session by pulling on the "output" node, attaching
  // the supplied input_tensor to the input node(s), and discarding
  // any returned output.
  // Note: We are ignoring Status from Run here since fuzzers don't need to
  // check it (as that will slow them down and printing/logging is useless).
  void RunInputs(const std::vector<std::pair<string, Tensor> >& inputs) {
    RunInputsWithStatus(inputs).IgnoreError();
  }

  // Same as RunInputs but don't ignore status
  Status RunInputsWithStatus(
      const std::vector<std::pair<string, Tensor> >& inputs) {
    return session_->Run(inputs, {}, {"output"}, nullptr);
  }

  // Dispatches to FuzzImpl;  small amount of sugar to keep the code
  // of the per-op fuzzers tiny.
  int Fuzz(const uint8_t* data, size_t size) {
    Status status = InitIfNeeded();
    TF_CHECK_OK(status) << "Fuzzer graph initialization failed: "
                        << status.error_message();
    // No return value from fuzzing:  Success is defined as "did not
    // crash".  The actual application results are irrelevant.
    FuzzImpl(data, size);
    return 0;
  }

 private:
  bool initialized_;
  std::unique_ptr<Session> session_;
};
```

There are two virtual functions very important -- `BuildGraph` and `FuzzImpl`.

`BuildGraph` is used to organize operands(op) to be tested into a computation graph, since computation graph is the execution unit of Tensorflow. `FuzzImpl` implements the logic that converts the raw byte buffer provided by LibFuzzer to `Tensors`, which are inputs of different operands.

`fuzz_session.h` also gives an example fuzzer for ops that take a single string, which is as below:

```c++
class FuzzStringInputOp : public FuzzSession {
  //void FuzzImpl(const uint8_t* data, size_t size) final {
  void FuzzImpl(const uint8_t* data, size_t size){
    Tensor input_tensor(tensorflow::DT_STRING, TensorShape({}));
    input_tensor.scalar<string>()() =
        string(reinterpret_cast<const char*>(data), size);
    RunInputs({{"input", input_tensor}});
  }
};
```

So how to inherit the `FuzzSession` class to construct a real fuzzer?

Let's take a look at `one_hot_fuzz.cc`.  

```c++
class FuzzOneHot : public FuzzSession {
  void BuildGraph(const Scope& scope) override {
    auto input =
        tensorflow::ops::Placeholder(scope.WithOpName("input"), DT_UINT8);
    auto depth =
        tensorflow::ops::Placeholder(scope.WithOpName("depth"), DT_INT32);
    auto on = tensorflow::ops::Placeholder(scope.WithOpName("on"), DT_UINT8);
    auto off = tensorflow::ops::Placeholder(scope.WithOpName("off"), DT_UINT8);
    (void)tensorflow::ops::OneHot(scope.WithOpName("output"), input, depth, on,
                                  off);
  }

```

Tensorflow API reference lists the detailed usage of `OneHot` op.

```c++
OneHot(const ::tensorflow::Scope & scope, ::tensorflow::Input indices, ::tensorflow::Input depth, ::tensorflow::Input on_value, ::tensorflow::Input off_value)
OneHot(const ::tensorflow::Scope & scope, ::tensorflow::Input indices, ::tensorflow::Input depth, ::tensorflow::Input on_value, ::tensorflow::Input off_value, const OneHot::Attrs & attrs)
```

Then, in the function `BuildGraph`, four `Placeholder`s are created to act as different parameters of the constructor function. Here comes the question, what are the values of these four variables (`input`, `depth`, `on` and `off`)? `FuzzImpl` will give you the answer.

```c++
  void FuzzImpl(const uint8_t* data, size_t size) override {
    int64 input_size;
    int32 depth;
    uint8 on, off;
    const uint8_t* input_data;

    if (size > 3) {
      depth = static_cast<int32>(data[0]);
      on = data[1];
      off = data[2];
      input_size = static_cast<int64>(size - 3);
      input_data = data + 3;
    } else {
      depth = 1;
      on = 1;
      off = 0;
      input_size = static_cast<int64>(size);
      input_data = data;
    }

    Tensor input_tensor(tensorflow::DT_UINT8, TensorShape({input_size}));
    Tensor depth_tensor(tensorflow::DT_INT32, TensorShape({}));
    Tensor on_tensor(tensorflow::DT_UINT8, TensorShape({}));
    Tensor off_tensor(tensorflow::DT_UINT8, TensorShape({}));

    auto flat_tensor = input_tensor.flat<uint8>();
    for (size_t i = 0; i < input_size; i++) {
      flat_tensor(i) = input_data[i];
    }
    depth_tensor.scalar<int32>()() = depth;
    on_tensor.scalar<uint8>()() = on;
    off_tensor.scalar<uint8>()() = off;

    RunInputs({{"input", input_tensor},
               {"depth", depth_tensor},
               {"on", on_tensor},
               {"off", off_tensor}});
  }
```

The `FuzzImpl` has the same parameter setting as `LLVMFuzzerTestOneInput`,  for the elegance of the `FuzzSession` class. In fact, its `data` and `size` are all the same as those of `LLVMFuzzerTestOntInput`. In `FuzzImpl`, the `data`(byte buffer) is divided and casted to the data type required by previously-built computation graph  and then fed to four `Placeholder`s  --`input_tensor`, `depth_tensor`, `on_tensor` and `off_tensor`. Then the four `Placeholders` are passed to `Session.run()` as the arguments. After that, a live `OneHot` object is created.

So we can conclude the general idea presented by `tensorflow/core/kernels/fuzzing` module. It leverages libFuzzer to fuzz kernel operands (that's why it resides in `core/kernels` module). To fuzz an op, we need to build a computation graph and make  the tested op as an intermediate node of the graph. Then we need to design the strategy of splitting the raw byte buffer provided by libFuzzer and use the data to initialize the Tensors. The last step is to run a `Session` to execute the computation graph.

However, as a big project with more than 630,000 lines of code, there are still a lot of attacking surfaces of Tensorflow. In the future posts, I will share my idea of fuzzing other aspects around Tensorflow.

