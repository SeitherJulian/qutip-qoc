---
jupyter:
  jupytext:
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.17.1
  kernelspec:
    display_name: Python 3 (ipykernel)
    language: python
    name: python3
---

# JOPT algorithm for a closed system

```python
import matplotlib.pyplot as plt
import numpy as np
from qutip import gates, qeye, sigmax, sigmay, sigmaz
import qutip as qt
from qutip_qoc import Objective, optimize_pulses

try:
    from jax import jit
    from jax import numpy as jnp
except ImportError:  # JAX not available, skip test
    import pytest
    pytest.skip("JAX not available")

def fidelity(gate, target_gate):
    """
    Fidelity used for unitary gates in qutip-qtrl and qutip-qoc
    """
    return np.abs(gate.overlap(target_gate) / target_gate.norm())
```

## Problem setup

```python
omega = 0.1  # energy splitting
sx, sy, sz = sigmax(), sigmay(), sigmaz()

Hd = 1 / 2 * omega * sz
Hc = [sx, sy, sz]

# objective for optimization
initial_gate = qeye(2)
target_gate = gates.hadamard_transform()

times = np.linspace(0, np.pi / 2, 250)
```

## Guess

```python
jopt_guess = [1, 1]
guess_pulse = jopt_guess[0] * np.sin(jopt_guess[1] * times)

H_guess = [Hd] + [[hc, guess_pulse] for hc in Hc]
evolution_guess = qt.sesolve(H_guess, initial_gate, times)

print('Fidelity: ', fidelity(evolution_guess.states[-1], target_gate))

plt.plot(times, [fidelity(gate, initial_gate) for gate in evolution_guess.states], label="Overlap with initial gate")
plt.plot(times, [fidelity(gate, target_gate) for gate in evolution_guess.states], label="Overlap with target gate")
plt.title("Guess performance")
plt.xlabel('Time')
plt.legend()
plt.show()
```

## JOPT algorithm

```python
@jit
def sin_x(t, c, **kwargs):
    return c[0] * jnp.sin(c[1] * t)

H = [Hd] + [[hc, sin_x] for hc in Hc]
```

### a) not optimized over time

```python
control_params = {
    id: {"guess": jopt_guess, "bounds": [(-1, 1), (0, 2 * np.pi)]}  # c0 and c1
    for id in ['x', 'y', 'z']
}

res_jopt = optimize_pulses(
    objectives = Objective(initial_gate, H, target_gate),
    control_parameters = control_params,
    tlist = times,
    minimizer_kwargs = {
        "method": "Nelder-Mead",
    },
    algorithm_kwargs={
        "alg": "JOPT",
        "fid_err_targ": 0.001,
    },
)

print('Infidelity: ', res_jopt.infidelity)

plt.plot(times, guess_pulse, 'k--', label='guess pulse sx, sy, sz')
plt.plot(times, res_jopt.optimized_controls[0], 'b', label='optimized pulse sx')
plt.plot(times, res_jopt.optimized_controls[1], 'g', label='optimized pulse sy')
plt.plot(times, res_jopt.optimized_controls[2], 'r', label='optimized pulse sz')
plt.title('JOPT pulses')
plt.xlabel('Time')
plt.ylabel('Pulse amplitude')
plt.legend()
plt.show()
```

```python
H_result = [Hd,
            [Hc[0], np.array(res_jopt.optimized_controls[0])],
            [Hc[1], np.array(res_jopt.optimized_controls[1])],
            [Hc[2], np.array(res_jopt.optimized_controls[2])]]
evolution = qt.sesolve(H_result, initial_gate, times)

plt.plot(times, [fidelity(gate, initial_gate) for gate in evolution.states], label="Overlap with initial gate")
plt.plot(times, [fidelity(gate, target_gate) for gate in evolution.states], label="Overlap with target gate")

plt.title('JOPT performance')
plt.xlabel('Time')
plt.legend()
plt.show()
```

### b) optimized over time

```python
# treats time as optimization variable
control_params["__time__"] = {
    "guess": times[len(times) // 2],
    "bounds": [times[0], times[-1]],
}

# run the optimization
res_jopt_time = optimize_pulses(
    objectives = Objective(initial_gate, H, target_gate),
    control_parameters = control_params,
    tlist = times,
    minimizer_kwargs = {
        "method": "Nelder-Mead",
    },
    algorithm_kwargs={
        "alg": "JOPT",
        "fid_err_targ": 0.001,
    },
)

opt_time = res_jopt_time.optimized_params[-1][0]
time_range = times < opt_time

print('Infidelity: ', res_jopt_time.infidelity)
print('Optimized time: ', opt_time)

plt.plot(times, guess_pulse, 'k--', label='guess pulse sx, sy, sz')
plt.plot(times[time_range], np.array(res_jopt_time.optimized_controls[0])[time_range], 'b', label='optimized pulse sx')
plt.plot(times[time_range], np.array(res_jopt_time.optimized_controls[1])[time_range], 'g', label='optimized pulse sy')
plt.plot(times[time_range], np.array(res_jopt_time.optimized_controls[2])[time_range], 'r', label='optimized pulse sz')
plt.title('JOPT pulses (time optimization)')
plt.xlabel('Time')
plt.ylabel('Pulse amplitude')
plt.legend()
plt.show()
```

