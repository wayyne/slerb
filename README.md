```text
        _            _             _            _           _        
       / /\         _\ \          /\ \         /\ \        / /\      
      / /  \       /\__ \        /  \ \       /  \ \      / /  \     
     / / /\ \__   / /_ \_\      / /\ \ \     / /\ \ \    / / /\ \    
    / / /\ \___\ / / /\/_/     / / /\ \_\   / / /\ \_\  / / /\ \ \   
    \ \ \ \/___// / /         / /_/_ \/_/  / / /_/ / / / / /\ \_\ \  
     \ \ \     / / /         / /____/\    / / /__\/ / / / /\ \ \___\ 
 _    \ \ \   / / / ____    / /\____\/   / / /_____/ / / /  \ \ \__/ 
/_/\__/ / /  / /_/_/ ___/\ / / /______  / / /\ \ \  / / /____\_\ \   
\ \/___/ /  /_______/\__\// / /_______\/ / /  \ \ \/ / /__________\  
 \_____\/   \_______\/    \/__________/\/_/    \_\/\/_____________/

                                 written by: Guy W. Dayhoff II, Ph.D.
```                                                                    

**SSH-Launched Execution Resource Broker** — a lightweight Bash-based GPU job launcher for small groups of Linux workstations and servers.

SLERB provides a practical subset of cluster-scheduler behavior without requiring a central daemon, database, or full SLURM installation. It discovers available NVIDIA GPUs over SSH, stages job inputs with `rsync`, protects allocations with atomic lock directories, launches foreground or detached jobs, queues work when resources are unavailable, and retrieves results to the submitting machine.

> Like SLURM, but with more duct tape.

## Features

- Probe CPU, memory, scratch space, and NVIDIA GPU availability across SSH-accessible nodes
- Submit work to a named node or automatically select a node with sufficient free GPUs
- Allocate a specific physical GPU or any requested number of GPUs
- Prevent conflicting SLERB allocations with atomic per-GPU lock directories
- Queue jobs automatically and launch them through a local background dispatcher
- Run job arrays with SGE-compatible task environment variables
- Detach long-running jobs and retrieve results later
- Cancel, restart, inspect, and fetch jobs by job ID
- Reserve an entire host or an individual GPU for non-SLERB use
- Keep local job history and queue state in simple TSV files
- Disable terminal colors with `NO_COLOR=1`

## How It Works

SLERB runs from a **controller machine** and communicates with each configured execution node over SSH.

1. The controller probes a node with SSH and `nvidia-smi`.
2. A free GPU is selected using utilization, memory, reservation, and lock state.
3. SLERB atomically creates one lock directory per allocated GPU.
4. A remote job directory is created under the node's SLERB scratch directory.
5. The input directory and run script are copied to the node with `rsync`.
6. The run script executes inside the staged work directory with GPU and array metadata exported as environment variables.
7. Logs, state, exit status, and staged files remain in the remote job directory until fetched or cleaned.

SLERB is intended for trusted users operating a small collection of Linux GPU machines. It is not a multi-tenant security boundary or a replacement for a production HPC scheduler.

## Requirements

### Controller machine

- Bash with `printf '%(...)T'` support, typically Bash 4.2 or newer
- OpenSSH client
- `rsync`
- Standard Unix tools including `awk`, `grep`, `sed`, `find`, `sort`, `xargs`, and `mktemp`
- Passwordless or agent-backed SSH access to every configured node

### Execution nodes

- Linux
- Bash
- OpenSSH server
- `rsync`
- NVIDIA drivers and `nvidia-smi`
- Standard Linux tools including `nproc`, `df`, `ps`, `find`, and `/proc`
- A writable scratch location for job data and lock directories

SSH is invoked in batch mode with an eight-second connection timeout. Interactive password prompts are not supported.

## Installation

Place the script somewhere on your `PATH`:

```bash
install -m 0755 slerb "$HOME/.local/bin/slerb"
```

Confirm that `~/.local/bin` is on your `PATH`:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

The first command that needs node configuration creates the default configuration file at:

```text
~/.slerb/nodes.tsv
```

## Quick Start

### 1. Configure nodes

Create or edit `~/.slerb/nodes.tsv`:

