# The Framework Desktop w/ Ryzen AI Max+ 395

Contents:
- [Benchmarks](#benchmarks)
  - [Whisper Timestamped](#whisper-timestamped)
    - [The Benchmark Code](#the-benchmark-code)

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


![Performance of Whisper-Timestamped](/assets/whisper.png)