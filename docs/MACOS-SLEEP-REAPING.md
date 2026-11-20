# macOS: sleep and child processes

## How it started

On a Mac laptop, strange timeouts showed up whenever the computer had been asleep. The hub restarted itself overnight with "stopped answering" or "start stalled for 1054 s", the guard reported health checks as `hung`, and design-history captures timed out after 120 s. Nothing like this happens on Linux or Windows.

Looking into it turned up two separate faults, both specific to macOS. This branch fixes both.

## 1. A sleeping Mac looked like a hung hub

The hub checks its own parts (writer, model, agents) every 10 s. It stored the time of each part's last good check by the wall clock, while the checks run on tokio's timers, whose clock stops while the machine sleeps.

A Mac with Power Nap wakes for about 2 s roughly every 15 minutes (a "DarkWake"). The guard probes `/healthz` during that wake, sees every part's last check minutes or hours old, and counts a failure. The wake is too short for the next check to run. Six wakes in a row ended a hub that was well ("stopped answering"). A start under way was killed as "start stalled" after the Mac had simply been asleep, and the build was marked bad.

**Fix**
- `native/host/src/guard.rs`: the last good check is kept on the monotonic clock (`mono_s`), the same clock as the timers.
- `native/guard/src/run.rs`: the start stall is measured on `Instant` from when the `phase` file was last seen changing.
- `native/guard/src/plan.rs` (`SleepClock`): a sleep shows up as the wall clock running more than 10 s past the monotonic one. After a sleep the miss count goes back to zero, and a failed check within 30 s of a wake is logged, not counted.

## 2. The hub missed child processes' exits

On Linux, tokio waits on each child through a pidfd. On macOS it has only SIGCHLD: `wait()` checks once, then waits for the signal. The macOS hub misses that signal, so a child that was still running at the first check is never seen to exit. The visible effects:
- git calls that never return (design-history captures over 120 s, #1110)
- stuck account checks (#1108)
- one zombie process every five minutes from the Claude usage check, which is killed and then dropped

Why the hub misses the signal is not known yet. A standalone program with the same tokio version reaps normally.

**Fix** (`native/host/src/proc.rs`)
- `proc::wait`: also asks for the exit with `try_wait`, every 5 ms at first and backing off to 250 ms. Whichever finishes first wins, the poll or the signal. `try_wait` goes through tokio, so it can never take another child's exit status.
- `proc::output`: `output()` with the exit taken by `proc::wait`. Every git and `sh` call (`files/git.rs`) uses it.
- `proc::kill_and_reap`: kills a child and reaps it in the background, instead of dropping it to tokio's orphan queue (which is drained only on SIGCHLD). Used by the account CLI calls.

## Other platforms

None of this is limited to macOS in the code.
- On Linux, the monotonic clock also stops during suspend, so the clock fix applies there too.
- Elsewhere the changes only add a cheap poll while a child runs. A forward jump of the system clock (more than 10 s) gives failed checks a 30 s grace.

## Still open

- Why the macOS hub misses SIGCHLD.
- Other `child.wait()` callers (tool installs, vault, MCP servers) still rely on it.

## Tests

- `proc::tests::a_childs_exit_is_taken_without_sigchld_and_a_killed_child_is_reaped`
- `plan::tests::a_sleep_is_told_from_a_hang_by_the_two_clocks`
