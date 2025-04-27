# DeepEP_ibrc_dual-ports_multiQP
DeepEP_ibrc_dual-ports_multiQP aims to implement dual-port and multi-qp solutions in [DeepEP](https://github.com/deepseek-ai/DeepEP) ibrc transport. Though DeepEP now also use IBGDA mode in normal kernels, in our practice, we found that not all NICs can enable ibgda. It is essential to achieve high performance in ibrc transport. 

The main contributions of this work are summarized as follows:

1. **Transparent Dual-Port and Multi-QP Support**: We modify NVSHMEM to enable dual-port and multi-Queue Pair (QP) support within the IBRC transport layer. This enhancement is fully decoupled from upper-layer applications (e.g., DeepEP).
2. **Comparable performance in ibrc**: Our solution achieves performance parity between dual-port and single-port environment. Simultaneously, our performance in RoCE is not inferior to IB or the current used ibgda stratgey.
3. **NCCL Version Sensitivity in DeepEP**: We are the [first](https://github.com/deepseek-ai/DeepEP/issues/82) to realize that different nccl versions may have an impact on DeepEP performance, on which we find out the potential causes and give a solution in higher version of nccl.

## Performance
### normal kernels test
We evaluate normal kernels on the H100 GPU under RoCE in both single-port and dual-ports environments, with a primary focus on inter-node communication performance.
| Type | Dispatch #EP	| Bottleneck bandwidth | Combine #EP | Bottleneck bandwidth |
|:-------:|:--------:|:-------:|:-------:|:--------:|
| Internode | 16 | 60GB/s(RDMA) | 16 | 61GB/s(RDMA) |
| Internode | 32 | 59GB/s(RDMA) | 32 | 57GB/s(RDMA) |
| Internode | 64 | 52GB/s(RDMA) | 64 | 50GB/s(RDMA) |
### test with different nccl version
In our pratice, we find out that nccl in version 2.21 and earlier can achieve higher performance than later versions. We show the performance of DeepEP using nccl_2.21.5 and nccl_2.22.3 to illustate this phenomenon.
| version | Dispatch #EP	| Bottleneck bandwidth | Combine #EP | Bottleneck bandwidth |
|:-------:|:--------:|:-------:|:-------:|:--------:|
| nccl2.21.5 | 32 | 59GB/s(RDMA) | 32 | 57GB/s(RDMA) |
| nccl2.21.5 | 64 | 52GB/s(RDMA) | 64 | 50GB/s(RDMA) |
| nccl2.22.3 | 32 | 37GB/s(RDMA) | 32 | 36GB/s(RDMA) |
| nccl2.22.3 | 64 | 40GB/s(RDMA) | 64 | 34GB/s(RDMA) |

After in-depth research, we speculate that the lazy connection mechanism introduced in nccl2.22 version caused the substantial drop of DeepEP performance. We conclude the evironment varaiables that can help us achieve high performance in later nccl version.

Single-port environment

```bash
-x NCCL_NVLS_ENABLE=0 \
-x NCCL_RUNTIME_CONNECT=0 \
-x NCCL_IB_QPS_PER_CONNECTION=4 \
-x NCCL_MAX_NCHANNELS=4 \
```

Dual-ports environment

```bash
-x NCCL_RUNTIME_CONNECT=0 \
```

Using the approach described above, we are also able to achieve high performance with NCCL version 2.22 and later.

## Quick start
The execution process is similar to DeepEP, we only make the following changes. We now test DeepEP with commit id a84a24808fb0ea732f49b874cc456a69dde69076. We will fix the conflicts with later version soon.

1. Replace the internode.cu under /DeepEP/csrc/kernels/ with ours.
2. Apply our patch on the original nvshmem_3.2.5-1(merged with the patch of DeepEP for convience and will continually support new version)
3. Add environment varaibles NVSHMEM_IB_MAX_TRANSPORT_EP_COUNT

```bash
# replace internode.cu to use ibrc mode
cp internode.cu /path/to/installed/DeepEP/csrc/kernels/

# use our modified nvshmem
cd /path/to/installed/nvshmem_src
git apply /path/to/installed/deepEP_ibrc_dual-ports_multiQP/nvshmem_ibrc.patch

# set qp num in nvshmem
-x NVSHMEM_IB_MAX_TRANSPORT_EP_COUNT=
```

You can now use test_internode.py to test your performance of ibrc transport.
