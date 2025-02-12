# Command

## Modify sequence length

切换到Qwen/compile/目录下后，手动将files/Qwen-VL-Chat/config.json中的`"seq_length": 2048,`修改为你需要的长度
```shell
cd /workspace/LLM-TPU/models/Qwen/compile
vi files/Qwen-VL-Chat/config.json
```

PS：
1. 由于导出的是静态onnx模型，所以必须手动修改为你所需要的长度

## Export onnx

```shell
cp files/Qwen-VL-Chat/* your_torch_path
export PYTHONPATH=your_torch_path:$PYTHONPATH
pip install transformers_stream_generator einops tiktoken accelerate transformers==4.31.0
```

### export basic onnx
```shell
python export_onnx.py --model_path your_torch_path --device cuda
```

### export onnx used for parallel demo
```shell
python export_onnx.py --model_path your_torch_path --device cuda --lmhead_with_topk 1
```

PS：
1. 最好使用cuda导出，cpu导出block的时候，会卡在第一个block，只能kill
2. your_torch_path：从官网下载的或者自己训练的模型的路径，例如./Qwen-7B-Chat

## Compile bmodel

```shell
pushd /path_to/tpu-mlir
source envsetup.sh
popd
```

### compile basic bmodel
```shell
./compile.sh --mode int4 --name qwen-vl-chat --addr_mode io_alone --seq_length 1024
```

### compile jacobi bmodel
```shell
./compile_jacobi.sh --mode int4 --name qwen-vl-chat --addr_mode io_alone --generation_mode sample --decode_mode jacobi --seq_length 1024
```

### compile bmodel for parallel demo
```shell
./compile.sh --mode int4 --name qwen-vl-chat --addr_mode io_alone --seq_length 1024 --num_device 8
```

PS：
1. mode：量化方式，目前支持fp16/bf16/int8/int4
2. name：模型名称，目前Qwen系列支持 Qwen-1.8B/Qwen-7B/Qwen-14B/Qwen-VL-Chat
3. addr_mode：地址分配方式，可以使用io_alone方式来加速
4. generation_mode：token采样模式，为空时，使用greedy search；为sample时，使用topk+topp；为all时使用topk + topp + temperature + repeat_penalty + max_new_tokens，并且可以作为参数传入
5. decode_mode：编码模式，为空时，使用正常编码；为jacobi时，使用jacobi编码，只有前面使用export_onnx_jacobi.py时，用jacobi才有意义
6. seq_length：模型支持的最大token长度

## Run Demo

如果是pcie，建议新建一个docker，与编译bmodel的docker分离，以清除一下环境，不然可能会报错
```
docker run --privileged --name your_docker_name -v $PWD:/workspace -it sophgo/tpuc_dev:latest bash
docker exec -it your_docker_name bash
```

### python demo

对于python demo，一定要在LLM-TPU里面source envsetup.sh（与tpu-mlir里面的envsetup.sh有区别）
```shell
cd /workspace/LLM-TPU
source envsetup.sh
```

```
cd /workspace/LLM-TPU/models/Qwen_VL_Chat/python_demo
mkdir build && cd build
cmake .. && make
cp *cpython* ..
cd ..


python3 pipeline.py --model_path qwen-vl-chat_int4_1dev.bmodel --tokenizer_path ../support/token_config/ --devid 0 --generation_mode greedy
```