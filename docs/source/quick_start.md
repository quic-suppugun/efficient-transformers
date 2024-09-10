
QEfficient Library was designed with one goal: 

**To make onboarding of models inference straightforward for any Transformer architecture, while leveraging the complete power of Cloud AI platform**

To achieve this, we have 2 levels of APIs, with different levels of abstraction.
1. Command line interface abstracts away complex details, offering a simpler interface. They're ideal for quick development and prototyping. If you're new to a technology or want to minimize coding effort.

2. Python high level APIs offer more granular control, ideal for when customization is necessary.

## Transformed models and QPC storage

By default, the library exported models and Qaic Program Container (QPC) files, which are compiled and inference-ready model binaries generated by the compiler, are stored in `~/.cache/qeff_cache`. You can customize this storage path using the following environment variables:

1. **QEFF_HOME**: If this variable is set, its path will be used for storing models and QPC files.
2. **XDG_CACHE_HOME**: If `QEFF_HOME` is not set but `XDG_CACHE_HOME` is provided, this path will be used instead. Note that setting `XDG_CACHE_HOME` will reroute the entire `~/.cache` directory to the specified folder, including HF models.
3. **Default**: If neither `QEFF_HOME` nor `XDG_CACHE_HOME` are set, the default path `~/.cache/qeff_cache` will be used.


## Command Line Interface

```{NOTE}
Use ``bash terminal``, else if using ``ZSH terminal`` then ``device_group``should be in single quotes e.g.  ``'--device_group [0]'``
```

### QEfficient.cloud.infer

This is the single e2e CLI API, which takes `model_card` name as input along with other compilation arguments. Check [Infer API doc](infer_api) for more details.

