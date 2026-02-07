# Hybrid KV Cache 学习总结

## 核心概念
- **Hybrid KV cache manager** 通过 `kv_cache_groups` 把不同注意力类型的层分成多个组，每个组拥有独立的块表，但这些块仍来自统一的 block pool，从而让不同注意力类型可以在同一模型中共存并进行前缀缓存管理。【F:vllm/v1/kv_cache_interface.py†L468-L506】【F:vllm/v1/core/kv_cache_coordinator.py†L338-L428】
- **HybridKVCacheCoordinator** 在多种注意力类型下，通过迭代固定点算法寻找“所有注意力组都命中”的最长缓存前缀。它把不同 KV cache spec 的组先分组，再按 full attention 优先的顺序求交，使最终命中长度满足所有组的 block_size 约束。【F:vllm/v1/core/kv_cache_coordinator.py†L430-L571】
- **block hash 与 group id**：Hybrid 模式下，KV block hash 需要带上 group id，避免不同组的同一 token block 产生冲突。`make_block_hash_with_group_id` 负责将 `BlockHash` 与 group id 拼接成统一 key。【F:vllm/v1/core/kv_cache_utils.py†L40-L67】

## KV cache 组与 block 管理
- KV cache groups 用来描述不同注意力类型的层集合，并为每组生成独立块表；这些 group 信息通过 `KVCacheConfig.kv_cache_groups` 提供给调度与缓存管理层。【F:vllm/v1/kv_cache_interface.py†L468-L506】
- Hybrid 模式在寻找缓存命中时，会根据每组的 `KVCacheSpec` 计算可用的最长前缀，结果是所有组都能接受的公共前缀长度；这样可以保证每个注意力类型的缓存块一致可用。【F:vllm/v1/core/kv_cache_coordinator.py†L430-L571】

## 与 prefix caching 的关系
- 请求级别的 `block_hashes` 仍由 token 序列生成，但在 hybrid 模式下需要在使用时再补上 group id，使得同一 token block 在不同注意力组中被当作不同的缓存实体管理。【F:vllm/v1/core/kv_cache_utils.py†L40-L67】【F:vllm/v1/request.py†L162-L210】
- Hybrid cache manager 通过 `find_longest_cache_hit` 统一处理各组命中结果，最终得到跨组一致的命中长度，用于后续的 KV cache 分配与缓存行为。【F:vllm/v1/core/kv_cache_coordinator.py†L430-L571】

## 关键限制与假设
- Hybrid 目前要求每组 `block_size` 与 `hash_block_size` 有整倍数关系；这保证了不同注意力类型之间能够对齐缓存块边界。【F:vllm/v1/core/kv_cache_coordinator.py†L387-L409】
- 部分功能（如 prefix cache hit 的通用实现、DCP/PCP 支持）仍有限制或只在简化场景下生效，需要在设计 connector 时显式处理或保持一致性假设。【F:vllm/v1/core/kv_cache_coordinator.py†L401-L527】
