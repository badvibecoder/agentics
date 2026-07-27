# agentics

Tracking the setup and use of claude code for infrastructure tasks.

## RoCM vs VULKAN on 7900 XTX Ollama Backend

We need to validate RoCM vs Vulkan for the backend. They are close enough on smaller and MoE models they are effectively the same, dense models it looks like RoCM may be more efficient. We will stick with RoCM. 

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

### Sidequest - GFX Verions

List of overrides.

```bash
================================================================================
GCN (Graphics Core Next) Architectures
================================================================================

GFX6 (GCN 1.0 / Southern Islands):
  - gfx600 (Tahiti)
  - gfx601 (Pitcairn)

GFX7 (GCN 2.0 / Sea Islands):
  - gfx700 (Kaveri APU)
  - gfx701 (Hawaii - R9 290)

GFX8 (GCN 3.0 & 4.0 / Volcanic & Arctic Islands):
  - gfx801 (Carrizo)
  - gfx803 (Fiji / Polaris - RX 480/580)

GFX9 (GCN 5.0 & 5.1 / Vega):
  - gfx900 (Vega 56/64)
  - gfx902 (Raven Ridge APU)
  - gfx906 (Radeon VII / MI50)

================================================================================
CDNA (Data Center / Instinct) Architectures
================================================================================

GFX9.0.A (CDNA 1 & 2):
  - gfx908 (Instinct MI100)
  - gfx90a (Instinct MI250X)

GFX9.4 (CDNA 3):
  - gfx940 (Instinct MI300A / MI300X)
  - gfx941 (Instinct MI300A / MI300X)
  - gfx942 (Instinct MI300A / MI300X)
qwen3-coder:30b-a3b-q8_0
================================================================================
RDNA (Radeon DNA) Consumer Architectures
================================================================================

[RDNA 1 (Navi 1x)]
GFX10.1.0: gfx1010 (Navi 10 - RX 5700 XT)
GFX10.1.1: gfx1011 (Navi 12)
GFX10.1.2: gfx1012 (Navi 14 - RX 5500 XT)

[RDNA 2 (Navi 2x)]
GFX10.3.0: gfx1030 (Navi 21 - RX 6800 / 6900 XT)
GFX10.3.1: gfx1031 (Navi 22 - RX 6700 XT)
GFX10.3.2: gfx1032 (Navi 23 - RX 6600 XT)
GFX10.3.4: gfx1034 (Navi 24 - RX 6400 / 6500 XT)
GFX10.3.5: gfx1035 (Rembrandt / Mendocino APUs - Radeon 680M)
GFX10.3.6: gfx1036 (Sabrina APU)

[RDNA 3 / RDNA 3.5 (Navi 3x)]
GFX11.0.0: gfx1100 (Navi 31 - RX 7900 XTX)
GFX11.0.1: gfx1101 (Navi 32 - RX 7700 XT / 7800 XT)
GFX11.0.2: gfx1102 (Navi 33 - RX 7600 XT)
GFX11.0.3: gfx1103 (Phoenix / Hawk Point APUs - Radeon 780M)
GFX11.5.0: gfx1150 (Strix Point APUs - Ryzen AI 9 / Radeon 890M)
GFX11.5.1: gfx1151 (Strix Halo APUs)

[RDNA 4 & 4m (Navi 4x)]
GFX11.7.0: gfx1170 (RDNA 4m APU architectures)
GFX11.7.1: gfx1171 (Mobile variants)
GFX11.7.2: gfx1172 (Mobile variants)
GFX12.0.0: gfx1200 (Navi 44 - RX 9060 Series)
GFX12.0.1: gfx1201 (Navi 48 - RX 9070 ￼XT / Pro R9700)
```

## Deciding Which Model to Use

Must have:

- Native tool calling
- Prefer dense models but MoE will work as well
- 27b-35b models for large 24GB-32GB single gpu setups
- 70b+ models for large dual gpu setups (48-64GB)
- Non-thinking models
- Suggested Models
    - Qwen3-Coder (30B Dense)
    - Qwen 2.5 Coder (32B Instruct)
    - Qwen 3.6 (27B Dense)

If you are running a single gpu and have a good amount of system RAM, 64-128G don't be afraid to grab a larger quant model than you think. There is a significant quality drop off between q8 and q4 for models under 35-40b parameters. It does depend on how the model is trained and structured, however over subscribing and getting a q8 model even if slower may perform much better at keeping the context straight over longer runs.

For instance:

- Speed Check between q4/q8
    - 9800X3D - 96GB 6000 - 7900 XTX 24GB
        - qwen3-coder:30b-a3b-q4_K_M
            - 109.75 tokens/s
        - qwen3-coder:30b-a3b-q8_0
            - 24.83 tokens/s

25 tokens a second is pleanty fast if it indeed keeps the accuracy near perfect and during long sessions with deep context keep the halucinations and looping to a minimum.

CPU/GPU splits

