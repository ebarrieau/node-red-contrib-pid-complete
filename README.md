# node-red-contrib-pid-complete

A node-red node to performe PID Calculations

# Getting started

This node is under active development and may have lots of bugs and breaking changes. I will release on NPM once I deem it stable.

## pid-complete node

This node implements a PID controller that is easy to implement into your flow, but gives you enough options to control complex systems. This implementation uses the parallel (ideal) form of the PID equation instead of the classical form. This means that the system uses three gains (kp, ki, and kd) as opposed to one gain and two times (kp, ti, and td) - this is a design choice of this library.

The setpoint and PID gains are not saved across restarts, so you must initialize these variables in your flow before starting the node or you will get an error.

### Input

1. `msg.payload` - must contain the input process variable (PV) unless a command is send.
2. `msg.cmd` - can contain **reset**, **disable**, **auto**, or **manual**.
3. `msg.topic` - can contain no topic or **sp** with the new setpoint in the `msg.payload`, **kp** with the new proportional gain in `msg.payload`, **ki** with the new integral gain in `msg.payload`, or **kd** with the new derivative gain in `msg.payload`.

### Output

1. `msg.topic` - contains the name of the node.
2. `msg.payload` - contains the output setpoint if in Auto or Manual.
3. `msg.proportional` - contains the output of the last proportional calculation.
4. `msg.integral` - contains the output of the last integral calculation.
5. `msg.derivative` - contains the output of the last derivative calculation.

### Configuration

| Setting                       | Description                                                                      |
| ----------------------------- | ---------------------------------------------------------------------------------|
| `Name`                        | Whatever you name the node. Will be used as topic of node output.               |
| `Calculation Interval`        | Time between calculations. If set faster than inject interval, node will calculate new input on every inject. If set slower than inject interval, node will wait until next inject after calculation interval has elapsed. Set this long if changing the control output quickly would damage the system (e.g. valve packing wearing out from quick control oscillations)    |
| `Max Output - Decreasing PV`  | Sets the largest control ouput that produces a decreasing PV.                    |
| `Max Output - Increasing PV`  | Sets the largest control output that produces an increasing PV.                  |
| `Output When Disabled`        | Default control output when in Disabled mode.                                    |
| `Proportional on Measurement` | Calculates proportional term from measurement instead of from error.             |
| `Derivative on Measurement`   | Calculates derivative term from measurement instead of from error.               |

## pid-complete-autotune node

This node should wire directly between all of your inputs and the PID node. This will sniff the values that it needs (e.g. current state, current setpoint, current PV, etc.) and pass all messages through to the PID node. When you send the command to start autotuning, this node will calculate a hysteresis value based on the `Bump Percent` and setpoint and then go through the following steps:

1. Set PID Node to **Manual**
2. If the PV is higher than setpoint - hysteresis, set the control output to `Max Output - Decreasing PV` until the PV is below setpoint - hysteresis
3. Set the control output to `Max Output - Increasing PV` until the PV is above setpoint + hysteresis
4. Set the control output to `Max Output - Decreasing PV` until the PV is below setpoint - hysteresis
5. Repeat steps 3-4 twice while recording the time and value of the max and min PV during each high and low peak
6. Calculate the PID parameters

Basic PID tuning uses two parameters and a set of tuning rules to calculate the three gains. The parameters are the ultimate gain and the ultimate period. These can be found manually by setting **ki** and **kd** to 0 and slowing increasing **kp** until the PV just starts to oscillate, that is your ultimate gain (ku). You can then measure the period of the oscillations, that is your ultimate period (pu). This autotune node does a simpler test which forces oscillations manually and then measures the commanded amplitude of oscilations vs the actual recorded amplitude and the period of oscillations. This method is not perfect and it is not fast, but is a good starting point to manually tweak your settings. The PID gains are then calculated by using the below tuning rules and equations.

| Rule                   | kp         | ki                | kd                 |
| ---------------------- | ---------- |------------------ |--------------------|
| *Ziegler-Nichols PID*  | 0.60 * ku  | kp / (0.50 * pu)  | kp * (0.125 * pu)  |
| *Ziegler-Nichols PI*   | 0.45 * ku  | kp / (0.833 * pu) | 0                  |
| *Ziegler-Nichols PD*   | 0.80 * ku  | 0                 | kp * (0.125 * pu)  |
| *Pessen-Integral*      | 0.70 * ku  | kp / (0.40 * pu)  | kp * (0.333 * pu)  |
| *Some Overshoot*       | 0.333 * ku | kp / (0.50 * pu)  | kp * (0.333 * pu)  |
| *No Overshoot*         | 0.20 * ku  | kp / (0.50 * pu)  | kp * (0.333 * pu)  |


### Input

1. `msg.payload` - must contain the input process variable (PV) unless a command is send.
2. `msg.cmd` - **autotune** starts the autotuning process. Any other valid `msg.cmd` cancels the autotune.


### Output

This node passes through all messages except `msg.cmd`=  **autotune** which starts autotuning and places the pid-complete node in **manual** control.

When the autotune completes successfully, the node sends a series of messages to set the **kp**, **ki**, and **kd**. These values are not saved in the PID node, so it is a good practice to intercept these values and save them in persistent storage within your node if that is important to your system.

### Configuration

| Setting                 | Description                                                                                              |
| ----------------------- | ---------------------------------------------------------------------------------------------------------|
| `Name`                  | What ever you name the node.                                                                             |
| `Output - Decreasing PV`| Sets the largest control ouput that produces a decreasing PV.                                            |
| `Output - Increasing PV`| Sets the largest control output that produces an increasing PV.                                          |
| `Output When Disabled`  | Default control output when in Disabled mode.                                                            |
| `Tuning Rule`           | Sets the autotuning parameters to be used.                                                               |
| `State after Tuning`    | Does the autotune set the PID controller to disabled, manual, auto, or whatever it was previously set to after successful tuning                                                                                                                    |
| `Autotune Bump Percent` | The hysteresis value used in tuning to separate the max and min peaks, specified as percent of setpoint. |

# Contributing

1. Fork this repo
2. Write a red unit test for your change
3. Implement the code and make the test green
4. Refactor your code to make it nice
5. Make a pull request

I will probably approve it ;)

# Credits

Special thanks to Martin Lundberg https://github.com/m-lundberg/simple-pid and Brett Beauregard https://github.com/br3ttb/Arduino-PID-Library who's work this library is based on.
