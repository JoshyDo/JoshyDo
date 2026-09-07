# HFT-Engine

An ultra-low-latency, zero-allocation High-Frequency Trading (HFT) arbitrage engine built with C++20, AF_XDP, and eBPF. 

This engine is designed to break the microsecond barrier by completely bypassing the standard Linux networking stack for both ingress and egress, utilizing data-oriented design to keep the hot path strictly within the L1 cache.

## Core Architecture

* **Data-Oriented Limit Order Book (LOB):** The LOB is implemented as a flat array to ensure it resides entirely within the L1 cache. The hot path enforces a strict zero-allocation policy.
* **Zero-Copy Ingress (AF_XDP & eBPF):** A custom eBPF/XDP pipeline routes Ethernet frames directly into userspace UMEM rings, completely bypassing the OS network stack.
* **Raw TCP Egress (Userspace Stack):** Total bypass of the OS TCP stack for order execution. Raw TCP/IP packets are constructed directly in the UMEM TX ring buffer. This includes manual, cycle-optimized 16-bit one's complement checksum calculations (TCP pseudo-header & IPv4), accounting for endianness padding.
* **Asynchronous Memory Management:** A custom run-to-completion loop ensuring precise asynchronous reaping of RX, TX, FILL, and COMPLETION ring buffers. This prevents UMEM starvation and avoids deadlocks during simultaneous ingress/egress operations on the same interface.

## Performance Benchmarks

*Hardware/Environment: AMD Ryzen 7 5800 / NVIDIA RTX 5070 Ti*

| Metric | Latency / CPU Cycles |
| :--- | :--- |
| **SBE Validation & LOB Update** | `~9.5 ns` (38 cycles) |
| **Wire-to-Book (AF_XDP Ingress)** | `~864 ns` (3,496 cycles) |
| **Standard OS T2T Limit (io_uring)* **| `2.33 µs` (Median) / `2.43 µs` (p99) |

*\* Note on OS architectural limits: The standard Linux stack was extensively benchmarked using AF_INET, io_uring with SQPOLL, IRQ-Pinning, TCP_QUICKACK, and busy polling. Hitting a hard limit of ~2.33 µs proved that sub-microsecond Tick-to-Trade (T2T) latencies strictly require kernel bypass.*

## Dependencies

* C++20 compatible compiler (GCC 11+ / Clang 14+)
* eBPF / AF_XDP toolchain (`libbpf`, `clang`, `llvm`, `iproute2`)
* CMake (3.20+)

## Build Instructions

```bash
git clone https://github.com/JoshyDo/HFT-Engine.git
cd HFT-Engine
mkdir build && cd build

# Build for release with maximum optimizations
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
```

## Current Focus & Roadmap

* **UMEM Stabilization:** Perfecting the asynchronous reaping of FILL and COMPLETION rings to guarantee starvation-free operations on shared RX/TX interfaces.
* **Hardware Acceleration:** Future integration of CUDA micro-kernels for parallelized quantitative pricing model computations.

## License
MIT License
