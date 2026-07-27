# agentics

Tracking the setup and use of claude code for infrastructure tasks.

## RoCM vs VULKAN on 7900 XTX Ollama Backend

We need to validate RoCM vs Vulkan for the backend. There are close enough on smaller and MoE models they are effectively the same, dense models it looks like RoCM may be more efficient. We will stick with RoCM. 

Insall the current RoCM version for your distro.

- main pc 
    - 67.28 tokens/s phi4:latest
    - 96.49 tokens/s qwen3.6:35b-a3b-q4_K_M
    - 32.61 tokens/s granite4.1:30b

```bash
[Service]
Environment="HSA_OVERRIDE_GFX_VERSION=11.0.0"
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KV_CACHE_TYPE=q8_0"
```

- main pc vulkan setup 
    - 72.03 tokens/s phi4:latest
    - 93.79 tokens/s qwen3.6:35b-a3b-q4_K_M
    - 17.88 tokens/s granite4.1:30b

```bash
Environment="OLLAMA_VULKAN=1"
Environment="GGML_VK_VISIBLE_DEVICES=0"
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KV_CACHE_TYPE=q8_0"
```

## Deciding Which Model to Use

TBD

## Ollama Setup

```bash
# Download Ollama
curl -fsSL https://ollama.com/install.sh | sh

sudo systemctl enable ollama.service

sudo systemctl edit ollama.service

# add for rdna3
[Service]
Environment="HSA_OVERRIDE_GFX_VERSION=11.0.0"
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KV_CACHE_TYPE=q8_0"
# save and exit Ctrl-O Ctrl-X

sudo systemctl daemon-reload
sudo systemctl restart ollama.service

ollama pull XYZmodels
```

## Claude Setup

```bash
# Download Claude
curl -fsSL https://claude.ai/install.sh | bash

# We need to edit env vars in the shell to tell claude to use ollama

# fish shell on arch/garuda linux example
vim .config/fish/config.fish

# Lets add overrides for RoCM and claude
set -x ROCM_PATH /opt/rocm
set -x HSA_OVERRIDE_GFX_VERSION 11.0.0
# Redirect Claude Code to target your local Ollama port
set -gx ANTHROPIC_BASE_URL "http://localhost:11434"
# Set the authentication token signature for the local engine wrapper
set -gx ANTHROPIC_AUTH_TOKEN "ollama"
# Completely wipe out any global cloud keys to prevent billing conflicts
set -gx ANTHROPIC_API_KEY ""

# If you are using bash shell they will look like this
vim ~/.bashrc

export ROCM_PATH="/opt/rocm"
export HSA_OVERRIDE_GFX_VERSION="11.0.0"
export ANTHROPIC_BASE_URL="http://localhost:11434"
export ANTHROPIC_AUTH_TOKEN="ollama"
export ANTHROPIC_API_KEY=""

# Save and exit, re-source your shell or open a new terminal

# If you want to not have to override anthropic vars you can use
ollama launch claude --model somemodelhere
```

## Optimize Models

We need to change some of the default behaviors on the models before use them for tooling. IE the context and response limits are generally to low by default depending on how you are using claude/tools.

We will setup `qwen3-coder:30b` with a modelfile and have a custom flavor to utilize with claude.

```bash
cd ~/Documents
vim Modelfile
```

We want to add the following to that `Modelfile`. 

- FROM <Model we are tuning>
- PARAMETER num_ctx <Context Window Size in Tokens>
- PARAMETER num_predict <Max Reponse Size in Tokens>
- SYSTEM <Directions to give the model for this role>

```bash
FROM qwen3-coder:30b
PARAMETER num_ctx 65536
PARAMETER num_predict 8192
SYSTEM """You are an elite, production-grade Linux systems administrator and automation agent. Your job is to perform system engineering tasks including remote SSH management, local bash execution, regex file parsing, and system configuration.

CRITICAL OPERATIONAL RULES:
1. Always utilize the provided tool-calling or function-calling syntax schemas required by the frontend client (e.g., Claude Code) to execute actions. 
2. Never output destructive or unverified formatting commands (like raw 'rm -rf /' without safe targeting vectors).
3. When writing automation workflows, structure your file edits using precise diff formats or sed/awk operations to preserve file integrity across deep directories.
4. Keep your internal logical reasoning sharp and concise, transitioning immediately into structured system commands."""
```

Save and exit.

With our KVCACHE set to FP8 we should be able to keep all of this context on VRAM. If it does spill over we should still have good speeds.

Create the customized version: `ollama create qwen3-core-claude-agent -f ./Modelfile`

```bash
╰─λ ollama list | grep qwen3
qwen3-core-claude-agent:latest               f1cb65d8300f    18 GB     15 seconds ago
qwen3.6:35b-a3b-q4_K_M                       07d35212591f    23 GB     25 hours ago
rafw007/qwen36-a3b-claude-coder:latest       36853d5c1fed    23 GB     34 hours ago
qwen3.6:27b                                  a50eda8ed977    17 GB     34 hours ago
qwen3-coder:30b                              06c1097efce0    18 GB     2 days ago
```

