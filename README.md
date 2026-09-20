# Documentation for installing a local LLM stack on CachyOS on an AMD RX 9070 XT or similar cards

The goal of this document is to create a local LLM service stack on CachyOS with an AMD RX 9070 XT. This card has 16 GB of VRAM. 

There are plenty of docker images that purport to do this for you, but the options for AMD are limited and the assumption throughout is NVIDIA cards or CUDA support. So, this is my attempt at doing it from scratch.

Last updated: 2026-09-20

## Starting Assumptions:

1. You have CachyOS installed and updated.
2. You have some familiarity with Linux and Arch-based systems
3. You have CachyOS's AMD kernel drivers installed. I switched from an NVIDIA card to an AMD card, so i used the instructions (here)[https://wiki.cachyos.org/features/chwd/gpu_migration/] from CachyOS do do the driver flip.
4. You have an AMD card from the RDNA4 class. These are cards in the Radeon RX 9000 series. This may work with other AMD cards but I cannot guarantee it.
5. At least 32 GB of system RAM
6. Sufficient hard drive space, preferably on an NVMe. I'm not sure exactly what sufficient is yet; I have a 1 TB drive for my OS and a 2 TB drive for ~/.

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

vulkaninfo is a diagnostic tool This confirms the card exists. On my machine, this returns:

```
WARNING: radv is not a conformant Vulkan implementation, testing use only.
	deviceName         = AMD Radeon RX 9070 XT (RADV GFX1201)
	driverName         = radv
```

```radv``` is the driver we want. You can ignore the warning; it's related to a certification process that the driver has not gone through. If the driverName and deviceName look right, then Vulkan-radeon recognizes your card.

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

Now, install rocm-smi from the AUR. For whatever reason, I had to install this using ```yay``` instead of ```paru```. Install yay if you haven't using ```pacman```, then input:

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

Weirdly, this tool will report a warning when your card is idle. This is normal though. I have nothing active going to the card beyond the basics. If I suspect power problems, temperature problems, fan problems, or want a direct confirmation from the card that my VRAM or GPU usage is pegged, this will confirm it directly from the card.
