# Documentation for installing a local LLM stack on CachyOS on an AMD RX 9070 XT or similar cards

The goal of this document is to create a local LLM service stack on CachyOS with an AMD RX 9070 XT. This card has 16 GB of VRAM. 

There are plenty of docker images that purport to do this for you, but the options for AMD are limited and the assumption throughout is NVIDIA cards or CUDA support. So, this is my attempt at doing it from scratch.

Last updated: 2026-09-20

## Starting Assumptions:

1. You have CachyOS installed and updated.
2. You have some familiarity with Linux and Arch-based systems.
3. You have CachyOS's AMD kernel drivers installed. I switched from an NVIDIA card to an AMD card, so i used the [instructions from CachyOS here](https://wiki.cachyos.org/features/chwd/gpu_migration/) to do the driver flip.
4. You have an AMD card from the RDNA4 class. These are cards in the Radeon RX 9000 series. This may work with other AMD cards but I cannot guarantee it.
5. You have only one discrete GPU, not multiple cards (integrated GPU with your motherboard does not count).
6. At least 32 GB of system RAM.
7. Sufficient hard drive space, preferably on an NVMe. I'm not sure exactly what sufficient is yet; I have a 1 TB drive for my OS and a 2 TB drive for ~/.

## First Packages

You need five packages to start:

