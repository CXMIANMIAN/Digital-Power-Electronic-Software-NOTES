# A Guide to Building C-Script Simulations in PlECS

## Preface

**Why choose C-Script as the sole method for circuit control?**

1.  It allows for a quick start with PLECS simulation based on the C language.
2.  It is not limited by existing libraries, allowing for the flexible implementation of various control methods, physical models, and signal processing.
3.  Existing C-Script code can be easily reused in MCUs.

This document attempts to build a systematic framework for C-Script simulation by explaining the configuration parameters, execution flow, key APIs, and design patterns of C-Script. Currently, it only covers the parts I have used.

**Table of Contents**  
I. Introduction to C-Script Parameters  
II. C-Script Execution Flow (Explaining Declaration-Start-...)  
III. Key APIs of the C-Script Module  
IV. Design Patterns for C-Script Simulation  
V. Precautions for C-Script Simulation  

---

## I. Introduction to C-Script Parameters

The Setup module provides multiple key settings, allowing for flexible configuration to achieve various functions.

*   **Number of inputs/outputs**: The quantity of input parameters. As a multi-input multi-output module, C-Script allows you to freely define the number of inputs and outputs. There are two ways to define them:
    *   **Scalar Definition (Digital Definition)**: You can directly input a number, but the C-Script module will only generate a single input/output interface. For multiple inputs/outputs, it must be used in conjunction with a Signal Multiplexer.
    *   **Vector Definition**: Input a vector of corresponding dimensions based on the quantity. In this case, the C-Script module will generate input/output interfaces corresponding to the number of dimensions.
    *   *Note: Scalar definitions and vector definitions can also be used together.*

*   **Number of cont. states**: The number of continuous states. Used to define the quantity of continuous states. Input a positive integer.

*   **Number of disc. states**: The number of discrete states. Used to define the quantity of discrete states. Input a positive integer.

*   **Number of zero-crossing**: The number of zero-crossings. Used to define the quantity of zero-crossing points. Input a positive integer.

*   **Direct feedthrough**: Input direct feedthrough indicator. Used to indicate which input values are direct feedthroughs. When "Number of inputs" uses a vector definition, this value is also defined using a vector of the same dimension:
    *   A corresponding dimension value of **0** indicates a **non-direct feedthrough** input, i.e., the current output does not depend on the current input.
    *   A corresponding dimension value of **1** indicates a **direct feedthrough** input, i.e., the current output depends on the current input.

*   **Sample time**: Sampling time. Determines the C-Script execution frequency.

*   **Parameters**: Parameters defined in the Code section can also be defined here.

---

## II. C-Script Execution Flow

The Code section defines multiple functions; this is where the code is written. The main functions and execution flow are explained below:

*   **Code declaration**: Used to include C standard libraries, define variables, write custom functions, etc. Typically, inputs/outputs and discrete states are also defined in this area.
*   **Start function code**: Used to initialize variables, states, etc., in the Code declaration.
*   **Update function code**: Used to perform parameter calculations and updates.
*   **Output function code**: Used to output signals.

Each time the C-Script is called, the execution flow of the functions in the code section is as follows:
`Output function code` -> `Start function code` -> `Update function code` -> `codeOutput function code` -> `Update function` -> ……

---

## III. Key APIs of the C-Script Module

C-Script is equipped with various APIs. The main ones used are:

*   **Input()/Output()**: Use this API when input/output is defined as a scalar (number).
*   **InputSignal(int i, int j)/OutputSignal(int i, int j)**: Use this API when input/output is defined as a vector. `int i` is the corresponding dimension index.
*   **DiscState(int i)**: Defines a discrete state. `int i` is the index of the "Number of disc. states".
*   **CurrentTime**: Gets the system running time.
*   **SampleTimePeriod(int i)**: Gets the Sample time.

---

## IV. Design Patterns for C-Script Simulation

The simulation functionality of C-Script is flexible, and various functions can be achieved through settings + coding.

### 1. Task Planning
In closed-loop control, you can place variables and custom functions in **Code declaration**, initialize variables and functions in **Start function code**, place loop calculations in **Update function code**, and output PWM wave signals in **Output function code** based on the output of the Update function code.

### 2. PWM Wave Generation
Through the Update function code, the switching frequency can be calculated. Accumulating the switching frequency calculated each time will produce a PWM wave, but the accumulation requires control to output a lossless PWM wave. Two methods can be used: fmod and Phase Accumulator.

*   **fmod Method**: This uses the `fmod` function from the C native library. It performs a triangular carrier calculation `fmod(t, T_SW) / T_SW` using the switching period `T_sw` calculated by the loop and the current time (CurrentTime) `t`, and outputs the PWM wave by controlling the comparison value. Since each phase calculation is real-time, frequency changes are reflected immediately, causing distortion. Therefore, it can only be used for open-loop simulation and cannot be used for closed-loop simulation.

