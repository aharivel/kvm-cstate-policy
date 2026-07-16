# KVM_CAP_CSTATE_POLICY — Implementation Notes

Based on Linux v7.2-rc3 source at `/home/aharivel/work/linux/`.

## 1. KVM_CAP_HALT_POLL — Template Pattern

5 touch points to replicate:

| Step | File | Location |
|------|------|----------|
| Define cap | `include/uapi/linux/kvm.h:932` | `#define KVM_CAP_HALT_POLL 182` |
| Check extension | `virt/kvm/kvm_main.c:4882` | `case KVM_CAP_HALT_POLL: return 1;` |
| Enable (set) | `virt/kvm/kvm_main.c:5042-5058` | Validate args, store in `struct kvm` |
| Per-VM storage | `include/linux/kvm_host.h:864-865` | `bool override_halt_poll_ns; unsigned int max_halt_poll_ns;` |
| Consumption | `virt/kvm/kvm_main.c:3697-3723` | `kvm_vcpu_max_halt_poll_ns()` called from `kvm_vcpu_halt()` |

Idiom: `override_*` bool + `smp_wmb/smp_rmb` pair for safe per-VM override of global module parameter.

## 2. Current MWAIT/HLT Exit Handling

### MWAIT — emulated as NOP

- VMX exit table: `arch/x86/kvm/vmx/vmx.c:6356` → `kvm_emulate_mwait`
- `kvm_emulate_mwait()`: `arch/x86/kvm/x86.c:2235` — calls `kvm_emulate_monitor_mwait()` which **skips the instruction** (NOP). Does NOT enter any C-state.
- With `cpu-pm=on`: `MWAIT_EXITING=0` in VMCS → guest MWAIT executes natively on hardware, no exit at all.

### HLT — always intercepted

- `kvm_emulate_halt()`: `arch/x86/kvm/x86.c:11769` → sets `KVM_MP_STATE_HALTED`
- Run loop calls `kvm_vcpu_halt()`: `virt/kvm/kvm_main.c:3721`
- HLT always causes VM exit regardless of cpu-pm setting.

### VMCS controls for MWAIT

- `arch/x86/kvm/vmx/vmx.c:4658-4660`:
  ```c
  if (kvm_mwait_in_guest(vmx->vcpu.kvm))
      exec_control &= ~(CPU_BASED_MWAIT_EXITING | CPU_BASED_MONITOR_EXITING);
  ```
- `kvm_mwait_in_guest()`: `arch/x86/kvm/x86.h:505-508` — returns `kvm->arch.disabled_exits & KVM_X86_DISABLE_EXITS_MWAIT`
- Set by QEMU via `KVM_CAP_X86_DISABLE_EXITS` ioctl

## 3. kvm_vcpu_halt() Call Chain

```
kvm_vcpu_halt()                     virt/kvm/kvm_main.c:3721
  ├── halt-poll loop (cpu_relax)    lines 3736-3745, up to halt_poll_ns
  └── kvm_vcpu_block()              line 3747
        └── schedule()              line 3662 (TASK_INTERRUPTIBLE)
              └── do_idle()         kernel/sched/idle.c:278
                    └── cpuidle_idle_call()    idle.c:181
                          ├── cpuidle_select()     cpuidle.c:359 → governor->select()
                          └── cpuidle_enter()      cpuidle.c:375 → state->enter()
```

## 4. cpuidle C-state Selection — Where to Enforce

### The `disable` mechanism

- `struct cpuidle_state_usage` at `include/linux/cpuidle.h:36`:
  ```c
  unsigned long long disable;  // bitmask: BIT(0)=user, BIT(1)=driver
  ```
- Per-CPU array: `struct cpuidle_device.states_usage[CPUIDLE_STATE_MAX]`
- Checked by **all governors** (menu.c:308, teo.c:292, ladder.c:89) — disabled states are skipped
- Also checked in `cpuidle_find_deepest_state()` at cpuidle.c:96

Disable bits:
- `CPUIDLE_STATE_DISABLED_BY_USER` = BIT(0) — via sysfs
- `CPUIDLE_STATE_DISABLED_BY_DRIVER` = BIT(1) — driver sets at registration

### `forced_idle_latency_limit_ns`

- `struct cpuidle_device` field at `include/linux/cpuidle.h:103` — per-CPU latency constraint
- Checked in `cpuidle_idle_call()` at idle.c:213-225
- Used to cap states by exit latency rather than by state index

### `intel_idle.max_cstate` — boot-time only

- `drivers/idle/intel_idle.c:72`: `static int max_cstate = CPUIDLE_STATE_MAX - 1;`
- Enforced at driver init via `intel_idle_max_cstate_reached()` (line 1743)
- States beyond max_cstate are never registered — **not runtime changeable**

### No existing per-task C-state cap

No mechanism to cap C-states per kernel task (thread). `states_usage.disable` is per-CPU, `forced_idle_latency_limit_ns` is per-CPU. Nothing is per-task.

## 5. Enforcement Strategy Options

### Option A: Temporarily disable cpuidle states per-CPU

Before `kvm_vcpu_block()` → `schedule()`:
1. Get the pCPU the vCPU is pinned to
2. Set `dev->states_usage[i].disable |= CPUIDLE_STATE_DISABLED_BY_DRIVER` for states > max_cstate
3. After wakeup, restore

**Problem:** per-CPU, not per-task. If multiple VMs share a pCPU (overcommit), they'd fight over the disable flags. Only safe with dedicated pinning (which is the NFV use case anyway).

### Option B: Use `forced_idle_latency_limit_ns`

Set `dev->forced_idle_latency_limit_ns` to the exit latency of max_cstate before blocking.

**Problem:** same per-CPU issue. Also, latency-based rather than state-index-based — less precise.

### Option C: New cpuidle API for per-task constraint

Add a per-task or per-kvm annotation that the cpuidle governor checks during `select()`. Cleanest but requires cpuidle subsystem changes — bigger patch, more maintainers to convince.

### Option D: Custom idle function for KVM

Instead of calling `schedule()` → generic idle path, KVM could use its own idle implementation (MONITOR+MWAIT with capped hint) when a policy is active. But this has the scheduler/RCU problems identified in RFC review.

## 6. Recommended Approach

**Option A for v1** — temporarily disable cpuidle states. Scoped to dedicated-pinning use case (NFV). Document that overcommit with mixed policies is unsupported.

If upstream wants broader support, **Option C** (new cpuidle API) is the clean path but larger scope.