- *llama-cpp* - This is the current heart/brain of many local LLM setups. It is a C++ inference engine that runs model files in GGUF format. It creates several programs we'll use further below, like llama-server, llama-cli, llama-bench, llama-quantize, etc.
- *ggml-vulkan* - GGML is a library underneath llama-cpp that does the actual math on your GPU. It is compiled to different GPU architectures. Vulkan is one of the two libraries you can run on AMD cards and is the default preference currently.
- *ggml-hip* - GGML, but now for the ROCm architecture which is the secret proprietary sauce of AMD's cards. HIP is AMD's CUDA-alike API. (Side note: CUDA is NVIDIA's platform for doing general-purpose computation on a GPU, and it is the default assumption baked into a lot of AI software because they were first.)
- *vulkan-radeon* - This is Mesa's open-source Vulkan driver for AMD cards. You may already have this, but it's good to double-check.
- *llama-swap-bin* - This is a reverse proxy that helps with model selection and with loading and unloading models. More on this below.

It is safe to have both ggml-vulkan and ggml-hip on the same computer. We can switch between them in later steps to benchmark a model's performance using either library.

At this stage, assuming you have the AMD drivers installed for the kernel, here is the stack:

amdgpu (kernel driver, cachy should install) -> vulkan-radeon (userspace driver) -> ggml-vulkan/ggml-hip (libraries that do computation on the card) -> llama-cpp (inference engine that loads the models and runs commands agains them) -> llama-swap (reverse proxy to make working with llama-cpp easier.)

*Installation*

```sudo pacman -S llama-cpp ggml-vulkan ggml-hip vulkan-radeon```

```paru -S llama-swap-bin```

If you do not have paru to install packages from the AUR, use your preferred AUR downloader or use pacman to install paru.

## Driver Sanity Check and Diagnostic Tools

Before we go deeper into the LLM side, let's make sure our drivers are correct.

```vulkaninfo --summary | grep -i -E "deviceName|driverName"```

vulkaninfo is a diagnostic tool. We use it to confirm the card exists to the Vulkan driver. On my machine, this returns:

```
WARNING: radv is not a conformant Vulkan implementation, testing use only.
	deviceName         = AMD Radeon RX 9070 XT (RADV GFX1201)
	driverName         = radv
```

```radv``` is the driver we want. You can ignore the warning; it's related to a certification process that the driver has not gone through. If the driverName and deviceName look right, then your card drivers are loaded.

Next, run:

```llama-server --list-devices```

I get back:

```
WARNING: radv is not a conformant Vulkan implementation, testing use only.
Available devices:
  ROCm0: AMD Radeon RX 9070 XT (16304 MiB, 16192 MiB free)
  Vulkan0: AMD Radeon RX 9070 XT (RADV GFX1201) (16304 MiB, 13910 MiB free)
```

We've asked llama-server (part of llama-cpp) whether it can see the card now. There are two devices because we installed ggml-vulkan and ggml-hip. This is normal. As before, you can ignore the warning.

*Take note of the two device names: ROCm0 and Vulkan0.* We will be referencing those later. 

Also take note that the amount of free memory is different for each one. This doesn't not mean we've doubled your VRAM! It's two different reports on the same card. Vulkan0, in this example, is reporting that some of my memory is taken up already by my desktop environment, which is run on Vulkan0. This is normal. If you start using both device types for different models and start finding yourself out of memory, use this command to see how the memory is split. That will tell you where you might need to unload something.

If you've gotten this far, congratulations! Your drivers are complete and llama-cpp is ready to work with them.

*Benchmarking Tool*

Now, install ```rocm-smi``` from the AUR. For whatever reason, I had to install this using ```yay``` instead of ```paru```. Install yay if you haven't using ```pacman```, then input:

```yay rocm-smi```

```rocm-smi``` is a system management interface for AMD GPUs. It's not strictly necessary, but it gives another window into how your card is doing. It may become useful if you push things.

If I run ```rocm-smi```, I get the following:

```
WARNING: AMD GPU device(s) is/are in a low-power state. Check power control/runtime_status

========================================= ROCm System Management Interface =========================================
=================================================== Concise Info ===================================================
Device  Node  IDs              Temp    Power  Partitions          SCLK    MCLK     Fan  Perf  PwrCap  VRAM%  GPU%
              (DID,     GUID)  (Edge)  (Avg)  (Mem, Compute, ID)
====================================================================================================================
0       1     0x7550,   3844   31.0°C  24.0W  N/A, N/A, 0         762Mhz  1258Mhz  0%   auto  330.0W  14%    8%
====================================================================================================================
=============================================== End of ROCm SMI Log ================================================
```

Weirdly, this tool will report a warning when your card is idle. This is normal though. I have nothing active going to the card beyond the basics. If I suspect power problems, temperature problems, fan problems, or think VRAM or GPU usage is pegged, this will confirm it directly from the card.

## Get Your First Model

I use HuggingFace to find models, and I find it is easier to use HuggingFace's tools than to have ```llama-server``` do the download for me.

### Preparation

Install python-huggingface-hub:

```sudo pacman -S python-huggingface-hub```

Now create a directory wherever you want to hold the models. I will be using ```~/models```. If you choose another location, you will need to update the configuration files later to point to your chosen location.

```
cd ~
mkdir ./models
```
### Accessing HuggingFace

HuggingFace is a repository of models. It will be easier on you if you make a HuggingFace account now at [huggingface.co](https://huggingface.co) because you'll get faster download speeds.

Once you make an account, go to [https://huggingface.co/settings/tokens](https://huggingface.co/settings/tokens). Click on the "+ Create new token" button. Choose a Read token type and give it a name, then click Create New Token. A modal will pop up with a key. Copy that key and paste it into a text file. Do not close the window before this! If you do, delete the token and create a new one.

Now, log into HuggingFace on the command line.

```hf auth login```

Choose "Paste an access token", copy your token string, then use ```Ctrl+shift+v``` to paste it into the console and hit enter. You will not see the pasted content.

You'll see something like:

```
❯ hf auth login
? How would you like to log in? Paste an access token
    To log in, `huggingface_hub` requires a token generated from https://huggingface.co/settings/tokens .
Enter your token (input will not be visible):
Token is valid (permission: read).
The token `read-tok` has been saved to /home/$USER/.cache/huggingface/stored_tokens
Your token has been saved to /home/$USER/.cache/huggingface/token
Login successful.
The current active token is: `read-tok`
```

$USER is a substitute for my username, and read-tok is the name I gave my token. 

If your token key ever gets exposed, revoke it from HuggingFace's website. Treat it like a password.

You can confirm you're authenticated by typing:

```hf auth whoami```

### Get a Model

It is very important to choose a model that fits your card, and this can involve a lot of experimentation. We are going to get a model from the [unsloth/gpt-oss-20b-GGUF]([unsloth/gpt-oss-20b-GGUF](https://huggingface.co/unsloth/gpt-oss-20b-GGUF) repository.

Let's see what's in this repository first:

```hf download unsloth/gpt-oss-20b-GGUF --dry-run```

**Very Important:** Do not neglect the --dry-run flag or you'll download the whole repository!

In the output I get something like:

```
❯ hf download unsloth/gpt-oss-20b-GGUF --dry-run
[dry-run] Fetching 22 files: 100%|█████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 22/22 [00:00<00:00, 52.32it/s]
Download complete: :                                                                                                                                                                                                                       |  0.00B            [dry-run] Will download 21 files (out of 22) totalling 179.4G.                                                                                                                                                                     |  0.00B /  0.00B
FILE                        SIZE
--------------------------- -----
.gitattributes              2.8K
README.md                   8.8K
config.json                 1.6K
gpt-oss-20b-F16.gguf        13.8G
gpt-oss-20b-Q2_K.gguf       11.5G
gpt-oss-20b-Q2_K_L.gguf     11.8G
gpt-oss-20b-Q3_K_M.gguf     11.5G
gpt-oss-20b-Q3_K_S.gguf     11.5G
gpt-oss-20b-Q4_0.gguf       11.5G
gpt-oss-20b-Q4_1.gguf       11.6G
gpt-oss-20b-Q4_K_M.gguf     -
gpt-oss-20b-Q4_K_S.gguf     11.6G
gpt-oss-20b-Q5_K_M.gguf     11.7G
gpt-oss-20b-Q5_K_S.gguf     11.7G
gpt-oss-20b-Q6_K.gguf       12.0G
gpt-oss-20b-Q8_0.gguf       12.1G
gpt-oss-20b-UD-Q4_K_XL.gguf 11.9G
gpt-oss-20b-UD-Q6_K_XL.gguf 12.0G
gpt-oss-20b-UD-Q8_K_XL.gguf 13.2G
notebook.ipynb              79.6K
params                      149.0
template                    7.4K
Download complete: :                                                                                                                                                                                                                       |  0.00B
Reconstruction complete: |                
```

Each .gguf file is a model that you can point ```llama-cpp``` toward. Let's get the Q4_K_M model and put it in our models directory.

```hf download unsloth/gpt-oss-20b-GGUF gpt-oss-20b-Q4_K_M.gguf --local-dir ~/models```

Change --local-dir to match wherever you want to put your models. 

Since we authenticated first with HuggingFace, you'll get the model very quickly. It took me at least 5 minutes to get a model before I set up authentication.

If you want to download other models, it's simply a matter of replacing the repository name and the file name. You can also download directly off of the HuggingFace website. Just make sure your models all land in your chosen directory. We will assume you're using this one though.

## Our First Test of the Model

Now we will use ```llama-server``` to load the model. Open a fresh terminal and use this command:

```
llama-server \
  --device Vulkan0 \
  --model ~/models/gpt-oss-20b-Q4_K_M.gguf \
  --n-gpu-layers 99 \
  --flash-attn on \
  --ctx-size 16384 \
  --host 127.0.0.1 \
  --port 8081
```

I used ```\```` to make it multiline so we can see all the parameters.

- **--device <device>**: What device are we using? It will either be ```Vulkan0``` or ```ROCm0``` in the setup.
- **--model <path+model>**: What model are we using? Point to your path and the file name.
- **n-gpu-layers 99**: Each model has a stack of transformer layers. When you send a token through, it passes through each one. This flag sets how many of those layers you want to have running on the GPU rather than CPU+RAM. Ideally, especially for your first runs, you want everything to run on the GPU. We set a very high number as a shorthand for "load all layers". If we omitted the flag, ```llama-cpp``` would default to running everything on the CPU!
- **flash-attn on**: We almost always want "flash-attention" on. The reasons why are complicated, but it will make your prompts faster and the KV cache memory lower, which gives us more room to work with. If you're turning this off, you should know why before you do.
- **ctx-size 16384**: This sets the context size. It is how many tokens the model can hold at once. The more you set, the more VRAM your model will take up above the overhead of loading the model into memory, but the more it will remember per conversation. If context size is too large, you could see errors in ```llama-cpp``` or the conversation will start dropping earlier tokens, which creates a loss of context. This is one of the major knobs to tweak with your experiements. 16384 (16K) is a good starting point for this model.
- **host <ip>**: This IS a server, so we need to say where we are hosting it. Right now, I am only hosting locally, so I point it to the localhost IP.
- **port <port>**: The port you're opening to create an access point to the server. 8081 is an arbitrary number I chose.

Run the command first to see if everything loads.

```
❯ llama-server \
        --device Vulkan0 \
        --model ~/models/gpt-oss-20b-Q4_K_M.gguf \
        --n-gpu-layers 99 \
        --flash-attn on \
        --ctx-size 16384 \
        --host 127.0.0.1 \
        --port 8081
WARNING: radv is not a conformant Vulkan implementation, testing use only.
0.00.130.367 I cmn  common_param: common_params_print_info: verbosity = 3 (adjust with the `-lv N` CLI arg)
0.00.130.591 W srv  llama_server: -----------------
0.00.130.591 W srv  llama_server: CORS is set to allow all origins ('*') and no API key is set
0.00.130.592 W srv  llama_server: this can be a security risk (cross-origin attacks)
0.00.130.592 W srv  llama_server: more info: https://github.com/ggml-org/llama.cpp/pull/25655
0.00.130.592 W srv  llama_server: -----------------
0.00.131.848 I srv    load_model: loading model '/home/$USER/models/gpt-oss-20b-Q4_K_M.gguf'
0.00.868.176 W load: setting token '<|message|>' (200008) attribute to USER_DEFINED (16), old attributes: 8
0.00.868.178 W load: setting token '<|start|>' (200006) attribute to USER_DEFINED (16), old attributes: 8
0.00.868.179 W load: setting token '<|constrain|>' (200003) attribute to USER_DEFINED (16), old attributes: 8
0.00.869.384 W load: setting token '<|channel|>' (200005) attribute to USER_DEFINED (16), old attributes: 8
0.00.875.715 W load: special_eog_ids contains both '<|return|>' and '<|call|>', or '<|calls|>' and '<|flush|>' tokens, removing '<|end|>' token from EOG list
0.23.437.498 I cmn          init: llama threadpool init, n_threads = 8
0.23.529.379 I srv    load_model: initializing, n_slots = 4, n_ctx_slot = 16384, kv_unified = 'true'
0.23.536.268 I srv  llama_server: model loaded
0.23.536.301 I srv  llama_server: listening on http://127.0.0.1:8081
```

The last two lines mean that ```llama-server``` has started and can be accessed at ```http://127.0.0.1:8081```. Some notes about the log:

- The numbers at the start are a timestamp since the server was started. The last line shows that ```lllama-server``` took 23 seconds, 536 milliseconds, and 301 microseconds to load this model.
- I and W are Info and Warning respectively.
- The line about verbosity is telling us how much we want the log to spit back at us. Useful for diagnosis. If you don't like all this startup noise, you can lower it by adding the ```-lv N``` tag, where N is the level you want. Numbers closer to 1 show fewer messages.
- The CORS warning lines are important, but only if we open up our server to the outside world or use it to browse outside sites. Eventually we will set this up.
- Everything from the load_model to the line with special_eog_ids is the model handshaking with llama-server. Despite the log saying it's a warning, it's normal. Different models will have different messages here.
- The init line that sets n_threads is the number of CPU cores that the model wants to have access to. This number will matter more if you start running models larger than your VRAM size.
- The load_model line after that has important information. n_slots is how many concurrent requests the server can handle. n_ctx_slot means each slot has 16384 tokens of context. kv_unified means all slots share the same key-value context. If we were running the server for a single purpose, we could tweak these numbers to make the model more performant. Let's leave it at the default for now.

Now, for a big test. Open a new terminal and send a curl request to your server:

```
curl -s localhost:8081/health
{"status":"ok"}
```

If you get the JSON there, the server is open to requests! Some things to try:

This will tell you what model it thinks it has loaded. FYI, jq is a JSON processor to help with printing and showing just the response we want from the returned JSON target.
```
curl -s localhost:8081/props | jq '.model_path'
```
Should return something like:

```
"/home/$USER/models/gpt-oss-20b-Q4_K_M.gguf"
```


And now let's send an actual request to the model and get something back:

```
curl -s --json '{"model":"x","messages":[{"role":"user","content":"hi"}]}' \
  localhost:8081/v1/chat/completions | jq -r '.choices[0].message.content'
```

Should return something like:

```
Hello! How can I assist you today?
```

Finally, you can point your browser to localhost:8081 (or whatever port you set), and get a nice shiny UI in your browser for talking with your model! We won't be using it later on, but it's a good sanity check.

## Where Are We At?

At this point, take a little break. You've done quite a lot! You've

- Got drivers and the LLM processing libraries talking to CachyOS and ```llama-cpp```
- Got a model from HuggingFace to play with and set up authentication with them to make it easy to download future models
- Got ```lllama-server``` to load your model
- Sent a request to the model and successfully got a response back
- Accessed ```llama-server```'s webUI to start talking.

If you are already familiar with how a REST API works, here are further details for ```llama-server```: [https://llama.app/docs/api](https://llama.app/docs/api)

Now we need to set up llama-swap for the next stage. See the [llama-swap page](./llama-swap.md) to continue this setup guide.
