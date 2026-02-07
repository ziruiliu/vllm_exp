# Offloading Connector 支持 HMA 的计划

## 目标
让 CPU memory offloading connector 能够在 Hybrid KV cache manager 下正确保存/恢复各类注意力缓存（不同 kv_cache_group），并显式声明支持 HMA。整体原则是：**为每个 kv_cache_group 维护独立的 block hash 与传输路径，但保持统一调度语义**。

## 需要完成的工作
1. **标记 connector 支持 HMA**
   - 让 offloading connector 实现 `SupportsHMA` 接口，并提供 `request_finished_all_groups` 路径，确保 scheduler 在 hybrid 模式下调用正确入口。【F:vllm/distributed/kv_transfer/kv_connector/v1/base.py†L82-L115】【F:vllm/v1/core/sched/scheduler.py†L1913-L1928】

2. **按 kv_cache_group 生成 block hash 与传输计划**
   - 调度侧在计算 prefix cache 命中与 load/store 计划时，使用 `make_block_hash_with_group_id` 把 group id 合并进 block hash，避免不同组冲突。【F:vllm/v1/core/kv_cache_utils.py†L40-L67】
   - 对每个 group 独立构建 transfer spec，并在 metadata 中区分 group id；这样即使 block_ids 在各组不同，也可以逐组传输。

3. **扩展 worker 端的 offloading handler 以识别 group**
   - Offloading handler 需要理解 “同一 transfer spec 只作用于某个 group 的 tensors”。
   - 通过 layer -> group 映射收集每个 group 对应的 tensor 索引，在 transfer 时按 group 选择对应子集。

4. **更新统计与完成状态的聚合逻辑**
   - load/store 可能拆分为多个 group job，worker 端需在所有 group job 完成后再把请求标记为 finished_recving / finished_sending。

5. **回归验证思路**
   - 在 hybrid 模型上进行 prefix cache 命中（例如 full+local 或 full+SW），确认 offloading 能同时保存/恢复各组 KV。
   - 检查 KV events 与 metrics，确保事件里带有带 group 的 block hash，不会发生跨组混淆。
