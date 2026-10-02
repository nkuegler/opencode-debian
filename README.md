# opencode-debian

Purpose: Running a sandbox opencode container in your Linux environment without admin privileges.

## Content
- Singularity container build file for opencode sandbox 
- scripts for easy use in your environment

## Tutorial
Step-by-step guide how to set up and use the opencode container.

## Security features
- `/data` is not available in the container **(MOST IMPORTANT)**
- home directory name (`$HOME`) is exactly the same in the container and outside but only explicitly specified directories (e.g., `~/.config`, `~/.cache`, `~/.local`) and files from it are available within the container
    - if you want to automatically push to Github/Gitlab, you need to mount `~/.ssh/` or certain files of it but I would not recommend this)
