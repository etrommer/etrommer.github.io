---
layout: post
title: "Embedded Machine Learning - Part 2: Setting up Modus Toolbox and Hello World on the PSoC 6"
categories:
  - Projects
  - Electronics
---

In this part of the series, we will:
- Set up an environment from which we can write code and program the MCU
- Get to know ModusToolbox
- Create a first embedded "Hello World" style program and flash it to the board
- Read its ouput on a UART console

I will be using the [PSoC 6 WiFi-BT Pioneer Kit](https://www.cypress.com/documentation/development-kitsboards/psoc-6-wifi-bt-pioneer-kit-cy8ckit-062-wifi-bt) together with Cypress' [PSoC 6 HAL](https://cypresssemiconductorco.github.io/psoc6hal/html/modules.html) and their [ModusToolbox IDE](https://www.cypress.com/products/modustoolbox-software-environment). I will hopefully provide all the necessary steps to get working environment. To provide a better overview over the additional tools that come with ModusToolbox, I will start from an empty project. If you are new to the PSoC platform, make sure you go through a few of the many examples that are provided and see what interests you! If you are already familiar with ModusToolbox or if you plan on using a different MCU, feel free to skip this part

## Use ModusToolbox with VS Code on Windows

ModusToolbox is a nicely integrated solution that comes with lots of examples and an eclipse based IDE. Starting from Version 2.1, it also [supports VSCode](https://iotexpert.com/modus-toolbox-2-1-released/) as an alternative IDE. VS Code is my preferred choice and it is fairly to set up, but it is not the default option. This means a few simple steps are required which I have documented here. There is also a variety of other tools that come with ModusToolbox. They are already integrated in Eclipse, but you can invoke them separately if you choose to not use Eclipse. This is the way I will be doing things here. One thing that took me a moment to sort out is that all the tools required to manage a project can be used within the Eclipse-based IDE. You are, however, free to use them as standalone tools as well if that is more your thing. This is what we will be doing here, too.

To create a project, run the ModusToolbox Project Creator:

![](/assets/images/weather_forecast/mtb_project_creator.png)



In the Project Creator, select your board and create a new project from the `Empty PSoC6 App` template (you could select one of the many examples as a template for your project, but with the aim of keeping things generic, let's stick with an empty project for now). You can change the project name in the field left of the template:

![ModusToolbox New project](/assets/images/weather_forecast/mtb_new_project.PNG)

when you are finished, click "Create" to Create the template at your selected location. ModusToolbox will fetch the necessary dependencies for you.

In order to convert the project from Eclipse to VS Code, a `make` command needs to be invoked. The only available instructions for setting up VS Code seem to refer to MacOS where it's arguably a lot easier to run `make` commands than it is on Windows. Luckily, the ModusToolbox install comes with its own shell that can be used for exactly this purpose. Find the tool `modus-shell` in the Start menu and run it:
![ModusToolbox Shell](/assets/images/weather_forecast/mtb_shell.png)

In the modus-shell, navigate to your project directory:

```bash
cd ~/mtw/ml_weather_station
```

and run the Make that creates the project files for VS Code:

```bash
make vscode
```

after the command was run, follow the instructions shown in the shell to set up VS code and import your project:

```
Instructions:
1. Review the modustoolbox.toolsPath property in .vscode/settings.json
2. Open VSCode
3. Install "C/C++" and "Cortex-Debug" extensions
4. File->Open Folder (Welcome page->Start->Open folder)
5. Select the app root directory and open
6. Builds: Terminal->Run Task
7. Debugging: "Bug icon" on the left-hand pane
```

## A first simple test: printf() to UART

Before we write any meaningful code, it's a good idea to test that everything works as intended by putting together a minimal program that prints some text over [UART](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter). We will flash the program and look at the output over a Serial console.

A very handy feature of the PSoC6 HAL is that it allows you to quickly forward the output of the `printf()` command to the board's UART. Conveniently, on the dev board, this UART is connected to the programmer, which opens it as an additional USB Serial device as soon as you connect it to a computer. An excellent combination for some quick `println()`-style debugging (as well as for our first test).

Connecting `printf()` to the UART is handled by the `retarget-io` library. It's not included in our project yet, so we need to change that. Open the [ModusToolbox library manager](https://www.cypress.com/file/513496/download). Then, in the library manager, open your project folder. Under `Libraries`, check `retarget-io`. Click "Update" and the library manager will update your project and fetch any additional files that might be required.

![ModusToolbox Library Manager](/assets/images/weather_forecast/mtb_library_manager.png)

You can now add the following lines to your project:

```c

// ...
#include "cy_retarget_io.h"


int main(void)
{
    // ...
   	result = cy_retarget_io_init(CYBSP_DEBUG_UART_TX, CYBSP_DEBUG_UART_RX, CY_RETARGET_IO_BAUDRATE);
    if (result != CY_RSLT_SUCCESS)
    {
        CY_ASSERT(0);
    }
    for(;;)
    {
        // ...
        printf("Hello printf()!\r\n");
        cyhal_system_delay_ms(1000);
    }
}

```

## Check KitProg Firmware

ModusToolbox only works with programmers that run the **KitProg3** firmware, which means that you _might_ need to upgrade the programmer on your board. Make sure the board is connected via USB and that you are using the USB-C on the board port that has `KitProg2 USB` written next to it. To check which version you are running, open `fw-loader` from the ModusToolbox group in the Windows Start menu.  In the terminal window that opens, run

```bash
./fw-loader.exe --device-list
```

your output should look like this

```
Info: Start API initialization
Info: Connected - KitProg3 CMSIS-DAP BULK-0B0706F301237400
Info: Hardware initialization complete 503 ms
Connected supported devices:
        1: KitProg3 CMSIS-DAP BULK-0B0706F301237400     FW Version 2.10.878
```

If the last line says `KitProg2` instead of `KitProg3`, you need to upgrade. To do this, use the fw-loader terminal to run

```bash
./fw-loader.exe --update-kp3
```

In order to update the firmware on the programmer.

## Program the board

We are finally ready upload our code to the board. You can do this either from VS Code or from the modus-shell. My preferred option is to upload using the Terminal, and only using the IDE for debugging. In order to upload the program from the terminal, open the `modus-shell` again like above. Then navigate to your project directory,

```bash
cd ~/mtw/<PROJECT_DIR>
```

where you can program the board:

```bash
make program
```

To see the output of the program you need to connect to the UART. First you need to find out which COM port your board uses. To do this, make sure your board is plugged in via USB. Once the board is connected, open the Windows device manager. Under `Ports (COM & LPT)`, look for the entry `KitProg3 USB-UART`. After that, it should show a COM port (`COM4` in my case).

Use [PuTTY](https://www.putty.org/) (or any other terminal emulator) to connect to this Port using a baud rate of `115200`.

![Putty Settings](/assets/images/weather_forecast/putty.PNG)

if everything worked, you should see your debug output on the console:

![Putty Hello World](/assets/images/weather_forecast/putty_output.PNG)

## Summary

In the course of this article, we have:

- Created an empty PSoC6 project
- Generated the files necessary for working with VS Code instead of Eclipse
- Added a new library to our project
- Generated some debug output
- Flashed our code to the board
- Displayed the debug output on a Serial console

