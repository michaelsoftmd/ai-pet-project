# AI Pet Project

A fully contained, daemonless, secure method for storing and using LLM locally on a mounted SSD. Uses Podman, Ollama, and AnythingLLM.

## Overview

Congratulations on choosing to build your own secure AI Pet! We will be using Podman, Ollama, and AnythingLLM to put this all together, but never fear, the process is quite simple. 

## Advantages

Here are some advantages over other methods:

- **External Storage**: Everything is stored externally. With simple changes this whole setup is transferable between computers.

- **Localhost Security**: Everything runs from localhost, meaning a far lower chance of outside interference or monitoring.

- **Rootless Containers**: The AI itself runs rootless in its own container, meaning even if it 'broke out' it would have no permission to interfere with your system (no 'sudo').

- **CPU-Based**: Everything runs via CPU, meaning the only limit is your computer's RAM. I might add GPU support later.

- **Full Capabilities**: The AI is still fully capable of agentic behaviour, including web browsing (if you let it).

## Requirements

There are some requirements, but these are simple:

- **External SSD**: A mounted external SSD with at least 500GB capacity. It **MUST** be formatted to ext4. The guide includes instructions on mounting.

- **Podman Compatibility**: Podman apparently conflicts with Docker. If you already have Docker, though, you should know this and could do this yourself.

- **Linux Distribution**: Linux Mint (this guide also works for Ubuntu and other similar distributions).

## Getting Started

I have made this guide as simple and straightforward as possible. I am not a coder by any means. I am a writer, so I have put it together from a beginner's mindset. This process should all be possible by someone who has decided to switch to Linux Mint after using Windows. 

Any errors should be covered in troubleshooting. I will add that the documentation for the programs we will be using is quite complete, also, so you can refer to that if you get stuck.

---

For more information on my writing projects please visit [www.akickintheteeth.com](http://www.akickintheteeth.com)