```text
# name        user    host                    env_file                    tags
jill          alice   gpu01.example.org       /home/alice/.slerb_env      gpu
explorer      bob     192.168.1.42            /home/bob/.slerb_env        gpu
localbox      -       localhost               /home/me/.slerb_env         local
```

The file is whitespace-separated and contains these fields:

| Field | Description |
|---|---|
| `name` | Short node name used by SLERB commands |
| `user` | SSH username; use `-` to use the current SSH default |
| `host` | Hostname or IP address |
| `env_file` | Environment file sourced on the remote node |
| `tags` | Optional single-field label shown by `slerb status` |

Use an absolute remote path for `env_file`.

Validate the file:

```bash
slerb validate
```

### 2. Configure each execution node

Create the environment file referenced by `nodes.tsv`, for example `/home/alice/.slerb_env`:

```bash
export SLERB_SCRATCH="/scratch/$USER/slerb"
export SLERB_JOBS_ROOT="$SLERB_SCRATCH/jobs"
export SLERB_LOCKS_ROOT="$SLERB_SCRATCH/locks"
export SLERB_WORK_ROOT="$SLERB_SCRATCH/work"

# Optional project environment setup:
# source "$HOME/miniconda3/etc/profile.d/conda.sh"
# conda activate training
```

Only `SLERB_SCRATCH` is required to change the default location. The remaining paths default beneath it.

### 3. Check node status

```bash
slerb status
```

Example status categories include:

- `READY` — at least one GPU is available
- `BUSY` — GPUs exist, but none are currently available
- `RESERVED` — the host has a SLERB reservation
- `NOGPU` — the node is reachable but has no visible NVIDIA GPU
- `DOWN` — the SSH probe failed

### 4. Create a run script

SLERB stages the contents of the input directory, copies the selected run script as `run.sh`, changes into the staged work directory, and executes it with Bash.

```bash
#!/usr/bin/env bash
set -euo pipefail

python train.py \
  --config configs/experiment.yaml \
  --output results
```

Write generated files beneath the current working directory so they are included when the remote job directory is fetched.

### 5. Submit a job

Run immediately on a named node:

```bash
slerb submit jill . ./submit.sh ./slerb-results
```

Select any node with a free GPU:

```bash
slerb submit --auto . ./submit.sh ./slerb-results
```

The command prints a job ID such as:

```text
slerb-20260710-142530-18421
```

Results are stored locally under:

```text
./slerb-results/<job_id>/
```

## Command Reference

### `status`

Probe every configured node and display node health, CPU load, memory usage, scratch capacity, lock count, and per-GPU state.

```bash
slerb status
```

A GPU is considered locally idle when all of the following are true:

- No SLERB host reservation applies
- No SLERB GPU lock exists
- GPU utilization is at most 5 percent
- GPU memory usage is at most 512 MiB

GPU states shown by the status command include `free`, `busy`, `locked`, and `reserved`.

### `validate`

Validate the configured node file or another supplied file:

```bash
slerb validate
slerb validate ./nodes.tsv
```

Each non-comment line must contain at least four whitespace-separated fields.

### `submit`

```text
slerb submit [options] <node|--auto> <input_dir> <run_script> <output_dir>
```

| Option | Description |
|---|---|
| `--keep` | Keep the remote job directory after a successful fetch |
| `--detach` | Launch immediately and return without waiting for completion |
| `--wait SECONDS` | Retry indefinitely at the specified interval instead of queueing when resources are unavailable |
| `--gpu ID[,ID...]` | Request specific physical GPU indexes on a named node |
| `--ngpu N` | Request any `N` free GPUs; defaults to `1` |
| `--array RANGE` | Submit one detached task per expanded array index |
| `--exclude NODES` | Exclude comma-separated nodes from automatic selection; may be repeated |

`--gpu` and `--ngpu` are mutually exclusive. Specific physical GPU selection cannot be combined with `--auto`.

#### Named node

```bash
slerb submit jill . ./submit.sh ./slerb-results
```

#### Automatic node selection

```bash
slerb submit --auto . ./submit.sh ./slerb-results
```

Automatic selection scores eligible nodes primarily by the number of free GPUs, then available CPU capacity, then scratch space.

#### Specific GPU

```bash
slerb submit --gpu 1 jill . ./submit.sh ./slerb-results
```

#### Specific multiple GPUs

```bash
slerb submit --gpu 0,1 jill . ./submit.sh ./slerb-results
```

