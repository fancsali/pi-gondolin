# pi-gondolin

A [pi coding agent](https://github.com/badlogic/pi-mono) extension that runs all tool calls (bash, read, write, edit) inside a [Gondolin](https://github.com/earendil-works/gondolin) micro-VM sandbox instead of directly on the host.

Your working directory is mounted read-write at `/workspace` inside the VM. The LLM sees `/workspace` as its working directory. Commands that could harm your host system run in an isolated environment — network access, filesystem, and processes are all contained.

## Requirements

- [pi coding agent](https://github.com/badlogic/pi-mono) (`npx @mariozechner/pi-coding-agent`)
- QEMU:
  - macOS: `brew install qemu`
  - Linux (x86_64): `sudo apt install qemu-system-x86`
  - Linux (aarch64): `sudo apt install qemu-system-arm`
- Node.js 18+

## Install

```sh
pi install git:github.com/fancsali/pi-gondolin
```

## Usage

Pi will load the extension automatically. On first run, gondolin downloads a guest image (~200 MB) into `~/.cache/gondolin/`.

## What it does

- **Overrides `bash`** — commands run via `/bin/bash -lc` inside the VM
- **Overrides `read`** — files are read from the VM's `/workspace` tree
- **Overrides `write` / `edit`** — writes go into the VM, synced to the host mount
- **Routes `!` commands** — user shell commands (`!ls`, `!!git diff`) also run in the VM
- **Patches the system prompt** — tells the LLM its cwd is `/workspace`
- **Lazy + eager start** — VM starts on `session_start`; any tool call also triggers it if needed
- **Clean shutdown** — VM is closed when the pi session ends

## Configuration

VM settings can be customized via JSON config files. Project-scoped config **replaces** user-scoped config (no merge between them). All provided options are passed directly to `VM.create()`.

### Config file locations

| Scope | Path |
|-------|------|
| Project | `<project>/<CONFIG_DIR_NAME>/gondolin.json` (typically `.pi/gondolin.json`) |
| User | `<getAgentDir()>/gondolin.json` (typically `~/.pi/agent/gondolin.json`) |

Paths are resolved via Pi's SDK (`CONFIG_DIR_NAME` and `getAgentDir()`), so they work correctly with Pi forks that use different config directory names.

**NOTE**: Only `ENOENT` (file not found) is treated as "config not present"; all other errors are reported to the user.

### Examples
#### Increase VM memory

```json
{
  "memory": "2G",
}
```

#### Increase rootfs disk size

```json
{
  "rootfs": {
    "size": "20G"
  }
}
```

### Supported options

Any option accepted by Gondolin's `VM.create()` -- commonly:
- `memory`
- `cpus`
- `rootfs.size`
- `vfs` (VfsOptions)

**NOTE**: The `vfs.mounts["/workspace"]` property is always ignored. The workspace mount (`/workspace` → project root) is mandatory and cannot be overridden. Other `vfs.mounts` entries and `vfs` settings are preserved and passed through.

## How it works

Gondolin boots a lightweight QEMU micro-VM in under a second. The `RealFSProvider` mounts your local working directory into the VM via a FUSE-backed virtual filesystem, so reads and writes are bidirectional. All `exec` calls go through gondolin's `vm.exec()` API with streamed output piped back to pi's tool result renderer.

The wiring follows the pattern from the [gondolin wiring gist](https://gist.github.com/ggoodman/6c56e13ca097e0b89f7cf0f9214c8f30) and the [gondolin pi example](https://github.com/earendil-works/gondolin/blob/main/host/examples/pi-gondolin.ts), adapted as a standalone loadable pi extension.

## License

MIT
