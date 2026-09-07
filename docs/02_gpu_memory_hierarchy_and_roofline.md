# 模块 2：显存层级、带宽瓶颈与 Roofline 模型

> **本章重点**：GPU 显存物理金字塔、SRAM vs HBM、Bank Conflict、算术强度（Arithmetic Intensity）、Roofline 性能模型数学边界。

---

## 1. 显存物理金字塔与访问延迟

```mermaid
flowchart TD
    REG["寄存器堆 (Register File)<br>容量: ~256 KB/SM | 延迟: ~0 周期 | 带宽: ~数十 TB/s"] --> SHMEM["共享内存 / L1 Cache (SRAM)<br>容量: ~128~228 KB/SM | 延迟: ~20~30 周期 | 带宽: ~10~15 TB/s"]
    SHMEM --> L2["芯片级统一 L2 Cache (SRAM)<br>容量: 50~256 MB | 延迟: ~100~200 周期 | 带宽: ~5~10 TB/s"]
    L2 --> HBM["高带宽显存 HBM3/HBM3e (DRAM)<br>容量: 80~144 GB | 延迟: ~200~400 周期 | 带宽: 2~4.8 TB/s (H100/H200)"]
    HBM --> PCIE["Host 主存 (通过 PCIe Gen5 x16)<br>延迟: 数千周期 | 带宽: 64 GB/s"]
```

---

## 2. 核心机制剖析

### 2.1 算术强度 (Arithmetic Intensity) 与 Roofline 模型
衡量一个算子（Kernel）性能上限的物理准绳是 **Roofline 模型**：
$$\text{可达算力 } P = \min(P_{\text{peak}}, I \times B_{\text{mem}})$$
- $P_{\text{peak}}$：GPU 的理论峰值浮点算力（TFLOPS）。
- $B_{\text{mem}}$：GPU 的峰值显存带宽（TB/s）。
- $I$：算术强度（Arithmetic Intensity），定义为 $\frac{\text{计算浮点操作数 (FLOPs)}}{\text{显存访存量 (Bytes)}}$。

```mermaid
flowchart LR
    subgraph Roofline ["Roofline 性能边界模型"]
        direction TB
        B_ZONE["内存受限区 (Memory-Bound)<br>性能斜线: P = I × Bandwidth<br>即使增加算力也无法提速！<br>例如: LLM Decode 阶段, FlashAttention 之前的 Softmax"]
        KNEE["转折点 (Knee Point)<br>I_crit = Peak_TFLOPS / Peak_Bandwidth"]
        C_ZONE["算力受限区 (Compute-Bound)<br>性能平顶: P = Peak_TFLOPS<br>受限于 Tensor Cores 矩阵乘吞吐<br>例如: LLM Prefill 阶段大 Batch GEMM"]
        B_ZONE --- KNEE
        KNEE --- C_ZONE
    end
```

### 2.2 共享内存 (Shared Memory) 与 Bank Conflict
- Shared Memory 位于芯片上的 SRAM 中，访问延迟极低。
- 物理上划分为 32 个等宽的 Memory Banks（每个 Bank 4 字节）。
- **Bank Conflict**：当同一个 Warp 内的多个线程同时访问同一个 Bank 内不同地址的数据时，硬件请求会被强制串行化，导致 Shared Memory 带宽严重缩水。

### 2.3 显存合并访存 (Coalesced Access)
Warp 内 32 个线程发起的全局内存读写，若地址是连续对齐的，硬件只需发起 1~2 次 32/64/128-byte 的 Cache Line 事务即可完成加载。如果线程访问跨步（Strided）或离散（Random），会触发大量冗余的数据搬运，将有效 HBM 带宽打碎。

---

## 3. K8s 资深工程师视角的心智模型映射

| Kubernetes / 分布式系统概念 | GPU 显存系统概念 | 机制相似性 |
| :--- | :--- | :--- |
| **本地 CPU L1/L2 Cache** | **Shared Memory (SRAM)** | 极小容量、超高带宽，要求开发者或运行时显式规划与分块（Tiling）。 |
| **节点本地 NVMe SSD** | **HBM3e 全局显存** | 空间容量较大，但成为整个计算流水线的主 IO 瓶颈。 |
| **网络跨节点跨机架 RPC** | **PCIe Host-to-Device 搬运** | 跨越总线边界，延迟陡增，应尽量避免或采用异步流式批处理隐藏。 |
| **I/O Wait (iowait) 导致 CPU 空转** | **Memory Stalls 导致 Warp 挂起** | 计算单元处于等待数据返回的饥饿状态。 |

---

## 4. 动手实操与排查命令

```bash
# 查看当前 GPU 的显存容量、已用显存与显存总带宽规格
nvidia-smi --query-gpu=name,memory.total,memory.used,memory.free --format=csv

# 检查当前节点 PCIe 链路协商速度与宽度（确保跑在 Gen5 x16）
sudo lspci -vvv -d 10de: | grep -E "LnkCap|LnkSta"

# 使用 dcgmi 监控显存控制器利用率
dcgmi dmon -e 203,204,210,211
```

---

## 5. 核心思考题
1. 为什么在大模型推理的自回归 Decode 阶段，即使 Batch Size 为 1，显卡利用率（`nvidia-smi` 看到的 GPU%）很高，但实际利用的 Tensor Core 算力却不到 5%？
2. FlashAttention 是如何利用 SRAM（Shared Memory）重排 Attention 计算图，从而打破 HBM 访存带宽瓶颈的？
