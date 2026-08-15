# Run GitHub CI in RISCOS 

![Test](https://github.com/vmactions/riscos-vm/workflows/Test/badge.svg)



See all the supported VMs: [VMActions.org](https://vmactions.org)

Powered by [AnyVM.org](https://anyvm.org)

## :robot: AI Ready

> [!TIP]
> **You don't need to write this workflow by hand.**
>
> These VMs are now AI-ready. With the **[vmactions-ci skill](https://github.com/vmactions/vmactions-skill)**, an AI coding agent -- Claude Code, Codex, Copilot CLI, Gemini CLI, and others -- understands the full vmactions interface and writes the GitHub Actions CI for you, **automatically**.
>
> Just describe what you want in plain language, e.g. *"run my tests on RISCOS"* or *"check that my project builds on RISCOS aarch64"*, and the agent generates a correct, ready-to-commit `test.yml`. It will:
>
> - pick the right action, `release`, and `arch` for your target;
> - install your toolchain and dependencies in the `prepare` step;
> - forward your secrets and environment variables into the VM;
> - sync your source code in and back out; and
> - steer around the common footguns -- the per-OS default shell, the `riscv64` sync method, keeping `runs-on: ubuntu-latest` even for other arches, pinning the action version, and more.
>
> No need to memorize releases, architectures, package managers, or shells -- the agent handles it. Install the skill once and just ask.
>
> ### >> [Get the vmactions-ci skill](https://github.com/vmactions/vmactions-skill) <<

Use this action to run your CI in RISCOS.

The github workflow only supports Ubuntu, Windows and MacOS. But what if you need to use RISCOS?


All the supported releases are here:



| Release | armv7 (ARM 32-bit, Cortex-A7) |
|---------|---------|
| 5.30 | ✅ (tar) |

<!-- arch-label: armv7 = armv7 (ARM 32-bit, Cortex-A7) -->

> **Note:** RISC OS support is a **tech preview**. Remote command execution
> and file sync both work, and the desktop comes up and is visible on the VNC
> console. There is **no keyboard** -- see below. Pointer input is untested:
> the ROM does carry a `usbmouse` driver, but no USB mouse has been attached
> to this guest, so nothing here claims it works.

> **Nothing from RISC OS Open is redistributed here.** The builder downloads
> the official Raspberry Pi SD card image from `riscosopen.org` at build time;
> this repository's release assets carry only its own work (the patched QEMU,
> the agent, the injector). Note that ROOL deletes superseded media -- the
> 5.30 zip answers 200 while the identically-shaped 5.28 URL answers 404 --
> so a new release **replaces** this row rather than adding one.

> **Linux x86_64 hosts only.** The patched QEMU below is published for that
> platform alone, and there is no system fallback -- no released QEMU can boot
> RISC OS on a raspi machine at all. `anyvm.py` fails fast with that message on
> any other host rather than starting an emulator that cannot work. macOS and
> Windows would need the same patched build produced on those runners; that is
> not done yet.

> **No working RISC OS port of QEMU existed, so this builder makes one.**
> `files/qemu-riscos-raspi.patch` is eight fixes plus a new USB NIC model,
> built by `files/build-qemu-riscos.sh` and published as this repo's own
> release asset. Four of the eight are **generic QEMU defects** with nothing
> to do with RISC OS: three in `hcd-dwc2.c` (the frame counter divided
> elapsed time by the frame interval in PHY clocks rather than the frame
> time; `GINTSTS_HCHINT` was raised but never lowered; the bus was started
> from `dwc2_attach()` instead of on the HPRT0 port-enable edge), and one in
> `bcm2835_dma.c`, where `xlen -= 4` on a `uint32_t` byte count wraps for any
> length that is not a multiple of four -- the loop then spins about 2^30
> times, scribbling over guest memory as it goes. The RISC OS specific ones
> are a BCM2835 mailbox power channel, a VCHIQ mailbox peer, byte access on
> the interrupt controller and the BSC/I2C FIFO (RISC OS uses `LDRB`/`STRB`
> where the models demanded word access), and `hw/usb/dev-smsc95xx.c`, a
> model of the SMSC LAN9512 that is the real Pi 2's NIC -- the guest has a
> driver for it and for nothing else QEMU offers.

> **There is no keyboard, and that is a RISC OS limitation.** The BCM2835 ROM
> ships no USB keyboard driver at all: its USB tree has
> `USBDriver/build/c/usbmouse` and nothing for keyboards, the boot prints
> `No keyboard present - autobooting`, and a whole boot's worth of USB traffic
> is `ep0` control transfers with zero interrupt-endpoint packets. Every HID
> device is reported as `[error &24425355]`, and `&24425355` is ASCII `"USB$"`
> -- USBDriver's "attached, no driver" tag. So there is no console to type an
> installer into, and the guest is prepared by patching the disc image offline
> instead (`files/rofilecore.py` parses FileCore's `SBPr` "BigDir" directories
> and overwrites one cosmetic boot script in place, at its existing length).
>
> Three things about that image will bite anyone who touches it. The live boot
> tree **does not exist** in a freshly downloaded image -- RISC OS copies
> `$.!Boot.RO530Hook.Boot` into `$.!Boot.Choices.Boot` on first boot, so
> patching the pristine image patches a template that is never run, and
> patching a booted one lands somewhere else entirely. Every injected command
> must be prefixed with `X` (`*X <cmd>` runs it and discards any error),
> because one error stops the boot on a dialog waiting for a keypress that
> can never come. And injected scripts must print **nothing**: any console
> output from a Tasks hook opens the Wimp's single-tasking output screen,
> which waits for a keypress at the end -- same brick, different cause.

> **The agent.** RISC OS ships **no remote-access server of any kind** --
> checked in a running guest, where `*Modules` lists 137 modules whose only
> networking is the stack (`Internet`, `Resolver`, `DHCP`, `EtherUSB`) and
> clients (`LanManFS` for SMB, `ShareFS`/`Freeway` for Acorn Access). No
> telnetd, no sshd, no ftpd. So, like reactos-builder, this builder supplies
> the server: `files/anyvmd.py`, a small Python agent on port 23
> (`VM_TRANSPORT=telnet`).
>
> The telnet is real, not a raw socket wearing the name: IAC is unescaped
> inbound and doubled outbound, and BINARY (RFC 856) is accepted both ways,
> which is what keeps the tar stream intact. Verified against the worst case
> it can meet -- 4096 bytes of `0xFF` (the IAC byte itself) followed by all
> 256 byte values, round-tripped byte-identical. It is not a *shell*, though:
> RISC OS has no pipes to a child and no `&&`, so the agent parses the two
> tar one-liners itself, services them with Python's `tarfile`, and prints
> the completion marker where a shell would have got it from the chained
> `&& echo`. It also holds a persistent session, several commands down one
> connection, because that is what `telnet_exec` does.
>
> It needs nothing installed -- ROOL's image already ships **Python 2.7.2** at
> `$.Programming.Python.!Python27` with a complete standard library. It runs
> inside a `TaskWindow` so the desktop stays interactive; single-tasking
> Python would freeze the machine for as long as it served, and `*Shutdown`
> would then do nothing at all.
>
> Sync was settled on the running guest rather than assumed. There is no sshd
> and no ssh client (so no rsync / sshfs / scp) and no 9P client. `Fat32FS`
> (1.63) *is* loaded and could in principle carry files on the FAT boot
> partition, but it serves removable media and cannot see the SD card's own
> boot partition -- `::0`, `::4` and `::PiBoot` all resolve to nothing, and
> `Fat32Map`, despite the name, is a DOS-extension-to-filetype table rather
> than a disc mapping.





## 1. Example: `test.yml`:

```yml

name: Test

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    name: A job to run test in RISCOS
    env:
      MYTOKEN : ${{ secrets.MYTOKEN }}
      MYTOKEN2: "value2"
    steps:
    - uses: actions/checkout@v6
    - name: Test in RISCOS
      id: test
      uses: vmactions/riscos-vm@v0
      with:
        envs: 'MYTOKEN MYTOKEN2'
        usesh: true
        prepare: |
          Echo anyvm

        run: |
          Echo anyvm
          Cat $.work





```


The latest major version is: `v0`, which is the most recommended to use. (You can also use the latest full version: `v0.0.0`)  


If you are migrating from the previous `v0`, please change the `runs-on: ` to `runs-on: ubuntu-latest`


The `envs: 'MYTOKEN MYTOKEN2'` is the env names that you want to pass into the vm.

The `run: xxxxx`  is the command you want to run in the vm.

The env variables are all copied into the VM, and the source code and directory are all synchronized into the VM.

The working dir for `run` in the VM is the same as in the Host machine.

All the source code tree in the Host machine are mounted into the VM.

All the `GITHUB_*` as well as `CI=true` env variables are passed into the VM.

So, you will have the same directory and same default env variables when you `run` the CI script.





## 2. Share code

The code is shared from the host to the VM via `rsync` by default, you can choose to use `sshfs` or `nfs` or `scp` to share code instead.


```yaml

...

    - name: Test
      id: test
      uses: vmactions/riscos-vm@v0
      with:
        sync: sshfs  # or: nfs


...


```

You can also set `sync: no`, so the files will not be synced to the  VM.


When using `rsync` or `scp`,  you can define `copyback: false` to not copy files back from the VM in to the host.


```yaml

...

    - name: Test
      id: test
      uses: vmactions/riscos-vm@v0
      with:
        sync: rsync
        copyback: false


...


```





## 3. NAT from host runner to the VM

You can add NAT port between the host and the VM.

```yaml
...
    - name: Test
      id: test
      uses: vmactions/riscos-vm@v0
      with:
        nat: |
          "8080": "80"
          "8443": "443"
          udp:"8081": "80"
...
```


## 4. Set memory and cpu

The default memory of the VM is 6144MB, you can use `mem` option to set the memory size:

```yaml

...
    - name: Test
      id: test
      uses: vmactions/riscos-vm@v0
      with:
        mem: 4096
...
```


The VM is using all the cpu cores of the host by default, you can use `cpu` option to change the cpu cores:

```yaml

...
    - name: Test
      id: test
      uses: vmactions/riscos-vm@v0
      with:
        cpu: 3
...
```


## 5. Select release

It uses [the RISCOS 5.30](conf/default.release.conf) by default, you can use `release` option to use another version of RISCOS:

```yaml
...
    - name: Test
      id: test
      uses: vmactions/riscos-vm@v0
      with:
        release: "5.30"
...
```

You can also give only the leading, `.` separated part of a release. The newest release that starts with it is used, so the workflow does not have to be edited for every point release:

```yaml
...
    - name: Test
      id: test
      uses: vmactions/riscos-vm@v0
      with:
        release: "5"
...
```

Here `release: "5"` runs the newest `5.x` release of RISCOS. Each part you give has to match in full, so a release that does not exist fails the job instead of quietly falling back to another one.

## 6. Select architecture

The vm is using x86_64(AMD64) by default, but you can use `arch` option to change the architecture:

```yaml
...
    - name: Test
      id: test
      uses: vmactions/riscos-vm@v0
      with:
        arch: aarch64
...
```

When you run with `aarch64`, the host runner should still be the normal `x86_64` runner: `runs-on: ubuntu-latest`

It's not recommended to use `ubuntu-24.04-arm` as runner, it's much more slower.



## 7. Custom shell

Support custom shell:

```yaml
...
    steps:
    - uses: actions/checkout@v6
    - name: Start VM
      id: vm
      uses: vmactions/riscos-vm@v0
      with:
        sync: nfs
    - name: Custom shell step 1
      shell: riscos {0}
      run: |
        pwd
        echo "this is step 1, running inside the VM"
    - name: Custom shell step 2
      shell: riscos {0}
      run: |
        pwd
        echo "this is step 2, running inside the VM"
...
```

The custom shell will automatically `cd` into `$GITHUB_WORKSPACE` if it exists before running your commands.

How file changes propagate between the host and the VM depends on the `sync` method:

- `sync: nfs` or `sync: sshfs`: the workspace is a live mount, so file changes are visible on both sides immediately.
- `sync: rsync` or `sync: scp`: the wrapper syncs the workspace to the VM before each custom shell step and syncs it back afterwards, so files created in the VM are available to later host steps (and vice versa). `rsync` transfers are incremental; `scp` copies the whole workspace each time, which can be slow for large workspaces.

You can also use `custom-shell-name` to set a custom name for the shell wrapper:

```yaml
...
    steps:
    - uses: actions/checkout@v6
    - name: Start VM
      id: vm
      uses: vmactions/riscos-vm@v0
      with:
        sync: nfs
        custom-shell-name: vmsh
    - name: Custom shell step 1
      shell: vmsh {0}
      run: |
        pwd
        echo "this is step 1, running inside the VM"
    - name: Custom shell step 2
      shell: vmsh {0}
      run: |
        pwd
        echo "this is step 2, running inside the VM"
...
```


## 8. Synchronize VM time

If the time in VM is not correct, You can use `sync-time` option to synchronize the VM time with NTP:

```yaml
...
    - name: Test
      id: test
      uses: vmactions/riscos-vm@v0
      with:
        sync-time: true
...
```


## 9. Disable cache

By default, the action caches `apt` packages on the host and VM images/artifacts. You can use the `disable-cache` option to disable this:

```yml
...
    - name: Test
      id: test
      uses: vmactions/riscos-vm@v0
      with:
        disable-cache: true
...
```


## 10. Cache the VM image after prepare

The `prepare` step (installing packages etc.) normally runs on every build. With `cache-after-prepare: true`, the action shuts the VM down cleanly after `prepare` has finished, caches the prepared VM image, and boots the VM again before `run`. Later runs with the same `prepare` script restore the prepared image, skip `prepare` entirely, and start directly at `run`:

```yml
...
    - name: Test
      id: test
      uses: vmactions/riscos-vm@v0
      with:
        cache-after-prepare: true
        prepare: |
          Echo anyvm
        run: |
          ...
...
```

Notes:

- The cache key includes a hash of the `prepare` script and the `sync` method, so changing either of them rebuilds the prepared image from the base image.
- The source tree is still synchronized into the VM on every run; only the `prepare` step is skipped.
- The first run (or any run after `prepare` changes) takes longer: the VM is shut down after `prepare`, the prepared image is cached, and the VM boots again before `run`.
- The action output `cache-after-prepare-hit` is `true` when a prepared image was restored and `prepare` was skipped.
- The option is ignored when `disable-cache: true` is set or when `prepare` is empty.


## 11. Debug on error

If you want to debug the VM when the `prepare` or `run` step fails, you can set `debug-on-error: true`.

When a failure occurs, the action will enable a remote VNC link and wait for your interaction. You can then access the VM via VNC to debug. To continue or finish the action, you can run `touch ~/continue` inside the VM.

[First create a variable `DEBUG_ON_ERROR` with value being "true"](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-variables),

Then use it in the workflow:

```yaml
...
    - name: Test
      id: test
      uses: vmactions/riscos-vm@v0
      with:
        debug-on-error: ${{ vars.DEBUG_ON_ERROR }}

...
```

You can also set the `vnc-password` parameter to set a custom password to protect the VNC link:

```yaml
...
    - name: Test
      id: test
      uses: vmactions/riscos-vm@v0
      with:
        debug-on-error: ${{ vars.DEBUG_ON_ERROR }}
        vnc-password: ${{ secrets.VNC_PASSWORD }}

...
```

You will be asked to input the username and password when you access the VNC link. The username can be any string, the password is the value of the `vnc-password` parameter.


See more: [debug on error](https://github.com/vmactions/.github/wiki/debug%E2%80%90on%E2%80%90error)



# Under the hood

We use Qemu to run the RISCOS VM.




# Upcoming features:

1. Support MacOS runner and Windows runner.















