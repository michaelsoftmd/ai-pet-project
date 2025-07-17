# ai-pet-project
A fully contained, daemonless, secure method for storing and using LLM locally on a mounted SSD. Uses Podman, Ollama, and AnythingLLM.


Congratulations on choosing to build your own secure AI Pet. We will be using Podman, Ollama, and AnythingLLM to put this all together, but never fear, the process is quite simple. Here are some advantages over other methods:

    Everything is stored externally. With simple changes this whole setup is transferable between computers.

    Everything runs from localhost, meaning a far lower chance of outside interference or monitoring.

    The AI itself runs rootless in its own container, meaning even if it ‘broke out’ it would have no permission to interfere with your system (no ‘sudo’).

    Everything runs via CPU, meaning the only limit is your computer’s RAM. I might add GPU support later.

    The AI is still fully capable of agentic behaviour, including web browsing (if you let it).



There are some requirements, but these are simple:

    A mounted external SSD. At least 500gb is good. It MUST be formatted to ext4. The guide includes instructions on mounting.

    Podman apparently conflicts with Docker. If you already have Docker, though, you should know this and could do this yourself.

    Linux Mint. This guide also works for Ubuntu and other similar distributions.



I have made this guide as simple and straightforward as possible. I am not a coder by any means. I am a writer, so I have put it together from a beginner’s mindset. This process should all be possible by someone who has decided to switch to Linux Mint after using Windows. Any errors should be covered in troubleshooting. I will add that the documentation for the programs we will be using is quite complete, also, so you can refer to that if you get stuck. 




For more information on my writing projects please visit www.akickintheteeth.com
