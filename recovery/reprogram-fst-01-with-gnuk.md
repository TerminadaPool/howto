# Manually reprogramming FST-01 with Gnuk
Follow these instructions if:

- You have locked your FST-01 device by entering the password incorrectly 3 times and you have not set a reset code
- For some other reason you "bricked" your Gnuk token (FST-01)

## Install GNU toolchain and newlib for 'arm-none-eabi' target
```
apt -y install binutils-arm-none-eabi gcc-arm-none-eabi gdb-multiarch libnewlib-arm-none-eabi openocd
```

## Get source for Gnuk, update submodule, configure and compile
```
mkdir -p ${HOME}/tmp/src && cd ${HOME}/tmp/src; \
git clone git://git.gniibe.org/gnuk/gnuk.git; \
cd gnuk/; \
git submodule update --init; \
cd src/; \
kdf_do=required ./configure --enable-factory-reset --vidpid=234b:0000 --target=FST_01SZ; \
make; \
cd ../regnual/; \
make;
```

## Connect ST-Link V2 to Gnuk
ST-Link V2 Pins:

Note the notch immediately to left of pin 5 for orientation.

                           +--------+
               1. T_JRST   | 1.  .2 |  2. 3V3 ----------------> 3V3
               3. 5V       | 3.  .4 |  4. T_JTCK/T_SWCLK -----> CLK
               5. SWIM       5.  .6 |  6. T_JTMS/T_SWDIO -----> DIO
    GND <----- 7. GND      | 7.  .8 |  8. T_JTDO
               9. SWIM_RST | 9.  .10| 10. T_JTDI
                           +--------+

Connect the **FST-01SZ** pins as follows:

Look at the underside of board as it has the pins labelled.


Connect the **FST-01 and FST-01G** pins as follows:

                        SWD port
                        (GND, SWD-CLK, SWD-IO)
    Power port +---------------------+
           Vdd |[]           []()() -------+
           GND |[]                  |      |
               |()() I/O port       | USB  |
               |      (PA2, PA3)    |      |
               |                    -------+
               +---------------------+


## Install Gnuk onto FST-01SZ using openocd and ST-Link/V2
Unprotect flash ROM
```
cd ~/src/gnuk/src

openocd -f interface/stlink.cfg -f target/stm32f1x.cfg -c init -c "reset halt" -c "stm32f1x unlock 0" -c reset -c exit
```

Program with new gnuk.elf
```
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg -c "program build/gnuk.elf verify reset exit"
```

Protect flash ROM
```
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg -c init -c "reset halt" -c "stm32f1x lock 0" -c reset -c exit
```

### Install Gnuk onto FST-01 and FST-01G using openocd and ST-Link/V2
When reprogramming the FST-01 and FST-01G board variants you need to induce a hard reset by shorting the NRST and Vssa pins (pins 4 and 5) together whilst asking the ST-Link programmer to connect.  The easiest way is to run a loop which continues to try to connect whilst you use a multi-meter test pin to short pins 4 and 5 on the STM32 chip.  See page: 27/114 of STM32F103TB manual: https://www.st.com/resource/en/datasheet/stm32f103tb.pdf

Run the following loop:
```
while true; do openocd -f interface/stlink.cfg -f target/stm32f1x.cfg; sleep 1; done
```
Then short pins 4 and 5.  You will see the LED light several times and the debugger should connect.

Once the debugger has connected then press Ctrl-C and run the above commands manually.  Alternatively run each command separately via the telnet interface from another terminal:
```
telnet localhost 4444
```
Then run commands:
```
init
reset halt
```
>[stm32f1x.cpu] halted due to debug-request, current mode: Thread 
>xPSR: 0x01000000 pc: 0x08000320 msp: 0x20005000

```
stm32f1x unlock 0
program build/gnuk.elf verify reset
```
> [stm32f1x.cpu] halted due to debug-request, current mode: Thread 
> xPSR: 0x01000000 pc: 0xfffffffe msp: 0xfffffffc
> ** Programming Started **
> ** Programming Finished **
> ** Verify Started **
> ** Verified OK **
> ** Resetting Target **

```
init
reset halt
```
> [stm32f1x.cpu] halted due to debug-request, current mode: Thread 
> xPSR: 0x01000000 pc: 0x08000320 msp: 0x20005000

```
stm32f1x lock 0
```
> device id = 0x20036410
> flash size = 128 KiB
> stm32x locked

etc.


