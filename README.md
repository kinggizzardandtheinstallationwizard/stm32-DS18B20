# DS18B20 libopencm3 OneWire library for stm32

The test was carried out on stm32f103c8t6, however, the list of supported devices should be much wider. Actually not used
no code specific to stm32f1.

## Description of the library

The library is designed to work with widely used temperature sensors Maxim (Dallas) DS18B20
by the OneWire protocol.

The library [libopencm3](https://libopencm3.github.io/) was chosen as the basis for the implementation.
libopencm3 is a very good basis for programming on stm32, however, due to poor documentation
is slightly underrated by hobbyists. The resulting code is easy to read, compact and efficient, compared to
SPL и HAL от ST Microelectronics.

The OneWire library implements SKIP, SEARCH, MATCH, READ, READ SCRATCHPAD, CONVERT TEMPERATURE, RECALL E2 and other commands and runs on a microcontroller
stm32f103c8t6 and, most likely, a host of others.

The library involves connecting sensors (up to 75 on one line) to any of the TX USART/UART outputs. Assumes availability
external power supply of sensors and pull-up of the DATA line to the power line with a resistance with a rating corresponding to the task
(default 4.7K). The USART/UART operating mode in this case is half-duplex. The developer still has free gpio(s) at his disposal.

Assuming [use of hardware USART/UART](https://www.maximintegrated.com/en/app-notes/index.mvp/id/214). The library should work well as part of an RTOS, because, in fact, all critical
operations are made atomic.

The library allows you to organize 5 independent OneWire buses (according to the number of USART/UART hardware support). The number of devices on the bus is determined
developer. In default mode, up to 5 devices are expected on the bus.

The project is built on Clion on any OS (Mac OS X, Windows, Linux) + arm-none-eabi + cmake. Debugging was carried out by blackmagic probe and .gdbinit
corresponds to initialization of the connection process.

## A little about the sensor

The DS18B20 sensor, as well as the DS18S20, belongs to a class of devices whose purpose and operation require explanation.

Firstly, these sensors have their own memory, settings and operating logic. Their interaction with uK is carried out on the basis of possible interaction protocols.
In the process of preparing new data, the sensor consumes quite a lot of electricity (up to 1.5mA) and therefore does not measure continuously.
Therefore, the last measurement he made does not mean the "current temperature". It is reasonable to assume that while a command has been sent to the sensor
temperature measurement, the microcontroller should be busy with something useful (or sleep), instead of simply waiting for the sensor to be ready.

Secondly, the sensor can do quite a lot besides measuring temperature. For example, the sensor can set “alarm” limits and in response to
special request to report that the temperature has exceeded these limits. This could be useful for example for a scenario like this:
placement of a group of sensors around the territory (for example, along a wall), simultaneous sending of a request to measure the temperature, and then a command to
identification of those sensors whose measured temperature goes beyond previously established limits. Accordingly, then polling not all sensors,
but only those who generated “anxiety”.

In addition, these two sensor bytes (which set the alarm limits) can be used arbitrarily to store some information that can be stored
in the sensor EEPROM and used later.

It is clear that these sensors are intended to be placed very far from uK. Its distance can be tens of meters
(actually up to 30 meters).


## Program structure

Let's assume that the OneWire bus will be implemented on the USART3 of the blue tablet (bluepile) stm32f103c8t6, a very cheap and affordable board for experiments.
The library uses the appropriate interrupt(s) to organize the reading of RX at the time of transfer to the line via TX. Reminds me of creation
loopback на UART.

Therefore you should declare:

```C
// Must be declared BEFORE include "OneWire.h" so that the appropriate interrupt handler is added
//#define ONEWIRE_UART5
//#define ONEWIRE_UART4
#define ONEWIRE_USART3
//#define ONEWIRE_USART2
//#define ONEWIRE_USART1

//maximum number of devices on the bus by default
//MAXDEVICES_ON_THE_BUS 5

#include "OneWire.h"
```

In order to use the library, you should initialize the “clock” in `static void clock_setup(void)`:

```C
rcc_periph_clock_enable(RCC_GPIOB);
```

Afterwards, configure gpio in `static void gpio_setup(void)`:

```C
gpio_set_mode(GPIOB, GPIO_MODE_OUTPUT_10_MHZ,
GPIO_CNF_OUTPUT_ALTFN_OPENDRAIN, GPIO_USART3_TX | GPIO_USART3_RX);

```

And declare (local or global - whichever is more convenient) a variable that will store information about devices on the bus:

```C
OneWire ow;
```

If you want to increase/decrease the number of devices about which information will be saved by default, determine it yourself BEFORE
connections OneWire.h `MAXDEVICES_ON_THE_BUS`

After this, everything is ready to use.
