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
| `--run` | exit 3 after the title's `int3`; 4 minute limit, not reached |

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

Hypothesis, not confirmed: the `int3` is the title's check on the command list that slot 73 returned, and the warning
names its cause, a console command-list proxy that cannot resolve the allocator proxy because the allocator came
from the PC slot 9. This is the area of stop 9 in [retail_boot_chain.md](retail_boot_chain.md), whose fix there is
the list proxy of #2490 plus local routing. It was not tested here.

## 3. What this does not show

One run, no repeat, no debugger. Whether `LEONIDA_SKIP_INT3=1` goes further was not tried. The Vulkan device the
layer used was not recorded, so an AMD or NVIDIA difference is not excluded.

## 4. Reproduce

The commands of [boot.md](boot.md) section 2, with `--overlay` and an empty `--trace` name. Keep the traces out of the
repository: they hold addresses of the executable.
