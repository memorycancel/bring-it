---
title: Using ggerganov / whisper.cpp
layout: home
---

# Using github.com/ggerganov/whisper.cpp

2025-03-07 23：00

[github.com/ggerganov/whisper.cpp](github.com/ggerganov/whisper.cpp)

I have to say this is a great work!

Set the CUDA ENV before make the project:

```shell
export CUDA_HOME=/usr/local/cuda-12.8
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda-12.8/lib64:/usr/local/cuda/extras/CUPTI/lib64
export PATH=$PATH:$CUDA_HOME/bin

```

Use the desktop APP -- `whisper.desktop`:

```ini
[Desktop Entry]
Type=Application
Name=whisper
Exec=/home/memorycancel/git/whisper.cpp/build/bin/whisper-stream -t 8 -fa -kc -m /home/memorycancel/git/whisper.cpp/models/ggml-base.en.bin
Terminal=true
```