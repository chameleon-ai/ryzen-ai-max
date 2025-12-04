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
It shouldshould report `amd-pstate-epp`

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