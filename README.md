# Contents:
- [Setup](#setup)
  - [Performance Tweaks](#performance-tweaks)
    - [Update Grub Cmdline](#update-grub-cmdline)
    - [Set CPU Scaling Governor](#set-cpu-scaling-governor)
    - [Set Platform Profile](#set-platform-profile)
- [Benchmarks](#benchmarks)
  - [Whisper Timestamped](#whisper-timestamped)
    - [The Benchmark Code](#the-benchmark-code)
    - [Whisper Benchmark Results](#whisper-benchmark-results)
  - [Large Language Models](#large-language-models)

# Setup
My specific machine is the [Framework Desktop](https://frame.work/desktop) 128GB.

I've chosen to install [Arch Linux](https://archlinux.org/) in a headless configuration with no desktop environment. The [Arch wiki](https://wiki.archlinux.org/title/Framework_Desktop) is a good resource for setup steps specific to the Framework Desktop.

## Performance Tweaks

### Update Grub Cmdline:
```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 amd_iommu=off ttm.pages_limit=29360128 ttm.page_pool_size=29360128"
```
The `amd_iommu=off` is an optimization that is supposed to increase performance by a small amount, though I haven't personally seen a difference.\
The `pages_limit` and `page_pool_size` set the ttm unified memory to advertise 112GB of available memory for allocation by the GPU.

### Set CPU Scaling Governor
Set the CPU [scaling governor](https://wiki.archlinux.org/title/CPU_frequency_scaling#Scaling_governors) to performance.

There are several ways to do this, but a minimal configuration is to make a one-shot service that runs the command `cpupower frequency-set -g performance` on startup: Create the file `/etc/systemd/system/set-cpupower.service `:
```
[Unit]
Description=Set cpu governor to performance
After=sysinit.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'cpupower frequency-set -g performance'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl daemon-reload
sudo systemctl enable --now set-cpupower.service
```

Verify that the governors are set by reading the `scaling_governor` file:
```
sudo cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

Alternatively you can install the [cpupower](https://archlinux.org/packages/?name=cpupower) package to help manage it.

### Set Platform Profile
Set the `platform_profile` to performance.

Like cpupower, I created a minimal startup script in `/etc/systemd/system/set-platform-profile.service`:
```
[Unit]
Description=Set ACPI platform_profile to performance
After=sysinit.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo performance > /sys/firmware/acpi/platform_profile'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl daemon-reload
sudo systemctl enable --now set-platform-profile.service
```

Verify that it's set correctly:
```
cat /sys/firmware/acpi/platform_profile
```

Verify the [amd-pstate](https://wiki.archlinux.org/title/CPU_frequency_scaling#amd_pstate):
```
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_driver
```
It should report `amd-pstate-epp`

Alternatively you can install the [power-profiles-daemon](https://archlinux.org/packages/extra/x86_64/power-profiles-daemon/)

Once these are enabled, you should see max power usage exceed 100W.

# Benchmarks

## Whisper Timestamped
[whisper-timestamped](https://github.com/linto-ai/whisper-timestamped) is a version of [OpenAI Whisper](https://github.com/openai/whisper) that has word-level timestamps. It's a good benchmark because whisper models are fairly small but computationally intensive and allow for apples-to-apples comparisons across multiple systems.

### The Benchmark Code

I've written a [custom benchmark](https://github.com/chameleon-ai/ryzen-ai-max/tree/main/assets/transcribe-benchmark) that you can run yourself.

Steps to run:
- Download the [script](/assets/transcribe-benchmark/transcribe.py) as well as the .opus files in the directory (or use your own audio files if you like!)
- make virtual environment (python 3.13):\
`python -m venv venv`
- Install torch:\
`pip install --pre torch torchvision torchaudio --index-url https://rocm.nightlies.amd.com/v2/gfx1151/`\
or\
`pip3 install --pre torch torchvision --index-url https://download.pytorch.org/whl/nightly/rocm7.1`
- Install requirements:\
`pip install -r requirements.txt`
- Run inference:\
`python transcribe.py 1-min.opus 5-min.opus 15-mins.opus 30-mins.opus`
- It'll transcribe each audio 5 times and output something like this:
```
Transcribing file "1-min.opus"
...
Elapsed times:
[69.52972419600701, 68.96859192500415, 69.21907865999674, 69.04367544899287, 69.29389223200269]
```

### Whisper Benchmark Results
Below are comparisons between the [6750XT](https://www.amd.com/en/products/graphics/desktops/radeon/6000-series/amd-radeon-rx-6750-xt.html) (12GB), the [6800XT](https://www.amd.com/en/products/graphics/desktops/radeon/6000-series/amd-radeon-rx-6800-xt.html) (16GB), the [6950XT](https://www.amd.com/en/products/graphics/desktops/radeon/6000-series/amd-radeon-rx-6950-xt.html) (16GB), and the AI Max's [8060S](https://www.amd.com/en/products/processors/laptop/ryzen/ai-300-series/amd-ryzen-ai-max-plus-395.html).

Note: A value of 1 denotes that the model takes 1 second to process 1 second of audio, so values smaller than 1 mean that whisper is processing audio faster than real time.

![Performance of Whisper-Timestamped](/assets/whisper.png)

We can see that all of the desktop GPUs handily outperform the 8060S, with the modest 6750XT being roughly twice as fast.

Note that Pytorch + ROCm version matters. Below are benchmarks all on the 8060S from [gfx1151](https://rocm.nightlies.amd.com/v2/gfx1151/) nightly (Nov 28 2025) and [rocm 7.1](https://download.pytorch.org/whl/nightly/rocm7.1) nightly (Dec 2 2025). As of the time of writing, the latest stable pytorch + ROCm release is still 6.4, so it's possible that 7.1 will continue to gain performance improvements in the future.

![Performance based on rocm install](/assets/whisper2.png)

The initial tests using the gfx1151 nightly vary wildly, and in the case of the 1 minute test, transcription takes longer than real-time, which is abysmal performance. Once changing to 7.1 the results are on average 25% faster and much more consistent.

## Large Language Models

Below is the performance of [IceAbsintheNeatRP-7b](https://huggingface.co/mradermacher/IceAbsintheNeatRP-7b-GGUF), a 7B model that uses up to 9GB of VRAM.

![Performance of IceAbsinthe7b](/assets/iceabsinthe.png)

The 8060S runs from 27-13 tokens/s depending on context length, with the 6800XT coming in at 52-27, just about twice as fast.

However, when running a [24B model](https://huggingface.co/mradermacher/BereavedCompound-v1.0-24b-GGUF) that the 6800XT's 16GB VRAM can't handle, the performance is flipped:

![Performance of BereavedCompound24b](/assets/bereavedcompound.png)

This shows us that the moment VRAM becomes a bottleneck, the power of the desktop GPU disappears. Here, the 8060S runs 8.7-6 tokens/s and the 6800XT runs 3.3-2.98. Note that faster performance on the 6800XT is possible, but I offloaded enough layers to be able to fit 8192 tokens of context in VRAM.

In fact, we can see this drastic difference with [GPT-OSS-20b](https://huggingface.co/bartowski/openai_gpt-oss-20b-GGUF-MXFP4-Experimental)

![Performance of GPT OSS 20b](/assets/oss20b.png)

When loading all layers to the GPU, the 6800XT can only process up to 4096 tokens of context before running out of memory. With all layers on the GPU, the 6800XT could run 116-95 tokens/s, but once it had to offload a few layers to CPU, performance dipped drastically to 34-29 tokens/s. The 8060S splits the middle with 70-48 tokens/s.

To illustrate this even further, we can see that the performance of the much more massive model [OSS 120b](https://huggingface.co/bartowski/openai_gpt-oss-120b-GGUF-MXFP4-Experimental) (63GB) on the 8060S runs faster than OSS 20b on the 6800XT.

![GPT OSS 20b vs OSS 120b](/assets/oss20vs120.png)

And for completeness, here's the benchmark for OSS 120b on both systems:

![Performance of GPT OSS 120b](/assets/oss120b.png)

The 6800XT runs 12.9-11.6 tokens/s while the 8060S runs 3-4x faster at 48-31.3 tokens/s. Note that I couldn't even run above 6144 tokens of context on the 6800XT, I simply ran out of memory on the whole system.