*   **Phase Accumulator Method**: This method calculates the phase value each time and accumulates it. When the phase is greater than 1, a Wrap is performed.
    ```c
    next_phase = current_phase + F_sw * detalT;
    if (next_phase >= 1.0f) {
        next_phase -= 1.0f;
    }
    ```
    Since an accumulation system is adopted, the switching frequency is only modified at the next Wrap when the frequency changes, so no distortion is produced.

**Details of the two methods are as follows:**

**Case 1: Phase Accumulator Method**
Phase per step: `phase = phase + f * Δt`. If it exceeds 1, wrap back to 0.

| Time (µs) | Frequency | Δφ | Phase after Accumulation | Phase after Wrap |
| :--- | :--- | :--- | :--- | :--- |
| 0 | 100k | +0.1 | 0.1 | 0.1 |
| 1 | 100k | +0.1 | 0.2 | 0.2 |
| ... | ... | ... | ... | ... |
| 4 | 100k | +0.1 | 0.5 | 0.5 |
| 5 | **→ Changed to 200k** | +0.2 | 0.7 | 0.7 |
| 6 | 200k | +0.2 | 0.9 | 0.9 |
| 7 | 200k | +0.2 | 1.1 | **wrap→0.1** |
| 8 | 200k | +0.2 | 0.3 | 0.3 |

**Case 2: fmod(CurrentTime, T_sw)**
Phase = `fmod(CurrentTime, T_sw) / T_sw`

| Time (µs) | Current Freq | Current Period (T_sw) | fmod(Time, Period) | Phase |
| :--- | :--- | :--- | :--- | :--- |
| 0 | 100k | 10 | 0 | 0.0 |
| 1 | 100k | 10 | 1 | 0.1 |
| 4 | 100k | 10 | 4 | 0.4 |
| 5 | **→ Changed to 200k** | 5 | 0 | **0.0** (Sudden Change) |
| 6 | 200k | 5 | 1 | 0.2 |
| 7 | 200k | 5 | 2 | 0.4 |

---

## V. Precautions for C-Script Simulation

Closed-loop control is one of the essences of digital power supply control. When using C-Script for closed-loop control, the C-Script module must be configured correctly to achieve stable loop control.

**1. Sample time Settings**
In a closed-loop situation, the Output function code will output a new PWM frequency each time the Sample time is called, and the frequency will not change between two calls. According to the official PLECS documentation:
*   When **Sample time = 0**, it indicates a continuous module. At this time, the execution timing of the C-Script is determined by `Simulation Parameters` -> `Solver` -> `Type`.
    *   When Type=Variable-step, the execution speed is Max step size.
    *   When Type=Fixed-step, the execution speed is Fixed step size.
*   When **Sample time > 0**, it indicates a discrete module, which also determines the execution time of the C-Script. However, it should be noted that the Sample time value must be able to describe the minimum level time $h$ of the output PWM wave, requiring `h > 2 * Sample time`.
*   When **Sample time = -1**, it indicates inheritance, and its characteristics are determined by the input signal.

**2. Simulation parameters solver type settings (Simulation Step Size)**
The simulation step size determines the basic time step for the simulator operation. There are usually two modes:
*   **Fixed-step**: The basic time step for simulator operation is fixed to the Fixed step size. According to the official PLECS manual, `Ts` (Sample time) should be an integer multiple `k` of the Fixed step size, indicating that the C-Script module is called once every `k` time steps.
*   **Variable-step**: The step size can be limited between 0 and the Max step size. The simulator will automatically adjust the step size based on `Ts`.

In the discrete case, fixed-step and variable-step are essentially the same: both call the C-Script once at integer multiples of `Ts`. At this time, the C-Script performs sampling, so the physical sampling time is `Ts`. The difference is that in fixed-step, it is called (sampled) once every `k` equidistant steps `h`, where `Ts = k*h`; in variable-step, it is uncertain what `k` is, it is just called (sampled) once every `Ts` time period, and there are `n` non-equidistant steps between every two call moments.

**3. Simulation Settings and Algebraic Loops**
According to the PLECS manual, if the C-Script is a continuous module (Sample time = 0) and uses continuous quantities directly connected to the inputs and outputs in the circuit, a loop direct feedthrough situation will occur, thereby producing an Algebraic Loop, which causes the loop to fail to work properly. Solutions include:

*   **Sample time > 0**: Set the C-Script module as a discrete module.
*   **Sample time = 0**: In this mode, the C-Script is a continuous module, and the calling time of the C-Script is determined by the simulation step size:
    *   Fixed-step mode: Step size is the Fixed step size.
    *   Variable-step mode: Step size is the Max step size.

    At this time, the input quantity needs to be set as a non-direct feedthrough quantity. The setting item is **Direct feedthrough**. Simultaneously, determine the number of discrete states (**Number of disc. states**) based on the code in the code module. The logic for setting is:

    **Cannot form an algebraic loop -> Direct feedthrough=[0,0] -> Output function cannot use non-direct feedthrough input values -> Need Update function for calculation + transfer -> Need to define discrete state variable -> Determine Number of disc. states based on quantity.**
