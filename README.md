# Documentation for installing a local LLM stack on CachyOS on an AMD RX 9070 XT or similar cards

The goal of this document is to create a local LLM service stack on CachyOS with an AMD RX 9070 XT. This card has 16 GB of VRAM. 

There are plenty of docker images that purport to do this for you, but the options for AMD are limited and the assumption throughout is NVIDIA cards or CUDA support. So, this is my attempt at doing it from scratch.

## Starting Assumptions:

1. You have CachyOS installed and updated.
2. You have CachyOS's AMD drivers installed. I switched from an NVIDIA card to an AMD card, so i used the instructions (here)[https://wiki.cachyos.org/features/chwd/gpu_migration/] from CachyOS do do the driver flip.
3. You have an AMD card from the RDNA4 class. These are cards in the Radeon RX 9000 series.
4. At least 32 GB of system ram
5. Sufficient hard drive space, preferably on an NVMe. I'm not sure exactly what sufficient is yet; I have a 1 TB drive for my OS and a 2 TB drive for ~/.