#### Any number of free GPUs

```bash
slerb submit --ngpu 2 --auto . ./submit.sh ./slerb-results
```

#### Detached job

```bash
slerb submit --detach --auto . ./submit.sh ./slerb-results
```

#### Wait for resources instead of queueing

```bash
slerb submit --wait 30 --auto . ./submit.sh ./slerb-results
```

This retries every 30 seconds until the job launches or the command is interrupted.

#### Exclude nodes from automatic selection

```bash
slerb submit \
  --exclude atlantis,odyssey \
  --auto \
  . ./submit.sh ./slerb-results
```

### Job arrays

Submit a detached job for every task in a range:

```bash
slerb submit --array 1-15 --auto . ./submit.sh ./slerb-results
```

Supported range forms:

```text
1-15
1,3,7-9
1-20:2
10-1
```

Each array task receives its own job ID, remote directory, GPU allocation, log, and result directory.

Inside the run script, these variables identify the task:

```bash
echo "$SGE_TASK_ID"
echo "$TASK_ID"
echo "$SLERB_ARRAY_TASK_ID"
echo "$SLERB_ARRAY_RANGE"
echo "$SLERB_ARRAY_ID"
```

List array summaries:

```bash
slerb arrays
```

### `jobs`

Show recent local job history:

```bash
slerb jobs
slerb jobs 100
```

The output includes job ID, state, exit code, node, GPU allocation, array task, start time, and local result directory.

### `queue`

Display queued jobs:

```bash
slerb queue
```

Queued input is snapshotted when the job enters the queue. Later changes to the original input directory or run script do not affect the queued copy.

### `dispatch`

Run the local queue dispatcher:

```bash
slerb dispatch
slerb dispatch --interval 10
```

SLERB automatically starts a detached dispatcher when it queues a job. Dispatcher output is written to:

```text
~/.slerb/dispatch.log
```

The queue is strict FIFO: the dispatcher retries the job at the head of the queue until it launches, is canceled, or fails for a non-resource reason. A large request at the head of the queue can therefore delay smaller jobs behind it.

All queued jobs launch detached.

### `log`

Print a job's log:

```bash
slerb log <job_id>
```

SLERB reads the local fetched log first. If no local log exists, it attempts to read the remote `job.log` over SSH.

### `fetch`

Fetch one completed job:

```bash
slerb fetch <job_id>
```

SLERB does not fetch partial results from a running job.

Fetch and remove the remote job directory regardless of whether the job succeeded:

```bash
slerb fetch --clean <job_id>
```

Fetch every completed detached or running job known to the local history:

```bash
slerb fetch --all-done
```

By default:

- Successful jobs are cleaned from the remote node after fetching unless submitted with `--keep`
- Failed and canceled jobs remain remotely after fetching
- `fetch --clean` removes the remote directory after fetching any completed state

### `cancel`

Cancel a queued or remote job:

```bash
slerb cancel <job_id>
```

For queued jobs, SLERB removes the queue entry and its local spool snapshot. For remote jobs, it records exit code `130`, terminates matching processes, and releases the job's GPU locks.

Cancellation does not automatically fetch remote files.

### `restart`

Restart a job on its recorded node and physical GPU allocation:

```bash
slerb restart <job_id>
```

A restart always launches detached. Existing `job.log` and `exit_code` files are archived with a restart timestamp before relaunch.

If the remote directory or staged `run.sh` is missing, SLERB attempts to recreate the job from the original local input directory and run script recorded in job history.

Refuse to restart while matching processes are still running unless forced:

```bash
slerb restart --force <job_id>
```

A forced restart attempts graceful termination first, followed by forced termination if necessary.

### `reserve`

Reserve an entire host:

```bash
slerb reserve explorer --reason "maintenance"
```

Reserve one physical GPU:

```bash
slerb reserve explorer --gpu 1 --reason "local interactive work"
```

Reservations prevent new SLERB allocations. They do not terminate workloads that are already running.

### `release`

Release a host reservation:

```bash
slerb release explorer
```

Release a GPU reservation:

```bash
slerb release explorer --gpu 1
```

Only reservation locks can be released with this command; active job locks are not treated as reservations.

## Runtime Environment

Every job receives the following variables:

