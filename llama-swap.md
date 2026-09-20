# Setting Up llama-swap

This is the second part of the setup guide. It assumes you have completed everything in part one.

```llama-swap``` is the next thing to configure. It is a single Go binary that presents an OpenAI- and Anthropic-compatible endpoint, reads the model field in the request, then starts or swaps the matching ```llama-server``` process. Basically, a reverse proxy for models.

This will let us:

- Run multiple models at once safely
- Swap models safely
- Give all of our tools that tie into ```llama-server``` one place to go safely

This should have been installed in part one. Now we need to configure it. In this start configuration, we're going to set the server to be greedy for resources and see how it responds.

## Initial Setup

Using a text editor of your choice, preferably one that can handle YAML syntax, create the file ~/.config/llama-swap/config.yaml.

First, put in the following:

```
# ~/.config/llama-swap/config.yaml
#
# One endpoint at http://127.0.0.1:8080/v1 for every client.
# Adjust model paths and quant filenames to match what you actually download.

healthCheckTimeout: 300
logLevel: debug
```

Anything with a # in front is a comment, but it's a good idea to put the file name and path at the top for reference.

- **healthCheckTimeout: 300**: Remember the curl request to localhost:8081/health we made last chapter? This marks how long llama-swap will wait for llama-server to answer before declaring a failure. It is in seconds. 5 minutes is generous.
- **logLevel: debug**: Tells llama-swap we want full details in the logs. Once everything is set, or if there is too much information, you can switch it from debug to info.

Save your file.

Below this, we are going to have a macros section, a models section, and groups section. These will hold the information llama-swap needs to tell llama-server to swap models correctly.

## Macros
Remember the large set of flags we needed to load the model last time? Now we're going to define some of them here so the models: section can reference them.

Add this to your file at the end:

```
macros:
  base: >
  /usr/bin/llama-server
    --device Vulkan0
    --host 127.0.0.1
    --port ${PORT}
    --n-gpu-layers 99
    --flash-attn on
    --metrics
    --no-webui
    --parallel 1

  kvq: --cache-type-k q8_0 --cache-type-v q8_0

  gptoss: >
      --jinja
      --temp 1.0
      --top-p 1.0
      --top-k 0
      --min-p 0
      --chat-template-kwargs {"reasoning_effort":"medium"}
```

This defines three macros: base, kvq, and gptoss. They all contain flags that ```llama-server``` can take. We're going to pass these macros to our model calls below. Some of this will be familiar. Here are the new flags:

- **--metrics**: This exposes an endpoint that llama-swap's UI will read to get token counts.
- **--no-webui**: This turns off the webUI for llama-server we played with in the last section. We'll be using something else.
- **--parallel 1**: Let's also set the number of KV slots to 1, which will give our tools maximum access to the KV cache.
- **--cache-type-k** and **--cache-type-v**: Sets the storage format of the KV cache. We're setting it this way to save some VRAM space, but leaving it as a macro so we can go back to the default for testing.
- **--jinja**: Tells the server to use the Jinja template stored in the gpt-oss model we're using. This lets us access fancier chat stuff for tool calls, chain of thought, etc.
- **--temp** **--top-p**, **--top-k**, and **--min-p**: These are resetting ```llama-server```'s defaults to something our gpt-oss model expects, which is no preset. These all control deep probability magic that models use. Note that any client sending its own version of these to llama-swap will override these. This is by design.
- **--chat-template-kwargs {"reasoning_effort":"medium"}**: This tells the Jinja template to set the keyword argument (kwarg) "reasoning_effort" to "medium". This sets how much our model will think about an answer before giving a response. We're setting a default here. Our tools may change it.

## Models

We only have the one model right now. Here is the entry for it. Remember to change --model to match the full path of your model location. Place it at the bottom of the file.

```
models:
  "gpt-oss-20b":
    cmd: |
      ${base}
      --model /home/$USER/models/gpt-oss-20b-Q4_K_M.gguf
      ${kvq}
      ${gptoss}
      --ctx-size 65536
      --batch-size 2048
      --ubatch-size 2048
      --n-cpu-moe 0
      --cache-reuse 256
    ttl: 1800
    aliases:
      - "agent"      
```

In models, we define our model name, "gpt-oss-20b":, then set the command (cmd). This is what gets sent when we tell llama-swap to activate this model. ${base} puts in everything in the macro we just defined into the command. Note that we've greatly boosted ctx-size for testing. The new flags need some explanation.

When you send something to an LLM, there are two processes that happen. The first is prefill, which is where the model takes your prompt and turns it into something it can understand. A simple chat message might have very fast and small prefill needs. Processing a a big chunk of code requires a lot of prefill. The second process is generation, which is the LLM creating the reply. Each can run at different speeds for different needs. Some of the new flags control that.

- **--ubatch-size** and **--batch-size**: Affects prefill. Bigger means the GPU uses more tokens per pass at the cost of VRAM. If we were to drop this from 2048 to 512, the LLM would have to chew a lot longer on a large message before you get your first token return, but the speed of generation afterward is the same.
- **--n-cpu-moe**: Affects generation. The model we are using is called a "mixture of experts" model, and this flag pushes a number of experts to the CPU. We don't want that, so we set it to 0.
- **--cache-reuse**: Speeds up prefill by reusing what is alrady in the cache if it has already been computed.

The ttl entry (time to live) is a timer in seconds for how long llama-swap will keep an idle connection open before shutting down. This frees up VRAM if you don't touch things for a while. At 1800, this gives half an hour of idle time.

Aliases are just that. They're another name we can reference for this model.

## Groups

A group is how llama-swap determines what gets shut down when a request arrives when a model isn't loaded. We only have one model now, but here's the entry. Place at the bottom of the file and save it.

```
groups:
  chat:
    swap: true
    exclusive: false
    members:
      - "gpt-oss-20b"
```

A group is made up of members. We have named this group "chat".

- **swap:** is a boolean that, when true, says "if another model in this group is called, unload the current one." If it was false, then models in the group can be live at the same time.
- **exclusive:** is like swap, but for entire groups. If it was true, loading any member of this group will kill the models of every other group.
- **members:** are the models under this group. They must be the actual model name, not an alias.


This completes the configuration for llama-swap. Now we need to activate it.

## Setting up the llama-swap service