```bash
╰─λ ollama ps
NAME                              ID              SIZE     PROCESSOR    CONTEXT    UNTIL
qwen3-core-claude-agent:latest    f1cb65d8300f    22 GB    100% GPU     65536      4 minutes from now

total duration:       5.02384267s
load duration:        98.112513ms
prompt eval count:    170 token(s)
prompt eval duration: 203.945ms
prompt eval rate:     833.56 tokens/s
eval count:           518 token(s)
eval duration:        4.719938s
eval rate:            109.75 tokens/s

╰─λ ollama ps
NAME                                 ID              SIZE     PROCESSOR          CONTEXT    UNTIL
qwen3-code-q8-claude-agent:latest    553988006ba1    36 GB    33%/67% CPU/GPU    65536      4 minutes from now

total duration:       11.776264955s
load duration:        97.702624ms
prompt eval count:    170 token(s)
prompt eval duration: 36.93ms
prompt eval rate:     4603.30 tokens/s
eval count:           289 token(s)
eval duration:        11.639627s
eval rate:            24.83 tokens/s
```

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

Create the customized version: `ollama create qwen3-code-claude-agent -f ./Modelfile`

```bash
╰─λ ollama list | grep qwen3
qwen3-code-claude-agent:latest               f1cb65d8300f    18 GB     15 seconds ago
qwen3.6:35b-a3b-q4_K_M                       07d35212591f    23 GB     25 hours ago
rafw007/qwen36-a3b-claude-coder:latest       36853d5c1fed    23 GB     34 hours ago
qwen3.6:27b                                  a50eda8ed977    17 GB     34 hours ago
qwen3-coder:30b                              06c1097efce0    18 GB     2 days ago
```

Our new version `qwen3-code-claude-agent:latest` is now available.

Start claude with: `claude --model qwen3-code-claude-agent:latest`

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

Claude also cannot run as root, there are other limitations but it can fully employ ansible. There is even an MCP (API) for claude to interact with ansible. I also try to build a base myself then translate that over to some form of ansible task, or break down into python and have ansible distribute and execute. One of the issues with large scale systems is accounting for issues, errors, things that you would have to read and cannot easily account for with a if/then/switch logic. But the llm can read the output and infer much more than we can easily account for in code.

Lets try another skill and setup claude code to not require sudo for running elevated commands in arch.

```bash
which claude
# /home/yourusername/.local/bin/claude

sudo visudo -f /etc/sudoers.d/claude-code

# add the line
yourusername ALL=(ALL) NOPASSWD: /home/yourusername/.local/bin/claude
# save and exit
```

This will ensure claude has root but we need to make sure the user (yours or a service account) will have passwordless sudo to tools as well.

```bash
# create a new file
sudo visudo -f /etc/sudoers.d/claude-system

# add the line
yourusername ALL=(ALL) NOPASSWD: /usr/bin/systemctl, /usr/bin/hostnamectl, /usr/bin/netplan, /usr/bin/ip, /usr/bin/ssh, /usr/bin/udevadm, /usr/bin/grub-mkconfig, /usr/bin/grub-install, /usr/bin/fwupdmgr, /usr/bin/modprobe, /usr/bin/journalctl, /usr/bin/dmesg, /usr/bin/lshw, /usr/bin/pacman
# add any bin tool or service that you want claude to access
```

The simpler alternative would be to create a claude service account and just give it access to everything.

Lets make a simple skill that requires sudo.

````bash
---
name: check-amd-overdrive
description: Run a bash command to check AMD OverDrive status in dmesg.
allowed-tools: [Bash]
---
Execute this exact Bash command right now. Do not write an introduction. Do not use code blocks. Just run the tool.

Command: sudo dmesg | grep -i "overdrive"

If the output says "enabled", reply: AMD OverDrive is active.
If the output is empty, reply: AMD OverDrive is disabled.
````

My prompt: "/check-amd-overdrive"

Output:

```bash
❯ /check-amd-overdrive

● Bash(sudo dmesg | grep -i "overdrive")
⎿  [    8.279801] amdgpu: Overdrive is enabled, please disable it before reporting any bugs unrelated to overdrive.

● AMD OverDrive is active.

✻ Cooked for 3s
```

I had to play with the skill file a bit, sometimes it would go back and forth between executing and simply pasting my text into <function> which does nothing.

One issue I was having was putting the code in markdown code blocks, this was not being interpretted reliably. Stripping the introduction post yaml data. Then pasting the command as a raw string and not in quotes or a block made the parser happy.

### We Have Overcome `sudo` Limits With Claude

This again is great, but it does not scale whatsoever. We cannot simply use claude to login to 10000 nodes to analyze and troubleshoot. But its a step in the right direction. We are able to overcome the sudo limitation and get dmesg logs which means we can also do other oneshot commands that do not require input but do need sudo.

I think the next step is to loop in ansible, which enables us for nearly all the system mgmt/statemgmt features we could want. It also allows claude to scale to massive levels. We can leverage ansible's sudo access, package mgmt, code deployment and use claude to drive decisions and if needed step in as a level one troubleshooter so we dont waste time logging into singletons to find out a link is down or a netplan wasnt applied etc...