| Variable | Meaning |
|---|---|
| `JOB_ID` | SLERB job ID |
| `NSLOTS` | Compatibility value; currently set to `1` |
| `SLERB_JOB_ID` | SLERB job ID |
| `SLERB_REMOTE_DIR` | Remote job directory |
| `SLERB_WORK_DIR` | Remote staged work directory |
| `SLERB_GPU_ID` | First allocated physical GPU index |
| `SLERB_GPU_IDS` | Comma-separated allocated physical GPU indexes |
| `SLERB_GPU_PHYSICAL_ID` | First allocated physical GPU index |
| `SLERB_GPU_UUID` | First allocated GPU UUID |
| `SLERB_GPU_UUIDS` | Comma-separated allocated GPU UUIDs |
| `SLERB_GPU_COUNT` | Number of allocated GPUs |
| `SLERB_CUDA_DEVICE` | First logical CUDA device; currently `0` |
| `SLERB_CUDA_DEVICES` | Logical CUDA device indexes, such as `0,1` |
| `CUDA_VISIBLE_DEVICES` | Allocated GPU UUIDs, or physical indexes if UUIDs are unavailable |

For multi-GPU jobs, CUDA remaps the allocated devices into a zero-based logical sequence. For example, physical GPUs `2,3` are exposed to the job as logical CUDA devices `0,1`.

Array tasks additionally receive:

| Variable | Meaning |
|---|---|
| `SGE_TASK_ID` | Current array task index |
| `TASK_ID` | Current array task index |
| `SLERB_ARRAY_TASK_ID` | Current array task index |
| `SLERB_ARRAY_RANGE` | Original range expression |
| `SLERB_ARRAY_ID` | Parent array identifier |

The remote environment file is sourced before the job starts. SLERB then reapplies its runtime variables so the environment file cannot accidentally replace the assigned job and GPU metadata.

## Configuration

### Controller environment variables

| Variable | Default | Description |
|---|---|---|
| `SLERB_HOME` | `~/.slerb` | Local state directory |
| `SLERB_NODES_FILE` | `$SLERB_HOME/nodes.tsv` | Node inventory |
| `SLERB_JOBS_FILE` | `$SLERB_HOME/jobs.tsv` | Local job history |
| `SLERB_QUEUE_FILE` | `$SLERB_HOME/queue.tsv` | Queue index |
| `SLERB_QUEUE_DIR` | `$SLERB_HOME/queue` | Queued job snapshots |
| `SLERB_DISPATCH_LOCK_DIR` | `$SLERB_HOME/dispatch.lock` | Local dispatcher singleton lock |
| `SLERB_DISPATCH_INTERVAL` | `30` | Default dispatcher polling interval in seconds |
| `SLERB_ENV_FILE` | `~/.slerb_env` | Fallback remote environment-file value |
| `NO_COLOR` | unset | Disable ANSI colors when set |

Example isolated controller configuration:

```bash
export SLERB_HOME="$PWD/.slerb"
export SLERB_NODES_FILE="$PWD/nodes.tsv"
export SLERB_DISPATCH_INTERVAL=10
```

### Remote environment variables

| Variable | Default | Description |
|---|---|---|
| `SLERB_SCRATCH` | `$HOME/slerb` | Base remote SLERB directory |
| `SLERB_JOBS_ROOT` | `$SLERB_SCRATCH/jobs` | Remote job directories |
| `SLERB_LOCKS_ROOT` | `$SLERB_SCRATCH/locks` | Host, GPU, and job allocation locks |
| `SLERB_WORK_ROOT` | `$SLERB_SCRATCH/work` | Configurable work root created by SLERB |

Current jobs are staged under:

```text
$SLERB_JOBS_ROOT/<job_id>/work
```

## Local State Layout

By default, controller state is stored under `~/.slerb`:

```text
~/.slerb/
├── nodes.tsv
├── jobs.tsv
├── queue.tsv
├── queue/
│   └── <job_id>/
│       ├── input/
│       ├── request.env
│       └── run.sh
├── dispatch.lock/
│   └── pid
└── dispatch.log
```

The TSV and queue files are implementation state. Avoid editing them while SLERB commands or the dispatcher are active.

## Remote Job Layout

A staged job uses this layout:

