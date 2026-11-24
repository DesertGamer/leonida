# The retail boot on plain `main`, RTX 5070: where `gta6rt --run` stops

**Status: one run on one machine, 2026-11-24. A measurement, not a claim on any code item.** It continues
[retail_boot_chain.md](retail_boot_chain.md) for a tree with none of the pull requests that page merged locally.
Nothing here quotes the executable: no addresses, offsets or bytes; only slot numbers, call shapes, counts and the
runtime's own log classes.

Setup: `main` `21625bca8`, nothing merged on top. AMD Ryzen 7 7800X3D, 32 GB, Windows 11 build 26200, NVIDIA
GeForce RTX 5070 (Windows driver version 32.0.16.1714; the machine also has an AMD integrated GPU, and which
Vulkan device the layer picked was not recorded), Vulkan SDK 1.4.363.0, `llvm-mc` 23.1.6 from conda-forge, Visual
Studio 2026 (v18) with CMake 4.4 for `RelWithDebInfo`. Own 1.0.2.1 copy, `LEONIDA_D3D12=vulkan`, `--overlay`
outside the package, no other switches.

## 1. What holds

| Pass | Result |
|---|---|
| `make_game.py --stages check,shaders,runtime` | built; 26,503 shaders translated (2,236 VS, 12,101 PS, 2,689 CS, 9,282 mesh, 195 geometry); the runtime built with Visual Studio 2026 |
| `--dry-run` | exit 0, no exception |
| `--init` | exit 0; 4 `DllMain` ok, 0 failed; the loader reports 53,246 index-0 TLS reads redirected to the module's own TLS (the stop 4 shape of [retail_boot_chain.md](retail_boot_chain.md) does not occur) |
| `--run` | ends on the title's `int3` (the shell reported 3); the 4 minute limit is not reached |
| `--run`, `LEONIDA_SKIP_INT3=1` | the `int3` is stepped over once, then a read from address 0 in the title (the shell reported 139) |

The `--run` startup trace has 21 events and no bad line: 5 `map`, 2 `tls_callback`, 4 `dllmain`, 5
`xmem_attributes`, 1 `xmem_physical_pages` (attributes `0x280`, 11,360 pages), 3 `exception`. The log has no
unimplemented import and no `[rt] cpu:` line.

## 2. The stop

The D3D12 trace records one call, `D3D12XboxCreateDevice ok`. The runtime then logs the first call of device slots
0, 1, 2, 26 (`GetCustomHeapProperties`), 8 (`CreateCommandQueue`), 36 (`CreateFence`), 48 (routed to
`CreateCommandQueueX`, queue flags `0x1`), 9 (`CreateCommandAllocator`) and 73 (routed to `CreateCommandListX`,
node mask 0, type 0), plus slots 0 and 1 of the first queue and the first fence. Two things belong to the last step:

- the runtime logs `allocators of this device are not wrapped: its command lists cannot resolve an allocator proxy`
  before the first slot 73 call;
- the title's `int3` (`0x80000003`) follows the slot 73 call, with no further D3D12 call.

The three exceptions in the trace are the `int3` and two C++ throws whose frames are all in the host's `d3d12.dll`
and `D3D12Core.dll`, not in the title.

This is stop 9 of [retail_boot_chain.md](retail_boot_chain.md). Reading `proxy_create_command_list_x` in
`runtime/shims/d3d12_xs_device.cpp` on this tree, slot 73 hands the title's `riid` to the PC
`ID3D12Device::CreateCommandList`, clears the output pointer first and returns the failure when the PC device does
not know the interface. Open pull requests #601 and #359 describe the title's `riid` as a console command-list IID
the PC device answers with `E_NOINTERFACE`, so the title's output pointer stays null and the `int3` is its check on
it. The allocator warning above is not that cause: it is printed once by device slot 9 and only says that the
allocator is passed on unwrapped. The `riid` and the returned `HRESULT` are not in this run's log, and neither pull
request was applied here, so the link between them and this stop is from the code and their descriptions, not from a
run.

With `LEONIDA_SKIP_INT3=1` the `int3` is stepped over once (`int3_skipped` 1) and the run then ends on a read from
address 0 a few instructions later in the same function of the title (the shell reported 139). The D3D12 trace is the same
single call and the log ends at the same slot 73 line. That is the "reads through a null list" of stop 9.

## 3. What this does not show

One run of each kind, no repeat, no debugger. Neither #601 nor #359 was applied, so whether they take this tree past
stop 9 was not tried. The Vulkan device the layer used was not recorded, so an AMD or NVIDIA difference is not
excluded.

## 4. Reproduce

The commands of [boot.md](boot.md) section 2, with `--overlay` and an empty `--trace` name. Keep the traces out of the
repository: they hold addresses of the executable.