* HuggingFace model files Download → Optimize for Cloud AI 100 → Export to `ONNX` → Compile on Cloud AI 100 → [Execute](#qefficientcloudexecute)
* It skips the export/compile stage based if `ONNX` or `qpc` files are found. If you use infer second time with different compilation arguments, it will automatically skip `ONNX` model creation and directly jump to compile stage.


```bash
# Check out the options using the help
python -m QEfficient.cloud.infer --help
python -m QEfficient.cloud.infer --model_name gpt2 --batch_size 1 --prompt_len 32 --ctx_len 128 --mxfp6 --num_cores 16 --device_group [0] --prompt "My name is" --mos 1 --aic_enable_depth_first  
```
If executing for batch size>1,
You can pass input prompts in single string but separate with pipe (|) symbol". Example below

```bash
python -m QEfficient.cloud.infer --model_name gpt2 --batch_size 3 --prompt_len 32 --ctx_len 128 --num_cores 16 --device_group [0] --prompt "My name is|The flat earth 
theory is the belief that|The sun rises from" --mxfp6 --mos 1 --aic_enable_depth_first
```

You can also pass path of txt file with input prompts when you want to run inference on lot of prompts, Example below, sample txt file(prompts.txt) is present in examples folder.

```bash
python -m QEfficient.cloud.infer --model_name gpt2 --batch_size 3 --prompt_len 32 --ctx_len 128 --num_cores 16 --device_group [0] --prompts_txt_file_path examples/prompts.txt --mxfp6 --mos 1 --aic_enable_depth_first  
```

### QEfficient.cloud.execute
You can first run `infer` API and then use `execute` to run the pre-compiled model on Cloud AI 100 cards.
Once we have compiled the QPC, we can now use the precompiled QPC in execute API to run for different prompts. Make sure to pass same `--device_group` as used during infer. Refer [Execute API doc](execute_api) for more details.

```bash
python -m QEfficient.cloud.execute --model_name gpt2 --qpc_path qeff_models/gpt2/qpc_16cores_1BS_32PL_128CL_1devices_mxfp6/qpcs --prompt "Once upon a time in" --device_group [0]  
```

### Multi-Qranium Inference
You can also enable MQ, just based on the number of devices. Based on the `--device-group` as input it will create TS config on the fly. If `--device-group [0,1]` it will create TS config for 2 devices and use it for compilation, if `--device-group [0]` then TS compilation is skipped and single soc execution is enabled.

```bash
python -m QEfficient.cloud.infer --model_name Salesforce/codegen-2B-mono --batch_size 1 --prompt_len 32 --ctx_len 128 --mxfp6 --num_cores 16 --device-group [0,1] --prompt "def fibonacci(n):" --mos 2 --aic_enable_depth_first  
``` 
Above step will save the `qpc` files under `efficient-transformers/qeff_models/{model_card_name}`, you can use the execute API to run for different prompts. This will automatically pick the pre-compiled `qpc` files.

```bash
python -m QEfficient.cloud.execute --model_name Salesforce/codegen-2B-mono --qpc-path qeff_models/Salesforce/codegen-2B-mono/qpc_16cores_1BS_32PL_128CL_2devices_mxfp6/qpcs --prompt "def binary_search(array: np.array, k: int):" --device-group [0,1] 
```

To disable MQ, just pass single soc like below, below step will compile the model again and reuse the `ONNX` file as only compilation argument are different from above commands.

```bash
python -m QEfficient.cloud.infer --model_name gpt2 --batch_size 1 --prompt_len 32 --ctx_len 128 --mxfp6 --num_cores 16 --device-group [0] --prompt "My name is" --mos 1 --aic_enable_depth_first
```


### Continuous Batching 

Users can compile a model utilizing the continuous batching feature by specifying full_batch_size <full_batch_size_value> in the infer and compiler APIs. If full_batch_size is not provided, the model will be compiled in the regular way.

When enabling continuous batching, batch size should not be specified.

Users can leverage multi-Qranium and other supported features along with continuous batching.

```bash
python -m QEfficient.cloud.infer --model_name TinyLlama/TinyLlama_v1.1 --batch_size 3 --prompt_len 32 --ctx_len 128 --num_cores 16 --device_group [0] --prompt "My name is|The flat earth 
theory is the belief that|The sun rises from" --mxfp6 --mos 1 --aic_enable_depth_first --full_batch_size 3
```
## Python API

### 1.  Model download and Optimize for Cloud AI 100
If your models falls into the model architectures that are [already supported](validated_models), Below steps should work fine.
Please raise an [issue](https://github.com/quic/efficient-transformers/issues), in case of trouble.



```Python
# Initiate the Original Transformer model
# import os

from QEfficient import QEFFAutoModelForCausalLM as AutoModelForCausalLM

# Please uncomment and use appropriate Cache Directory for transformers, in case you don't want to use default ~/.cache dir.
# os.environ["TRANSFORMERS_CACHE"] = "/local/mnt/workspace/hf_cache"

# ROOT_DIR = os.path.dirname(os.path.abspath(""))
# CACHE_DIR = os.path.join(ROOT_DIR, "tmp") #, you can use a different location for just one model by passing this param as cache_dir in below API.

# Model-Card name (This is HF Model Card name) : https://huggingface.co/gpt2-xl
model_name = "gpt2"  # Similar, we can change model name and generate corresponding models, if we have added the support in the lib.

qeff_model = AutoModelForCausalLM.from_pretrained(model_name)
print(f"{model_name} optimized for AI 100 \n", qeff_model)
```

### 2. Export and Compile with one API

Use the qualcomm_efficient_converter API to export the KV transformed Model to ONNX and Verify on Torch.

```Python
# We can now export the modified models to ONNX framework
# This will generate single ONNX Model for both Prefill and Decode Variations which are optimized for
# Cloud AI 100 Platform.

# While generating the ONNX model, this will clip the overflow constants to fp16
# Verify the model on ONNXRuntime vs Pytorch

# Then generate inputs and customio yaml file required for compilation.
# Compile the model for provided compilation arguments
# Please use platform SDk to Check num_cores for your card.

generated_qpc_path = qeff_model.compile(
    num_cores=14,
    mxfp6=True,
    device_group=[0],
)
```

### 3. Execute

Benchmark the model on Cloud AI 100, run the infer API to print tokens and tok/sec

```Python
# post compilation, we can print the latency stats for the kv models, We provide API to print token and Latency stats on AI 100
# We need the compiled prefill and decode qpc to compute the token generated, This is based on Greedy Sampling Approach

qeff_model.generate(prompts=["My name is"])
```
End to End demo examples for various models are available in **notebooks** directory. Please check them out.