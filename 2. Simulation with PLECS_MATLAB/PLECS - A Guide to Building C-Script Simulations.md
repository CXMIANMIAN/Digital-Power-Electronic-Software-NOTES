
Closed-loop control is one of the essences of digital power supply control. When using C-Script for closed-loop control, the C-Script module must be configured correctly to achieve stable loop control.

**1. Sample time Settings**
### 1. Sample time Settings
In a closed-loop situation, the Output function code will output a new PWM frequency each time the Sample time is called, and the frequency will not change between two calls. According to the official PLECS documentation:
*   When **Sample time = 0**, it indicates a continuous module. At this time, the execution timing of the C-Script is determined by `Simulation Parameters` -> `Solver` -> `Type`.
*   When Type=Variable-step, the execution speed is Max step size.
*   When Type=Fixed-step, the execution speed is Fixed step size.
*   When **Sample time > 0**, it indicates a discrete module, which also determines the execution time of the C-Script. However, it should be noted that the Sample time value must be able to describe the minimum level time $h$ of the output PWM wave, requiring `h > 2 * Sample time`.
*   When **Sample time = -1**, it indicates inheritance, and its characteristics are determined by the input signal.

**2. Simulation parameters solver type settings (Simulation Step Size)**
### 2. Simulation parameters solver type settings (Simulation Step Size)
The simulation step size determines the basic time step for the simulator operation. There are usually two modes:
*   **Fixed-step**: The basic time step for simulator operation is fixed to the Fixed step size. According to the official PLECS manual, `Ts` (Sample time) should be an integer multiple `k` of the Fixed step size, indicating that the C-Script module is called once every `k` time steps.
*   **Variable-step**: The step size can be limited between 0 and the Max step size. The simulator will automatically adjust the step size based on `Ts`.

In the discrete case, fixed-step and variable-step are essentially the same: both call the C-Script once at integer multiples of `Ts`. At this time, the C-Script performs sampling, so the physical sampling time is `Ts`. The difference is that in fixed-step, it is called (sampled) once every `k` equidistant steps `h`, where `Ts = k*h`; in variable-step, it is uncertain what `k` is, it is just called (sampled) once every `Ts` time period, and there are `n` non-equidistant steps between every two call moments.

**3. Simulation Settings and Algebraic Loops**
### 3. Simulation Settings and Algebraic Loops
According to the PLECS manual, if the C-Script is a continuous module (Sample time = 0) and uses continuous quantities directly connected to the inputs and outputs in the circuit, a loop direct feedthrough situation will occur, thereby producing an Algebraic Loop, which causes the loop to fail to work properly. Solutions include:

*   **Sample time > 0**: Set the C-Script module as a discrete module.