```python
times2 = times[time_range]
if opt_time not in times2:
    times2 = np.append(times2, opt_time)

H_result = qt.QobjEvo(
    [Hd, [Hc[0], np.array(res_jopt_time.optimized_controls[0])],
         [Hc[1], np.array(res_jopt_time.optimized_controls[1])], 
         [Hc[2], np.array(res_jopt_time.optimized_controls[2])]], tlist=times)
evolution_time = qt.sesolve(H_result, initial_gate, times2)

plt.plot(times2, [fidelity(gate, initial_gate) for gate in evolution_time.states], label="Overlap with initial gate")
plt.plot(times2, [fidelity(gate, target_gate) for gate in evolution_time.states], label="Overlap with target gate")

plt.title('JOPT (optimized over time) performance')
plt.xlabel('Time')
plt.legend()
plt.show()
```

## Global optimization

```python
res_jopt_global = optimize_pulses(
    objectives = Objective(initial_gate, H, target_gate),
    control_parameters = control_params,
    tlist = times,
    algorithm_kwargs = {
        "alg": "JOPT",
        "fid_err_targ": 0.001,
    },
    optimizer_kwargs = {
       "method": "basinhopping",
       "max_iter": 100,
    }
)

global_time = res_jopt_global.optimized_params[-1][0]
global_range = times < global_time

print('Infidelity: ', res_jopt_global.infidelity)
print('Optimized time: ', global_time)

plt.plot(times, guess_pulse, 'k--', label='guess pulse sx, sy, sz')
plt.plot(times[global_range], np.array(res_jopt_global.optimized_controls[0])[global_range], 'b', label='optimized pulse sx')
plt.plot(times[global_range], np.array(res_jopt_global.optimized_controls[1])[global_range], 'g', label='optimized pulse sy')
plt.plot(times[global_range], np.array(res_jopt_global.optimized_controls[2])[global_range], 'r', label='optimized pulse sz')
plt.title('JOPT pulses (global optimization)')
plt.xlabel('Time')
plt.ylabel('Pulse amplitude')
plt.legend()
plt.show()
```

```python
times3 = times[global_range]
if global_time not in times3:
    times3 = np.append(times3, global_time)

H_result = qt.QobjEvo(
    [Hd, [Hc[0], np.array(res_jopt_global.optimized_controls[0])],
         [Hc[1], np.array(res_jopt_global.optimized_controls[1])], 
         [Hc[2], np.array(res_jopt_global.optimized_controls[2])]], tlist=times)
evolution_global = qt.sesolve(H_result, initial_gate, times3)

plt.plot(times3, [fidelity(gate, initial_gate) for gate in evolution_global.states], label="Overlap with initial gate")
plt.plot(times3, [fidelity(gate, target_gate) for gate in evolution_global.states], label="Overlap with target gate")

plt.title('JOPT (global optimization) performance')
plt.xlabel('Time')
plt.legend()
plt.show()
```

## Comparison

```python
fig, axes = plt.subplots(1, 3, figsize=(18, 4))  # 1 row, 3 columns

titles = ["JOPT sx pulses", "JOPT sy pulses", "JOPT sz pulses"]

for i, ax in enumerate(axes):
    ax.plot(times, guess_pulse, label='initial guess')
    ax.plot(times, res_jopt.optimized_controls[i], label='optimized pulse')
    ax.plot(times[time_range], np.array(res_jopt_time.optimized_controls[i])[time_range], label='optimized (over time) pulse')
    ax.plot(times[global_range], np.array(res_jopt_global.optimized_controls[i])[global_range], label='global optimized pulse')
    ax.set_title(titles[i])
    ax.set_xlabel('time')
    ax.set_ylabel('Pulse amplitude')
    ax.legend()

plt.tight_layout()
plt.show()
```

## Validation

```python
assert res_jopt.infidelity < 0.001
assert fidelity(evolution.states[-1], target_gate) > 1-0.001

assert res_jopt_time.infidelity < 0.001
assert fidelity(evolution_time.states[-1], target_gate) > 1-0.001

assert res_jopt_global.infidelity < 0.001
assert fidelity(evolution_global.states[-1], target_gate) > 1-0.001
```

```python
qt.about()
```