Our new version `qwen3-core-claude-agent:latest` is now available.

Start claude with: `claude --model qwen3-core-claude-agent:latest`

This this setup I was able to successfully run system command, claude determined its own capabilities, and I could ssh to my mini pc and fetch details.

## Claude Skills

We need to leverage claude skills to give our agent some further direction and scope. These skills are simply structured markdown files. These are what set the SOP (Standard Operating Procedures) for the agent. There is a global way to set this for ALL sessions and a project level way to set. We can also set a global 

```bash
 ~/.claude/                      <-- 1. GLOBAL / PERSONAL LAYER
 └── skills/
     └── skillname-action/       <-- Give the folden a name with an intent
         └── SKILL.md            <-- Available in ALL your terminal sessions

 [your-project-folder]/          <-- 2. PROJECT LAYER
 └── .claude/
     └── skills/
         └── skillname-action/   <-- Give the folden a name with an intent
             └── SKILL.md        <-- Only available inside this specific repository

 [your-project-folder]/          <-- 3. PROJECT LAYER
 └── CLAUDE.md                   <-- static project memory
```

- CLAUDE.md
    - Tells the model everything it needs to know about your codebase/project on session initialization
    - Build and Test Commands, tell the model how to build, test, run the app with exact commands
    - Cody Style Guidelines
        - What language, what type of program components, imperitive etc...
    - Architecture notes
        - Tell the model where specific useful files live like (IP list X, csv Y is here)
    - CLAUDE.md fully loads on EVERY turn/step so keep it as short as possible otherwise it will eat the context window rapidly
- ~/.claude/skills
    - These are skills that can be used across multiple projects
        - git commit message formatter
        - code styling
- ~/myproject/.claude/skills
    - How to handle localized workflow specific to this exact code/project
    - Deployment, testing, specific executions

Skills have 3 main components, the YAML metadata block and the body which contains the content, and resources (optional).

YAML Frontmatter: Requires a skill name and skill description

```yaml
---
name: system-ping-check
description: Pings various infrastructure systems, checks success or failure status, and measures latency. Use when checking uptime, network health, or endpoint performance.
allowed-tools: [bash]
---
```

Body/Content: This is the actual work/content that the model will use. It should contain procedures, knowledge, workflows, best practice and any guidance.

```bash
# System Ping Check

## Parameters
* **host**: The target IP address or domain name (e.g., `10.0.0.1` or `google.com`).

## Quick Start
Use the terminal utility `ping` to gather packet loss and latency data about a host. 

### Execution Rules
* **Count Limit**: You must always limit the ping to a maximum count of 4 to prevent hanging the terminal session.
* **Syntax**: Use the `-c` flag followed by the count.

### Example Command
`ping -c 4 <host>`

### Expected Output Interpretation
* **Success**: 0% packet loss, valid round-trip time (rtt) min/avg/max values.
* **Failure**: 100% packet loss, "Unknown host", or "Request timeout".
* **Warning**: When round-trip time exceeds 100 ms for avg or max values.
```

The claude skill files are ingested and inferred by the model, there is no callouts in the cli chat. Because we are trying systems actions and not looking at a code base we can simply ask the cli to ping xyz.

Run claude, here is the output:

```bash
❯ please run a system ping check on 10.0.0.102, 8.8.8.8, google.com, and facebook.com

...
...
...

● System Ping Check Results

I've completed ping checks on all requested targets. Here are the results:

1. 10.0.0.102

- Status: Success
- Packet Loss: 0%
- Response Times: 4.700/5.754/7.073 ms (min/avg/max)
- Notes: All packets received with low latency

2. 8.8.8.8

- Status: Success
- Packet Loss: 0%
- Response Times: 12.179/15.324/17.403 ms (min/avg/max)
- Notes: All packets received with moderate latency

3. google.com

- Status: Success
- Packet Loss: 0%
- Response Times: 13.730/15.983/18.039 ms (min/avg/max)
- Notes: All packets received with low to moderate latency

4. facebook.com

- Status: Success
- Packet Loss: 0%
- Response Times: 14.363/15.652/16.433 ms (min/avg/max)
- Notes: All packets received with low latency

Overall Assessment

All targets are reachable with no packet loss. The network connectivity appears to be functioning properly. Response times are within
normal ranges for typical internet connectivity, with the fastest responses to local network targets (10.0.0.102 and 8.8.8.8) and
slightly higher latencies for external sites (google.com and facebook.com). No warnings or failures detected in any of the ping
tests.
```

This doesn't scale and is about as inefficient as possible, takes way to much power and vram to just ping somethings. However we can execute commands, leverage local tools, and use the llm to determine success on those commands. Writing code/ansible to execute pings across a cluster of x nodes, logging the output then processing that output looking for outliers would be something an llm can do without writing code. The same goes for drivers, udev rules, grub, and just about anything system related. 

The trick is to know what you want to execute and how to get data, then let the model interpret the data.