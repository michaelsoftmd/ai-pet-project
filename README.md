# AI Pet Project

A fully contained, daemonless, secure method for storing and using LLMs locally on a mounted SSD using Podman. Includes a SIMPLE llama.cpp method for installing a Vulkan backend with llama.cpp for extensive AMD GPU support. 
Works on 6000 series with RDNA2! 4gb VRAM! 
You can run a 24b model at Q6 Quant without overheating!

Also includes: Basic guides for Ollama, AnythingLLM

A cursory guide for installing and using Llamafile

And something fun to read!


All with Podman! Everything runs as USER in an easy-to-use container.

## Overview

Congratulations on choosing to build your own secure AI Pet! We will be using Podman, Ollama, and Open WebUI to put this all together, but never fear, the process is quite simple. If you have an integrated GPU, please don't bother with anything else and use the LLAMA CPP guide above.

UPDATE: I am now incorporating Zendriver support for web automation purposes. So far it works, with some awkward kinks that I am straightening out. Zendriver works by providing a web browser and a set of instructions on how to use that browser to your local LLM. That browser is fully contained inside Podman, can be used by the LLM for multiple purposes, and works as the user sends instructions through Open WebUi. You can visually watch the LLM use that browser, safely and securely, from outside the container by use of a VNC program (also included).

ANOTHER UPDATE: I have built a docker-compose file that will easily run all services at the user's discretion. It will be uploaded when the project is completed.


## Specific Features (some incoming)

- Open WebUI for a versatile chat interface.
- OpenHands functionality for an agentic development environment.
- Zendriver support for web automation of research and social media tasks.
  - Zendriver features: Navigation, clicking/typing, search, infinite scroll, data extract, page interaction. Currently also CACHES PAGE DATA to preserve context length. 

## Advantages

Here are some advantages over other methods:

- **External Storage**: Everything is stored externally. With simple changes this whole setup is transferable between computers.

- **One Central Model Directory** For ease of use, all models are kept at /mnt/ssd/podman/models/gguf/[your model here]

- **Localhost Security**: Everything runs from localhost, meaning a far lower chance of outside interference or monitoring.

- **Rootless Containers**: The AI itself runs rootless in its own container, meaning even if it 'broke out' it would have no permission to interfere with your system (no 'sudo').

- **CPU or GPU**: You can choose between running your LLMs on CPU (with Ollama) or with AMD GPU acceleration (with llama.cpp and Vulkan). There is currently excellent support for AMD iGPUs, and this guide helps you find the best way to optimise your hardware.

- **Secure Capabilities**: The AI is still capable of agentic behaviour, but doesn't have permission to mess with system files. CURRENTLY DEVELOPING ZENDRIVER SUPPORT.


## Requirements

There are some requirements, but these are simple:

- **External SSD**: A mounted external SSD with at least 500GB capacity. It **MUST** be formatted to ext4. The guide includes instructions on mounting.

- **Podman Compatibility**: Podman apparently conflicts with Docker. If you already have Docker, though, you should know this and could do this yourself.

- **Linux Distribution**: Linux Mint (this guide also works for Ubuntu and other similar distributions).

## Getting Started

I have made this guide as simple and straightforward as possible. I am not a coder by any means. I am a writer, so I have put it together from a beginner's mindset. This process should all be possible by someone who has decided to switch to Linux Mint after using Windows. Any errors should be covered in troubleshooting. I will add that the official documentation for the programs we will be using is quite complete, also, so you can refer to that if you get stuck. If you have issues loading models, try creating a folder in the Podman working directory as /mnt/ssd/podman/models/gguf/[your models here]

To get started, download PODMAN AI PET README near the top of this page.

This guide now has an addendum, an expansion pack or DLC, that uses llama.cpp. It supports integrated GPUs, meaning the AI works better. Everything in that readme still works under your podman container setup. Have fun!

---

For more information on my writing projects please visit [www.akickintheteeth.com](http://www.akickintheteeth.com)
