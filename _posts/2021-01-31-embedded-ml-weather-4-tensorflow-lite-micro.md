---
layout: post
title: "Embedded Machine Learning - Part 4: Running Tensorflow lite micro on the PSoC6"
categories:
  - Projects
  - Electronics
---

[Tensorflow lite micro](https://www.tensorflow.org/lite/microcontrollers) is Tensorflow's runtime that is geared towards small microcontrollers. The runtime is implemented in C++ and will run inference on `*.tflite` files created with the [Tensorflow lite Converter](https://www.tensorflow.org/lite/convert). For many boards, the Tensorflow repository already has examples and dedicated build targets that allows a user to quickly build Tensorflow lite micro for those boards. Unfortunately, our PSoC6 is not among these boards that are relatively easy to target. It sports an ARM Cortex M4, though, so there is generally no problem with running TFlite micro on it. It just requires a bit of extra effort to get it work as intended.

## Running the TFlite micro 'hello world' example

I recommend doing the steps below in a Unix-like environment. I found Windows' WSL2 to work with any problems for this.

In order to include TFlite in our project, we need to take several steps:

1. Compile the TFlite sources as a static library using the Makefile from the TFlite repository
2. Extract the relevant header files from the TFlite repository
3. Extract the source files from the 'hello world' example project
4. Create a function that allows TFlite to write out error messages

Unfortunately, this is a slightly involved process, which is why I have done my best to document the individual steps here.

Before we can start, clone the Tensorflow repository to your local disk 

```bash
git clone https://github.com/tensorflow/tensorflow.git
cd tensorflow
```

### 1. Compile TFlite micro
First, we compile the TFlite sources using the normal TFlite Makefile. Since the PSoC6 is a Cortex-M4 MCU with a floating point unit, we pass the parameters `TARGET=cortex_m_generic TARGET_ARCH=cortex-m4+fp` to the Makefile. After that, we generate a 'hello world'. This project will be useful in two ways: 
- We still need header files for the pre-compiled libraries. Normally, these files are all over the repository. The Makefile will gather the ones relevant for us and put them in the project folder, so we don't need to find them ourselves.
- We can use the code that handles setting up and carrying out the inference as a template if we want to run our own models later on.

In order to generate the library and project, run these two Make targets:

```bash
 make -j -f tensorflow/lite/micro/tools/make/Makefile TARGET=cortex_m_generic TARGET_ARCH=cortex-m4+fp microlite
 make -j -f tensorflow/lite/micro/tools/make/Makefile TARGET=cortex_m_generic TARGET_ARCH=cortex-m4+fp generate_hello_world_make_project
```

**Important:** The aim of this is to get a minimal working setup to play around with, not production-ready code. If you care about performance, you definitely want to build your project using [CMSIS-NN Kernels](https://arm-software.github.io/CMSIS_5/NN/html/index.html). This requires you to have the CMSIS-NN sources in your ModusToolbox project. You also need to add `TAGS=cmsis-nn` to both of the `make` commands above. I will not lay out the exact steps for this here, however.

We can now copy our newly generated static library to our ModusToolbox project
```bash
cp tensorflow/lite/micro/tools/make/gen/cortex_m_generic_cortex-m4+fp/lib/libtensorflow-microlite.a <MTB_Project_Dir>
```
### 2. Extract header files and third-party libraries
In this step, we will copy the TFlite micro header files from the hello world project that is currently buried deep in the Tensorflow directory to our own project. 
Go to the directory where the 'hello world' project was created:
```bash
cd tensorflow/lite/micro/tools/make/gen/cortex_m_generic_cortex-m4+fp/prj/hello_world/make
```
Unfortunately, this folder not only contains the header files, but also a lot of source files that we don't need (because they have already been compiled into the `libtensorflow-microlite.a` file which we created above). This means that we need to use some shell magic in order to copy _only_ the `*.h`, but not the source files from this directory. We can use `find` to do that: 
```bash
# Create temporary folder for header files
mkdir inc
# Copy only the headers and maintain folder structure
find tensorflow/ -type f -name '*.h' -exec cp --parents '{}' -t inc/ \;
```
We can now copy our extracted header files along with the third-party libraries to our own project
```bash
cp -r inc/tensorflow <MTB_Project_Dir>
cp -r third_party <MTB_Project_Dir>
```
### 3. Extract project sources
As mentioned above, we need a handful of source files to give us a basis that compiles and runs inferences as a starting point for our own projects. Another subtlety is that these are C++ sources with the extension `*.cc`. ModusToolbox' automatic build flow would ignore them because it only considers files with the extension `*.cpp` C++ source files. This is just a naming convention, but it means that we need to rename them, otherwise ModusToolbox won't pick them.
```bash
# Go to the project directory where the required source files live
cd tensorflow/lite/micro/examples/hello_world

# Remove the main file, as we will be implementing our own
rm main.cc

# Rename .cc to .cpp so MTB picks them up
rename 's/\.cc$/\.cpp/i' *.cc

# Copy them over to our own project
cp *.cpp <MTB_Project_Dir>
```

### 4. Enable Debug logging

The only real implementation that we need to provide is one that tells TFlite micro what to do with Debug output. Most of this work has already been done for us, so we really only need to provide one simple functions.

- Download [debug_log_callback.h](https://raw.githubusercontent.com/tensorflow/tensorflow/master/tensorflow/lite/micro/cortex_m_generic/debug_log_callback.h) and [debug_log.cc](https://raw.githubusercontent.com/tensorflow/tensorflow/master/tensorflow/lite/micro/cortex_m_generic/debug_log.cc) for generic Cortex-M series MCUs from the Tensorflow repository and place them in your `<MTB_Project_Dir>`

- Create a `main.c` like this:

```c
#include "cy_pdl.h"
#include "cyhal.h"
#include "cybsp.h"
#include "cy_retarget_io.h"

#include "tensorflow/lite/micro/examples/hello_world/main_functions.h"
#include "debug_log_callback.h"

void debug_log_printf(const char* s)
{
    printf(s);
}     

int main(void)
{
    cy_rslt_t result;
    result = cybsp_init();
    if (result != CY_RSLT_SUCCESS)
    {
        CY_ASSERT(0);
    }
    result = cy_retarget_io_init(CYBSP_DEBUG_UART_TX, CYBSP_DEBUG_UART_RX,
                                 CY_RETARGET_IO_BAUDRATE);
    if (result != CY_RSLT_SUCCESS)
    {
        CY_ASSERT(0);
    }
    RegisterDebugLogCallback(debug_log_printf);
    setup();
    for (;;)
    {
        loop();
    }
}
```

Notice that we are using our `printf()` that was retargetted to UART like before to easily display some debugging info on the Serial console.

As a very last step, we need to make two small changes to the MTB Makefile. TFlite micro requires two additional precompiler definitions and we need to make sure that we are using the hardware FPU of our chip. Open the `Makefile` in your `<MTB_Project_Dir>` and find this section:

```makefile
# Add additional defines to the build process (without a leading -D).
DEFINES= 

# Select softfp or hardfp floating point. Default is softfp.
VFP_SELECT=
```

change it to this:

```makefile
# Add additional defines to the build process (without a leading -D).
DEFINES+= TF_LITE_STATIC_MEMORY
DEFINES+= TF_LITE_MCU_DEBUG_LOG

# Select softfp or hardfp floating point. Default is softfp.
VFP_SELECT=hardfp
```



## Inference away!

You can now compile and flash your program like before. Once done, open Putty again and connect to the board. You should see the inference output of the [TFlite micro hello world model](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/lite/micro/examples/hello_world), which approximates a sine function:

![tflite_helloworld](/assets/images/weather_forecast/tflite_helloworld.PNG)

## Summary

In this article, we have 

- Compiled TFlite micro into a static library
- Created a sample project
- Extracted all the necessary files to compile our project in ModusToolbox
- Implemented the necessary debug logging wrappers
- Implemented the code that calls the TFlite micro setup and inference routines
