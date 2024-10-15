# The Techie's guide to code generation in Simulink

This document is intended as a collection of guidelines and instructions on how to effectively generate code from Matlab/Simulink, more specifically the second. It does not contain strict rules to follow, but aims to be a collection of suggested practices learned in the years in iCub Tech.

The document targets the **AMCx family** of embedded boards.

-----

**Table of contents**

- [The Techie's guide to code generation in Simulink](#the-techies-guide-to-code-generation-in-simulink)
  - [Setup](#setup)
    - [Requirements](#requirements)
      - [Matlab Toolboxes](#matlab-toolboxes)
      - [Board support packages](#board-support-packages)
    - [Operating System](#operating-system)
    - [Compiler](#compiler)
  - [Understanding model configuration parameters](#understanding-model-configuration-parameters)
    - [Solver](#solver)
    - [Math and Data Types](#math-and-data-types)
    - [Hardware Implementation](#hardware-implementation)
    - [Model Referencing](#model-referencing)
    - [Simulation Target](#simulation-target)
    - [Code Generation](#code-generation)
    - [Optimization](#optimization)
    - [Report](#report)
    - [Custom Code](#custom-code)
    - [Interface](#interface)
      - [Code replacement libraries](#code-replacement-libraries)
  - [Modelling guidelines](#modelling-guidelines)
    - [Simulink models](#simulink-models)
    - [Stateflow charts](#stateflow-charts)
    - [Simscape ???](#simscape-)

-----

## Setup

### Requirements

#### Matlab Toolboxes

The generation of code starts from Matlab and Simulink.
The minimum requirements are the following toolboxes:

- Matlab Coder
- Simulink Coder
- Embedded Coder

The Embedded Coder is special, since it allows to generate production code that can be deployment on the boards, and leverage board support packages/CMSIS.

System Composer is a nice addition, since it allows to create complex architectural models with embedded tests and requirements.

Other toolboxes are needed depending on the software to implement:

- Control System Toolbox, for controllers such as PIDs
- DSP Toolbox for filters

and so on.

#### Board support packages

Board support packages (BSPs) are add-ons that contain features targeted for specific microcontrollers. With them the developer can leverage native uC features, such as ARM instructions (CMSIS), blocks that implement communication protocols and so on.

The AMCx boards have ARM Cortex M processors, which BSP can be found here: [click](https://it.mathworks.com/matlabcentral/fileexchange/43095-embedded-coder-support-package-for-arm-cortex-m-processors).

### Operating System

The operating system (OS) in which Matlab is installed can affect the code generation, depending on the target system.
Since we are targeting a specific microcontroller architecture with dedicated BSP, we won't be affected.

However, if a developer wants to target a e.g. Linux distribution, it is **not** suggested to generate code on a different platform. This is caused by the Embedded Coder searching for host OS libraries instead of the target ones by default. It can happen when using features like mutexes.

### Compiler

To enable code generation, a compiler is needed. Simulink will scan the programs $PATH for supported compilers, to help with selection. At the moment of writing this document, no formal analysis has been performed regarding which compiler is preferrable.

This is because while we generate code in Simulink, we copy it and compile it in the ARM Keil IDE, using the integrated ARM Clang compiler.

Nonetheless, to enable code generation, it is possible to install for example one of the following:

- The Visual Studio compiler, by downloading Microsoft Visual Studio
- The ARM Clang compiler, by downloading ARM Keil v5.39.

## Understanding model configuration parameters

The model configuration parameters window contains most settings available to Simulink models.

To open the window, open the Simulink model of your choice, and type Ctrl+E.

This will open:
![](assets/modelparams.png)

Next, we will go through the most relevant parts of the configuration parameters for the AMCx boards.

### Solver

![](assets/modelparams.png)

Solver parameters influence both the code generation and the simulation environment.

**Solver selection**: The solver type must be `Fixed step` with `discrete` method. In this way, generation of code for continuous states can be avoided.

**Solver details > Fixed step size**: The step size can be set to:

- a value that is the least common denominator (LCD) between all the sample times of the application
- `auto` to let Simulink find the LCD step size; note that if the rates are not integer multiples, an `step()` function with the LCD will be generated in the code

**Tasking and sample time options**:

- [x] *Allow tasks to execute concurrently*: this setting is very important, since it allows to generate code with different sampling times, and unlocks the full scheduling capabilities of Simulink through the *Configure Tasks...* button
- [x] *Automatically handle rate transition*: This settings is useful to let simulink insert the proper rate transition block between blocks that run at different rates; the data transfer can be set as always deterministic, if the application can allow significant delays; we use `Never` to minimize delay and insert the rate transition blocks manually

### Math and Data Types

![](assets/par_math.png)

**Data Types > Default for underspecified data type**: `single` because the processor of the AMCx boards does not support double precision. Nonetheless, it is always good practice to specify the smallest possible type for each variable.

### Hardware Implementation

![](assets/par_hw.png)

The *Hardware Implementation* tab is used to set the target architecture and specific BSP.
For our use case, we don't target a specific hardware board.

**Code Generation system target file**: `ert.tlc`, to use the Embedded Coder generation rules.

**Device vendor:** ARM Compatible, since the AMCx boards use ARM processors.

**Device type:** ARM Cortex-M, the type of ARM processor. Note that this setting is available only if the proper BSP for the Embedded Coder is installed.


### Model Referencing

![](assets/par_mod.png)

**Options for referenced models > Rebuild:**: set to *Always* to make sure to rebuild all referenced models at every Simulink compilation step. Useful before deployment. For development and quick iterations, *If changes in known dependencies are detected* can be used.

**Options for referencing this model > Total number of instances allowed per top model**: this setting is useful to constrain model referencing. Set it to *One* if you want to treat the current model as a singleton, or *Multiple* if you want to let this model be used in multiple places.

### Simulation Target

![](assets/par_sim.png)


**Custom code**: in this form you can specify header and source files with functions that can be called in Simulink. Setting them in this section allows running both simulation and code generation without errors.

**Advanced parameters > Compiler optimization level**: this setting is not dedicated to code generation, but setting *Optimizations on* can significantly speed up simulations.

### Code Generation

![](assets/par_code.png)

**Target selection > System target file**: `ert.tlc`, to use the Embedded Coder generation rules.

**Shared coder dictionary**: It could be useful to use a Simulink dictonary to store codegen configurations, though not mandatory.

**Language/Language Standard**: at the moment of writing this document, the language standard used for code generation is C++03, in accordance to the available features of the ARMCLANG compiler that comes with ARM Keil.

  - [x] **Generate code only**: ticked to avoid the compilation step by Makefile execution. Compilation occurs in the ARM Keil IDE.

**Build process > Toolchain**: Selecting a specific toolchain allows usage of compiler-specific language features, and compilation parameters. At the time of writing this document, no analyses have been performed so any installed compiler can be used. The Visual Studio compiler and ARMCLANG are usually used during development.

**Build process > Build Configuration**: Set it to *Faster Runs* to select the available optimization options of the selected compiler.



### Optimization

![](assets/par_opt.png)

**Default parameter behavior**: Set it to *Inlined* to allow parameters to be replaced by their respective constant values, so that the code does not allocate memory to allocate them. The *Tunable* setting will instead create variables for tunable parameters present in the model, at the cost of more memory usage.

**Pass reusable subsystem outputs as**: *Structure reference* to make the code more convenient to use, at the cost of using more global memory. The option *individual arguments* passes each reusable subsystem output argument as an address of a local, instead of as a pointer to an area of global memory containing the output arguments. 

- [x] **Data initialization > Remove...**: You can tick both options to remove the zero initialization of both internal variables and IO ports.
  
**Optimization levels > Level**: Maximum to apply all the possible optimizations.
**Optimization levels > Priority**: *Maximize execution speed*.


### Report

![](assets/par_rep.png)

It is very useful to generate a summary report of the code generation process, especially for checking which functions triggered a code replacement.

To do so, tick *Create code generation report*, and *Summarize which block triggered code replacement*.

### Custom Code

![](assets/par_cust.png)

In this section, it is useful to tick the setting *Use the same custom code settings as Simulation Target*. See the [Simulation Target](#simulation-target) section for details.

### Interface

![](assets/par_iface.png)

In this section you will be able to set up the interface of the generated code, especially the way it can be called.

Additionally, in **Software environment > Code replacement libraries**, it is possible to select which replacement libraries to apply. For more details, refer to the dedicated section.

We will now focus on the visible settings.

**Software enviroment > Support**:

- [x] floating-point numbers
- [x] absolute time
- [ ] non-finite numbers -> tick it if you expect signals to reach `inf`
- [ ] complex numbers
- [ ] variable-size signals -> tick it if you expect data containers to change in size at runtime

**Code interface > Code interface packaging**:

- *Nonreusable function*: selecting this option generates code which entry point functions directly access the data structure; this option is suggested for top-level architectural models
- *Reusable function*: selecting this option generates re-entrant multi-instance code; this option is needed for Simulink models that need to be referenced, especially if they need to be wrapped in special blocks (e.g. a *For Each Subsystem*)
- *C++ class*: this options generates a class with constructor, destructor, inputs setter and outputs getter; this option is available only if the C++ language is selected as target

#### Code replacement libraries
Code replacement libraries (CRL) are sets of functions that can be used to replace native C/C++ operations, to leverage the target hardware or respect specific requirements.

When clicking on the *Select* button, the following window will pop up.
![](assets/crl.png)
On the left the available but unused libraries are listed, while on the right we can see the ones selected for usage, with decreasing priority order.

If the BSP for the Cortex-M was installed, it will appear here as available. Its CRL includes functions such as the trigonometric ones, squared root, even clark-parke transforms.

It is possible to define a custom CRL, such as the iCubTech one. 
The iCubTech library replaces native mutex calls with custom functions that trigger interrupts. More information on how to create a custom library can be found in the [Matlab documentation](https://it.mathworks.com/help/ecoder/ug/quick-start-library-development-sc.html).

## Modelling guidelines

### Simulink models

### Stateflow charts

### Simscape ???