```text
$SLERB_JOBS_ROOT/<job_id>/
├── .slerb_state
├── .slerb_launcher.sh       # detached jobs
├── exit_code                # written at completion or cancellation
├── job.log
├── pid
└── work/
    ├── run.sh
    └── ... staged input files and generated outputs
```

GPU locks are directories beneath `$SLERB_LOCKS_ROOT`:

```text
$SLERB_LOCKS_ROOT/
├── host.lock/
│   └── info
└── gpu-0.lock/
    └── info
```

Atomic `mkdir` operations provide allocation locking between cooperating SLERB processes.

## Input and Result Transfer

SLERB uses checksum-enabled, archive-mode `rsync` and follows symbolic links when staging input.

These paths are excluded from input staging and queue snapshots:

```text
.git/
.slerb/
slerb-results/
```

The complete remote job directory is fetched into the local result directory, including `job.log`, `.slerb_state`, `exit_code`, and the staged `work/` tree.

## Job States

Common local history states include:

| State | Meaning |
|---|---|
| `QUEUED` | Waiting in the local queue |
| `DISPATCHING` | Dispatcher is attempting launch |
| `STAGED` | Remote directory and GPU allocation created |
| `DETACHED` | Launched in the background |
| `RUNNING` | Confirmed incomplete during a fetch attempt |
| `DONE` | Completed with exit code `0` |
| `FAILED` | Completed with a nonzero exit code or failed during dispatch |
| `CANCELED` | Canceled locally or remotely; exit code `130` |
| `MISSING` | Recorded remote job directory no longer exists |

## Operational Notes

- SLERB schedules NVIDIA GPU jobs only; it does not currently submit CPU-only jobs.
- GPU availability combines SLERB locks with instantaneous `nvidia-smi` utilization and memory readings.
- Locks coordinate SLERB instances that share the same remote lock root. External workloads are detected heuristically through GPU utilization and memory use.
- Host and GPU reservations block new SLERB allocation but do not stop existing processes.
- Local queue state and history are specific to the controller's configured `SLERB_HOME`.
- Running SLERB from multiple controller machines against the same nodes is supported at the GPU-lock level, but each controller maintains an independent queue and job history.
- The remote environment file and queued `request.env` files are sourced as shell code. Use SLERB only with trusted configuration and state files.
- Foreground submissions wait for completion, fetch results automatically, and return the remote job's exit code.
- Detached and queued submissions require a later `fetch` or `fetch --all-done` to copy results locally.

## Troubleshooting

### A node appears as `DOWN`

Verify non-interactive SSH access:

```bash
ssh -o BatchMode=yes user@host true
```

Then verify that Bash and the configured remote environment file are available.

### A node appears as `NOGPU`

Run this on the node:

```bash
nvidia-smi
```

The command must be available after the node's environment file is sourced.

### A GPU remains `locked`

Inspect the corresponding lock metadata on the node:

```bash
cat "$SLERB_LOCKS_ROOT/gpu-0.lock/info"
```

Use `slerb cancel <job_id>` for an active SLERB job or `slerb release <node> --gpu 0` for a reservation. The release command intentionally refuses to remove active job locks.

### A detached job is not visible locally

Check local history and then inspect its remote log:

```bash
slerb jobs 100
slerb log <job_id>
```

Fetch completed detached jobs with:

```bash
slerb fetch --all-done
```

### Queued work is not launching

Inspect the queue and dispatcher log:

```bash
slerb queue
tail -f "$HOME/.slerb/dispatch.log"
```

You can also run a dispatcher in the foreground:

```bash
slerb dispatch --interval 10
```

### A job cannot be restarted

A restart requires the original node and recorded physical GPUs to be available. If restaging is necessary, the original local input directory and run script must still exist at the paths recorded during submission.

## Scope and Limitations

SLERB intentionally favors transparency and portability over scheduler complexity. It does not provide:

- Fair-share or priority scheduling
- Backfilling around an unlaunchable queue head
- Resource accounting or usage quotas
- Dependencies between jobs
- Distributed queue coordination between controllers
- CPU, RAM, or scratch reservations per job
- Container orchestration
- Multi-node jobs
- User isolation or privilege separation
- A web interface or central service

For a trusted lab, workstation pool, or small GPU group, these constraints keep deployment simple: one Bash script, SSH access, and a shared understanding of the configured nodes.

## License

No license is specified in the provided source.

