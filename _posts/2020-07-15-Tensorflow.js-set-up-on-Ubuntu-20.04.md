---
​--- 
layout: post
title:  "Tensorflow.js set up on Ubuntu 20.04"
date:   2020-07-15 18:32:09 +0800
categories: tensorflow
---

`TensorFlow.js` is a Javascript library to train and deploy ML models on browsers and Node.js. It takes several steps to set it up on `Ubuntu 20.04`

Step 1. Install `node.js` and `npm`

```bash
sudo apt-get install nodejs
sudo apt-get install npm
```

Step 2. Initiate a `node.js` project

```
npm init
```

Step 3. Install `TensorFlow.js`

```
npm install @tensorflow/tfjs
npm install @tensorflow/tfjs-node # install with C++ bindings
```

Step 4. Test with examples

I tested the installation with an official case.

```javascript
const tf = require('@tensorflow/tfjs');

// 可选加载绑定：
// 如果使用GPU运行，请使用'@tensorflow/tfjs-node-gpu'
require('@tensorflow/tfjs-node');

// 训练一个简单模型:
const model = tf.sequential();
model.add(tf.layers.dense({units: 100, activation: 'relu', inputShape: [10]}));
model.add(tf.layers.dense({units: 1, activation: 'linear'}));
model.compile({optimizer: 'sgd', loss: 'meanSquaredError'});

const xs = tf.randomNormal([100, 10]);
const ys = tf.randomNormal([100, 1]);

model.fit(xs, ys, {
  epochs: 100,
  callbacks: {
    onEpochEnd: (epoch, log) => console.log(`Epoch ${epoch}: loss = ${log.loss}`)
  }
});
  
```

Save a file called `test.js` and run with `node test.js`.

```bash
verriding the gradient for 'Max'
Overriding the gradient for 'OneHot'
Overriding the gradient for 'PadV2'
Overriding the gradient for 'SpaceToBatchND'
Overriding the gradient for 'SplitV'
node-pre-gyp info This Node instance does not support builds for N-API version 6
node-pre-gyp info This Node instance does not support builds for N-API version 6
2020-07-15 17:54:37.755169: I tensorflow/core/platform/cpu_feature_guard.cc:142] Your CPU supports instructions that this TensorFlow binary was not compiled to use: AVX2 FMA
2020-07-15 17:54:37.793073: I tensorflow/core/platform/profile_utils/cpu_utils.cc:94] CPU Frequency: 2195835000 Hz
2020-07-15 17:54:37.793702: I tensorflow/compiler/xla/service/service.cc:168] XLA service 0x1246000 initialized for platform Host (this does not guarantee that XLA will be used). Devices:
2020-07-15 17:54:37.793726: I tensorflow/compiler/xla/service/service.cc:176]   StreamExecutor device (0): Host, Default Version
Epoch 1 / 100
eta=0.0 ==============================================================================================> 
81ms 806us/step - loss=1.25 
Epoch 0: loss = 1.254112720489502
Epoch 2 / 100
eta=0.0 ==============================================================================================> 
34ms 341us/step - loss=1.23 
Epoch 1: loss = 1.2288624048233032
Epoch 3 / 100

```

If there are errors like 

```bash
/home/dsk/xxx/tensorboard/fuzzer_dsk/node_modules/@tensorflow/tfjs-node/dist/index.js:49
    throw new Error("The Node.js native addon module (tfjs_binding.node) can not "
    ^

Error: The Node.js native addon module (tfjs_binding.node) can not be found at path: /home/dsk/xxx/tensorboard/fuzzer_dsk/node_modules/@tensorflow/tfjs-node/lib/napi-v5/tfjs_binding.node. 
Please run command 'npm rebuild @tensorflow/tfjs-node build-addon-from-source' to rebuild the native addon module.
```

Run `npm rebuild @tensorflow/tfjs-node build-addon-from-source` as told.

If you met **python** errors during this saying `gyp==0.1 distribution was not found`, run `pip install git+https://chromium.googlesource.com/external/gyp` as shown in [this](https://stackoverflow.com/questions/40025591/the-gyp-0-1-distribution-was-not-found).

Test another case from [official document](https://www.tensorflow.org/js/tutorials/conversion/import_saved_model)

```typescript
import * as tf from '@tensorflow/tfjs';
import {loadGraphModel} from '@tensorflow/tfjs-converter';

const MODEL_URL = 'model_directory/model.json';

const model = await loadGraphModel(MODEL_URL);
const cat = document.getElementById('cat');
model.execute(tf.fromPixels(cat));
```

Save it as `test.ts` and compile it with `tsc test.ts`.

If there are errors saying `Accessors are only available when targeting ECMAScript 5 and higher`, use `tsc -t es5 test.ts` instead.

If there are errors around "WebGL2RenderingContext compatibility", like this [article](https://dev.to/dkp1903/solving-typescript-tensorflowjs-incompat-issues-4f80) shows, run `npm -i --save @types/webgl2` to fix it.

If there are errors like `Build: Cannot find type definition file for 'node'`,  run `npm install @types/node --save-dev` as shown in [this](https://stackoverflow.com/questions/43542710/buildcannot-find-type-definition-file-for-node).

After solving all issues, you will get a `test.js` file, then you can run it with `node test.js`.

