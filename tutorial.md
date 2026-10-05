# Setting up OpenCode container on Debian system without admin privileges

## OpenCode in a Singularity Container (no root)

These installation instructions document how to build and run an OpenCode sandbox in a Singularity container on a Debian system without admin privileges.

---

> [!NOTE] 
> **Features of this sandbox container:**
> - `/data` is not available in the container **(MOST IMPORTANT)**
> - home directory name (`$HOME`) is exactly the same in the container and outside but only explicitly specified directories (e.g., `~/.config`,`~/.cache`,`~/.local`) and files from it are available within the container
>     - if you want to automatically push to Github/Gitlab, you need to mount `~/.ssh/` or certain files of it but I would not recommend this approach.

### 1. Singularity definition file

Create a file named `opencode_singularity.def`:

```
Bootstrap: docker
From: node:20-slim

%environment
    export WORKSPACE=/workspace

%post
    # Install OpenCode globally via npm
    npm install -g opencode-ai

    # Create working directory
    mkdir -p /workspace

%runscript
    cd /workspace
    exec opencode "$@"
```

Notes:

- Uses `Bootstrap: docker` to pull `node:20-slim` as the base image.
- Installs `opencode-ai` via npm in the `%post` section.
- Sets `/workspace` as the working directory inside `%runscript` (avoids unsupported `%workdir` section).

---

### 2. Build the container (fakeroot, no root)

From the directory containing `opencode_singularity.def`:

```bash
singularity build --fakeroot opencode.sif opencode_singularity.def
```

This creates a single-file SIF image: `opencode.sif`.

I would recommend building this in the `/tmp` of your local machine. The reason is that building on storageunified (`/data`) does not work and building on a compute server failed with large containers for me more often than using the local machine.

---

### 3. Shell function to run OpenCode

Create a file `bash.opencode` (e.g. in `~/.config/opencode/` or your project directory):

```bash
#!/bin/bash

# Make run-opencode available as shell function.
# The function spins up a container running opencode inside.

## Check if singularity container file exists
OPENCODE_SIF="/data/u_kuegler_software/opencode/opencode.sif"
if [ -f "$OPENCODE_SIF" ]; then
  VALID_OC_CONTAINER=true
else
  VALID_OC_CONTAINER=false
  echo "Opencode singularity container not available. Skipping definition of run-opencode()."
fi

## Check if SAIA_API_KEY is defined
if [ -n "${SAIA_API_KEY}" ]; then
  VALID_API=true
else
  VALID_API=false
  echo "SAIA_API_KEY not defined or empty. Skipping definition of run-opencode()."
fi

## Define relevant binds within the home directory, as home directory is excluded in the container
binds=(
  "-B ${HOME}/.cache:${HOME}/.cache"
  "-B ${HOME}/.config:${HOME}/.config"
  "-B ${HOME}/.local:${HOME}/.local"
)

## Define run-opencode() function or a placeholder.
if [ "$VALID_API" = true ] && [ "$VALID_OC_CONTAINER" = true ]; then
  echo "Activating run-opencode command."
  run-opencode() {
    local extra_mount="${1:-}"

    # Dynamically create the final bind array from the default binds
    local mybinds=("${binds[@]}")
    if [ -n "$extra_mount" ]; then
      mybinds+=("-B ${extra_mount}:/other")
    fi

    singularity run \
      --no-home \
      --no-mount "/data" \
      -B "$PWD":/workspace \
      "${mybinds[@]}" \
      -W /workspace \
      --env SAIA_API_KEY="$SAIA_API_KEY" \
      "$OPENCODE_SIF" "$@"
  }
else
  # Define a stub function to avoid "command not found" errors
  run-opencode() {
    echo "run-opencode: Function disabled because required environment variables are missing. Please check your ~/.bashrc !" >&2
    return 1
  }
fi
```

Key points:

- `OPENCODE_SIF` is defined once; update this if you move the `.sif` file.
- a few checks if the necessary files and variables are available
- `run-opencode`:
    - Mounts the current directory at `/workspace`.
    - Optionally mounts a second directory at `/other` (first argument). This can be used to make testdata available (WARNING: isolate that test data into a separate directory, so that opencode is not able to access `/data`)
    - binds `.cache/` and `.config/` from the home directory (`.config` contains the opencode.json necessary for the GWDG models)
        - optionally add more directories to the `binds` array
    - forwards `SAIA_API_KEY` into the container (must be globally defined in your `.bashrc` before)
    - Passes any additional arguments to `opencode` via `"$@"`.

---

### 4. Integrate into your shell

In your `~/.bashrc`, add (adjust the path to `bash.opencode`):

```bash
# make the function available manually by running the runoc alias

alias runoc="source ~/.bashrc.opencode" # make run-opencode command available to run opencode in custom singularity container
```

Then reload your shell:

```bash
source ~/.bashrc
```

---

### 5. Pull information on GWDG models

We will use this plugin to pull up-to-date information about GWDG-hosted models every time OpenCode starts.

```bash
mkdir -p ~/.config/opencode/plugins

curl -fsSL <https://raw.githubusercontent.com/jaisonlewis/opencode-saia-plugin/master/src/saia.ts> \
  -o ~/.config/opencode/plugins/saia.ts

curl -fsSL <https://raw.githubusercontent.com/jaisonlewis/opencode-saia-plugin/master/src/generate-saia-config.mjs> \
  -o ~/.config/opencode/plugins/generate-saia-config.mjs
```

Create the `opencode.json` as config file

```bash
~/.config/opencode/opencode.json
```

and add the following code block to it:

```bash
{
  "$schema": "<https://opencode.ai/config.json>",
  "plugin": ["./saia"]
}
```

Now you can pull all information about the available models on GWDG-servers by running:

```bash
cd ~/.config/opencode/plugins
node generate-saia-config.mjs
```

### 6. How to use OpenCode

From any directory:

```bash
# Make the shell function run-opencode available 
runoc

# Run OpenCode with current directory mounted at /workspace
run-opencode

# Run with an additional directory mounted at /other
run-opencode /path/to/other/dir

# Pass arguments to opencode itself
run-opencode --some-flag file.txt
run-opencode /path/to/other/dir --some-flag file.txt
```

Inside the container:

- Working directory: `/workspace` (your host `PWD`).
- Optional extra mount: `/other` (host path you passed as first argument).
- `HOME` cannot be overwritten, so this stays the same. However, the directory is not mounted. Therefore, I manually mount a few directories from HOME to the container. Your home directory in the container will be called the same as outside, but only contains a few selected directories.

### 7. Testing the container

Create a bash script with the same content, but change the `singularity run` to `singularity shell`. Optionally adjust the name of the function. Now source the file. After that, the shell function will launch the same container interactively rather than running the opencode GUI.