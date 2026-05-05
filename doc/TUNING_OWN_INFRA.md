# Own Infrastructure Tuning

This repository keeps the MoneroOcean features enabled. The goal here is not a
portable public miner profile, but fast CPU mining on:

* Intel Xeon Scalable 2nd Generation servers (Cascade Lake)
* macOS arm64 machines, especially Apple Silicon

Measure every change with the same pool, algorithm mix, power limits and ambient
temperature. RandomX hashrate changes of 1-3% can disappear in noise if the host
is not pinned to a stable power and frequency policy.

## Build Targets

Use the GitHub release artifact that matches the host:

* `xmrig-linux-x64-cascadelake.tar.gz` for Xeon Scalable 2nd Gen Linux servers.
  It is built with `-march=cascadelake -mtune=cascadelake` and should not be
  treated as a portable x86_64 binary.
* `xmrig-linux-x64.tar.gz` for generic Linux x86_64 fallback.
* `xmrig-macos-arm64.tar.gz` for Apple Silicon.

For a local Xeon build:

```bash
cmake -S . -B build-cascadelake \
  -DCMAKE_BUILD_TYPE=Release \
  -DWITH_MO_BENCHMARK=ON \
  -DWITH_OPENCL=OFF \
  -DWITH_CUDA=OFF \
  -DCMAKE_C_FLAGS_RELEASE="-Ofast -march=cascadelake -mtune=cascadelake" \
  -DCMAKE_CXX_FLAGS_RELEASE="-Ofast -s -march=cascadelake -mtune=cascadelake"
cmake --build build-cascadelake --parallel
```

For a local Apple Silicon build:

```bash
brew install cmake libuv openssl@3 hwloc
cmake -S . -B build-macos-arm64 \
  -DCMAKE_BUILD_TYPE=Release \
  -DOPENSSL_ROOT_DIR="$(brew --prefix openssl@3)" \
  -DWITH_MO_BENCHMARK=ON \
  -DWITH_OPENCL=OFF \
  -DWITH_CUDA=OFF
cmake --build build-macos-arm64 --parallel
```

The release workflow disables OpenCL and CUDA because this setup is CPU-focused.
That removes plugin/backend surface from the binary and avoids GPU build
dependencies in CI. Re-enable them only if you actually mine with GPUs.

## Xeon Linux Host Setup

RandomX wants memory locality, huge pages and stable clocks more than exotic
compiler flags.

Recommended kernel boot parameters:

```text
msr.allow_writes=on default_hugepagesz=1G hugepagesz=1G hugepages=3
```

For dual-socket machines, reserve at least one 1 GB page per NUMA node used for
mining. For more than two sockets or several miner instances, raise `hugepages`.

Runtime checks:

```bash
lscpu | egrep 'Model name|Socket|NUMA|Thread|Core'
grep -i huge /proc/meminfo
cat /sys/devices/system/node/node*/hugepages/hugepages-1048576kB/free_hugepages
```

CPU governor and frequency policy:

```bash
sudo apt-get install -y linux-tools-common linux-tools-$(uname -r) msr-tools
sudo cpupower frequency-set -g performance
sudo modprobe msr
```

On dedicated mining hosts, disable deep power saving and keep package power
limits consistent in BIOS. Prefer stable all-core clocks over short boost
peaks. If Hyper-Threading reduces hashrate on a specific algorithm mix, test
with physical cores only before disabling it globally.

## Xeon Config Baseline

Start with this RandomX/CPU section:

```json
{
    "randomx": {
        "init": -1,
        "init-avx2": 1,
        "mode": "fast",
        "1gb-pages": true,
        "rdmsr": true,
        "wrmsr": true,
        "cache_qos": false,
        "numa": true,
        "scratchpad_prefetch_mode": 1
    },
    "cpu": {
        "enabled": true,
        "huge-pages": true,
        "huge-pages-jit": false,
        "hw-aes": true,
        "priority": 3,
        "memory-pool": true,
        "yield": false,
        "max-threads-hint": 100,
        "asm": "intel"
    }
}
```

Tuning notes:

* Keep `numa=true` with `WITH_HWLOC=ON` on multi-socket servers.
* Keep `wrmsr=true`; XMRig applies the Intel RandomX MSR preset when permitted.
* Try `scratchpad_prefetch_mode` values `1` and `2` on each CPU model. Mode `1`
  is the default baseline; mode `2` can win on some Intel systems.
* Use `yield=false` only on dedicated miners. Keep it `true` on shared hosts.
* If the machine becomes unresponsive, lower `priority` before reducing threads.

For manual NUMA separation, run one miner per socket:

```bash
numactl --cpunodebind=0 --membind=0 ./xmrig --config config-node0.json
numactl --cpunodebind=1 --membind=1 ./xmrig --config config-node1.json
```

This can beat one global process on hosts where automatic placement is uneven.
Benchmark both.

## macOS ARM64 Baseline

macOS does not provide Linux-style 1 GB huge pages or MSR tuning. Keep the config
simple and focus on thermal stability.

```json
{
    "randomx": {
        "init": -1,
        "init-avx2": 0,
        "mode": "fast",
        "1gb-pages": false,
        "rdmsr": false,
        "wrmsr": false,
        "cache_qos": false,
        "numa": false,
        "scratchpad_prefetch_mode": 1
    },
    "cpu": {
        "enabled": true,
        "huge-pages": true,
        "huge-pages-jit": false,
        "hw-aes": null,
        "priority": null,
        "memory-pool": true,
        "yield": false,
        "max-threads-hint": 100,
        "asm": true
    }
}
```

Apple Silicon tuning notes:

* Test `max-threads-hint` between `70` and `100`. On laptops or compact Macs,
  fewer threads can maintain higher sustained clocks.
* Keep the machine plugged in and prevent sleep.
* Watch temperature and throttling over at least 20 minutes, not only the first
  benchmark window.
* A GitHub-hosted macOS ARM runner validates the build, but it is not a reliable
  proxy for M2 Max hashrate. Use a self-hosted runner on your M2 Max if you want
  release artifacts built on that exact CPU class.

## MoneroOcean

Keep `WITH_MO_BENCHMARK=ON`. The `algo-perf` calibration is useful for
MoneroOcean's algo switching and should be generated on each hardware class, not
copied between Xeon and Apple Silicon hosts.

Run a calibration after changing build flags, thread counts, huge pages or power
limits:

```bash
./xmrig --config config.json --rebench-algo
```

After calibration, preserve the generated `algo-perf` values per host class or
per exact machine type.
