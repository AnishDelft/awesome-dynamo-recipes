# AI Configurator and DynoSim

This README shows a minimal setup for using AI Configurator with DynoSim mocker and replay workflows.

## Installation

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install aiconfigurator
pip install "ai-dynamo[mocker, replay]"
```

## Simulation Requirements

AI Configurator and DynoSim simulation workflows do not require GPUs to run. They are intended for offline planning, configuration search, performance modeling, and request-flow simulation before spending time on real GPU clusters.

## AI Configurator vs DynoSim

AI Configurator and DynoSim are complementary. AI Configurator is best for finding a deployment configuration from model, hardware, backend, and SLA inputs. DynoSim is best for exercising serving behavior under simulated runtime conditions, including mocked engines or replayed traffic.

| Area | AI Configurator | DynoSim Mocker | DynoSim Replay |
| --- | --- | --- | --- |
| Primary purpose | Search and rank deployment configurations. | Simulate a Dynamo worker/engine without loading the real model on GPUs. | Re-run captured or prepared traffic patterns through the simulator. |
| Main question answered | "What configuration should I start with?" | "How would this modeled engine behave in the serving stack?" | "How would this workload behave if replayed against a simulated setup?" |
| Inputs | Model path, GPU count, system type, backend, sequence lengths, SLA targets. | Model path, engine type, AIC performance-model parameters, block/cache settings, speedup ratio. | Replay trace or request dataset, plus simulator/runtime configuration. |
| Outputs | Ranked candidate configurations and optional deployment files. | Simulated runtime service behavior for development and performance exploration. | Simulated workload behavior for repeatable experiments. |
| Use when | Planning a new deployment or comparing hardware/backend choices. | Testing integration and approximate serving behavior without GPUs. | Studying repeatable workload behavior from known traffic. |
| GPU requirement | No GPU required for the configuration search itself. | No GPU required. | No GPU required. |

## AI Configurator

[AI Configurator](https://github.com/ai-dynamo/aiconfigurator) helps find strong starting configurations for disaggregated LLM serving. Given a model, GPU count, GPU type, backend, and SLA targets such as TTFT and TPOT, it searches deployment options and can generate configuration files for Dynamo or llm-d.

### Support Matrix Example

```bash
aiconfigurator cli support \
  --model-path deepseek-ai/DeepSeek-V3 \
  --system all --backend all
```

```text
15:37:38 [aiconfigurator] [utils.py:926] [I] Quant inference result: quant_algo=fp8_block, kv_cache_quant_algo=None, quant_dynamic=True
15:37:38 [aiconfigurator] [utils.py:1033] [I] Loaded model config for deepseek-ai/DeepSeek-V3: architecture=DeepseekV3ForCausalLM, layers=61, n=128, n_kv=128, d=56, hidden_size=7168, inter_size=18432, vocab=129280, context=163840, topk=8, num_experts=256, moe_inter_size=2048, extra_params={'kv_lora_rank': 512, 'qk_rope_head_dim': 64}

======================================================================
  AIC Support Matrix
======================================================================
  Model:  deepseek-ai/DeepSeek-V3
  Arch:   DeepseekV3ForCausalLM
----------------------------------------------------------------------
                           trtllm          sglang           vllm
  System                  agg disagg      agg disagg      agg disagg
  ----------------------------------   -------------   -------------
  a100_pcie                 -      -        -      -        -      -
  a100_sxm                 NO     NO       NO     NO       NO     NO
  a30                       -      -        -      -        -      -
  b200_sxm                 NO     NO      YES    YES      YES    YES
  b300_sxm                 NO     NO      YES    YES       NO     NO
  b60                       -      -        -      -       NO     NO
  gb200                    NO     NO      YES    YES       NO     NO
  gb300                    NO     NO      YES    YES       NO     NO
  h100_pcie                 -      -        -      -        -      -
  h100_sxm                 NO     NO       NO     NO       NO     NO
  h200_sxm                YES    YES      YES    YES       NO     NO
  l4                        -      -        -      -        -      -
  l40s                     NO     NO       NO     NO       NO     NO
  rtx_pro_6000_server       -      -       NO     NO        -      -
======================================================================
  YES = supported  NO = not supported  - = no data
  Using latest available version per system/backend combination.
======================================================================
```

### Default Configuration Search Example

```bash
aiconfigurator cli default \
  --model-path deepseek-ai/DeepSeek-V3 \
  --total-gpus 16 \
  --system b200_sxm \
  --backend sglang \
  --isl 20000 --osl 22 --prefix 1000 \
  --ttft 2500 --tpot 30 \
  --request-latency 3000 \
  --database-mode SILICON \
  --max-seq-len 21000 \
  --top-n 5 \
  --save-dir results/deepseek_case1_b200
```

## DynoSim Mocker

DynoSim Mocker runs a simulated Dynamo engine process. Instead of launching a real inference backend on GPUs, it uses modeled performance parameters, such as the AI Configurator performance model settings below, to approximate engine timing and capacity. This makes it useful for local development, control-plane testing, and early performance exploration.

DynoSim Mocker requires NATS and etcd before running the mocker process.

### Start etcd

```bash
docker run -d --name etcd \
  -p 2379:2379 -p 2380:2380 \
  quay.io/coreos/etcd:v3.5.9 \
  etcd \
  --advertise-client-urls http://0.0.0.0:2379 \
  --listen-client-urls http://0.0.0.0:2379
```

### Start NATS

```bash
docker run -d --name nats-server -p 4222:4222 nats:latest
```

### Run DynoSim Mocker

```bash
python -m dynamo.mocker \
  --model-path deepseek-ai/DeepSeek-V3 \
  --engine-type vllm \
  --aic-perf-model \
  --aic-system h200_sxm \
  --aic-tp-size 8 \
  --aic-moe-tp-size 8 \
  --aic-moe-ep-size 1 \
  --num-gpu-blocks-override 8192 \
  --block-size 64 \
  --max-num-seqs 256 \
  --speedup-ratio 10.0
```

## DynoSim Replay

DynoSim Replay is intended for repeatable workload simulation. Use it when you already have a request trace or prepared workload and want to replay that traffic through a simulated Dynamo setup without needing GPUs. Replay is useful after using AI Configurator to choose candidate configurations or after using Mocker to validate the simulated service path.

## TODO

- Add replay workflow.
