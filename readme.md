Raspberry Pi source files, see https://iosoft.blog for details

WS2812B LEDs (‘NeoPixels’) are intelligent devices, that can be programmed to a specific 24-bit red, green & blue (RGB) colour, by a pulse train on a single wire. They are capable of being daisy-chained, so a single pulse line can drive a large number of devices.

The programming pulses have to be accurately timed to within fractions of a microsecond, so conventional Raspberry Pi techniques are of limited use; they can only handle a small number of pulse channels, driving a maximum of 1 or 2 strings of LEDs, which is insufficient for a complex display.

This blog post describes a new technique that uses the RPi Secondary Memory Interface (SMI) to drive 8 or 16 channels with very accurate timing. No additional hardware is needed, apart from a 3.3 to 5V level-shifter, that is required for all NeoPixel interfaces.

Pulse shape
To set one device, it is necessary to send 24 pulses with specific widths into its input; they represent green bits G7 to G0, red bits R7 to R0, and blue bits B7 to B0, in that order. If you send more than 24 bits, the extra will emerge from the data output pin, which can drive the data input of the next device in the chain, so to drive ‘n’ LEDs, it is necessary to send n * 24 pulses, without any sizeable gaps in the transmission. If the data line is held low for the ‘reset’ time (or longer), the next transmission will restart at the first LED. Here is a waveform for 2 LEDs in a chain, the first being set to red (RGB 240,0,0) the second to green (RGB 0,240,0).


LED pulse sequence
It is just about possible to see the 0 and 1 pulses on the oscilloscope trace above, here is a zoomed-in section of that trace, to show the varying pulse width more clearly.


LED pulses
You can see that the pulses for the second LED are offset by about 200 nanoseconds from those of the first LED; this is because the first LED is regenerating the signal, rather than just copying the input to output.

The precise definition of a 0/1/reset pulse depends which version of the LED you are using, but the commonly-accepted values in microseconds (us) are:

0:   0.4 us high, 0.85 us low, tolerance +/- 0.15 us
1:   0.8 us high, 0.45 us low, tolerance +/- 0.15 us
RST: at least 50 us (older devices) or 300 us (newer devices)
To simplify the code, we can tweak the values slightly, whilst still remaining in the tolerance band, so my code generates a ‘0’ pulse as 0.4 us high, 0.8 us low, and a ‘1’ pulse as 0.8 us high, 0.4 us low.

Generating the pulses
Since the pulses have to be quite accurately timed, and the Raspberry Pi has no specific hardware for pulse generation, the pulse width modulation (PWM) or audio interfaces are generally used, but this imposes a significant limitation on the number of LED channels that can be supported. I’m using the Secondary Memory Interface (SMI) instead, which could provide up to 18 channels, though my software only supports 8 or 16.

If you are interested in learning more about SMI, I have written a detailed post here, but the key points are:

The SMI hardware is included as standard in all Raspberry Pi versions, but is little-used due to the lack of publicly-available documentation.
As the name implies, it is intended to support fast transfers to & from external memory, so it can efficiently transfer blocks of 8- or 16-bit data.
The timing of the transfers can be controlled to within a few nanoseconds, making it ideal for generating accurate pulses.
When driven by Direct Memory Access (DMA), the SMI output will proceed without any CPU intervention, so the timing will still be accurate even when using the slowest of CPUs (e.g. Pi Zero or 1).
SMI output uses specific pins on the I/O header: SD0 to SD17 as shown below.

SMI pins on I/O header
My code supports both 8 and 16-bit SMI output, which equates to 8 or 16 output channels, where each channel can have an arbitrarily long string of LEDs.

Each LED can be individually programmed to any RGB value, the only limitation is that all channels will transmit the same number of RGB values. This is not a problem, since any extra settings have no effect; if 2 LEDs receive 5 RGB values, they will accept the first 2 values, and ignore the rest.

When first generating a pulse train, it is worth checking that the output is as expected, so I first sent the following byte values to the 8-bit SMI interface, using DMA:

// Data for simple transmission test
uint8_t tx_test_data[] = {1, 2, 3, 4, 5, 6, 7, 0};
This should result in a classic stepped binary output, but instead the lowest 3 data bits were as follows:


Binary test waveform: 8-bit SMI
The voltage level and pulse width are correct, but the bytes in each 8-bit word are swapped. This can be corrected by a simple function, using a GCC builtin byte-swap call:

// Swap adjacent bytes in transmit data
void swap_bytes(void *data, int len)
{
    uint16_t *wp = (uint16_t *)data;
     
    len = (len + 1) / 2;
    while (len-- > 0)
    {
        *wp = __builtin_bswap16(*wp);
        wp++;
    }
}
The resulting waveform is correct:


Corrected test waveform
This byte-swapping isn’t necessary if running a 16-bit interface; the first byte has the least-significant data bits, in the usual little-endian format.

Hardware
The data outputs are SD0 – SD7 for 8 channels, or SD0 – SD15 for 16 channels:


SMI I/O lines
If 8 channels are sufficient, it is worthwhile setting the software to do this, since it halves the DMA bandwidth requirement, and reduces the possibility of a timing conflict with the video drivers.

The LEDs need a 5 volt power supply, which can be quite sizeable if there is a large number of devices. A single device takes around 34 mA when at full RGB output, so the standard Pi supply can only be used for relatively small numbers of LEDs.

The channel output signals also need to be stepped up from 3.3V to 5V. There are various ways to do this, I used a TXB0108 bi-directional converter, which generally works OK, but the waveform isn’t correct driving some budget-price devices. In the following graphic, the bottom oscilloscope trace shows a good-quality square wave with 5V amplitude; the upper trace peaks around 5V, but then decays to nearly 3V, which is outside the LED specification.


Correct (lower) and incorrect (upper) drive waveforms
The cheap devices seem to have higher input capacitance than other NeoPixels, and this triggers an issue with the TXB0108, which has an unusual automatic bi-directional ability. Every time an input changes, it emits a brief current pulse to drive the output, then keeps the output in that state using a weak drive. The TXB0108 data sheet warns against driving high-capacitance loads; to get a good-quality waveform, it’d be much better to use a conventional level-shifter such as the 74LVC245.

For quick testing, it is possible to drive a few LEDs from the RPi 3.3V supply rail, in which case the RPi output pin can be connected directly to the LED digital input, without level-shifting; this is outside the specification of the device, but generally works, providing the supply isn’t overloaded.

Software
As the application has to drive the SMI interface and DMA controller, it is written in C, and must run with root privileges (using ‘sudo’). You can find detailed information on SMI here, and DMA here.

In contrast to my other DMA programs, driving WS2812 LEDs is relatively straightforward; it just requires a single block of data to be transmitted, and a single DMA descriptor to transmit it. There is no need for additional DMA pacing, as all the pulse timing is handled by SMI, to a really high accuracy (around 1 nanosecond, according to my measurements).

The only tricky part is the preparation of data for the transmit buffer. Each LED needs to receive 24 bits of GRB (green, red, blue) data, and each bit has 3 pulses: the first is ‘1’, the second is ‘0’ or ‘1’ according to the data, and the third is ‘0’. Each pulse is a single SMI write cycle. The data for all channels is sent out simultaneously, so if we are driving 8 channels, the byte sequence will be:

           Ch7 Ch6 Ch5 Ch4 Ch3 Ch2 Ch1 Ch0
            1   1   1   1   1   1   1   1
Grn bit 7:  x   x   x   x   x   x   x   x
            0   0   0   0   0   0   0   0
            1   1   1   1   1   1   1   1
Grn bit 6:  x   x   x   x   x   x   x   x
            0   0   0   0   0   0   0   0
..and so on until..
            1   1   1   1   1   1   1   1
Grn bit 0:  x   x   x   x   x   x   x   x
            0   0   0   0   0   0   0   0
            1   1   1   1   1   1   1   1
Red bit 7:  x   x   x   x   x   x   x   x
            0   0   0   0   0   0   0   0
..and so on until..
            1   1   1   1   1   1   1   1
Blu bit 0:  x   x   x   x   x   x   x   x
            0   0   0   0   0   0   0   0
The encoder function takes a 1-dimensional array of RGB values (1 RGB value per channel), converts them to GRB, and writes the corresponding sequence to the transmit buffer. To handle 8 or 16 channels, the buffer data type is switched between 8 and 16 bits.

#define LED_NCHANS      16  // Number of LED string channels (8 or 16)
#define BIT_NPULSES     3   // Number of O/P pulses per LED bit
 
#if LED_NCHANS > 8
#define TXDATA_T        uint16_t
#else
#define TXDATA_T        uint8_t
#endif
 
// Set transmit data for 8 LEDs (1 per chan), given 8 RGB vals
// Logic 1 is 0.8us high, 0.4 us low, logic 0 is 0.4us high, 0.8us low
void rgb_txdata(int *rgbs, TXDATA_T *txd)
{
    int i, n, msk;
 
    // For each bit of the 24-bit RGB values..
    for (n=0; n<LED_NBITS; n++)
    {
        // Mask to convert RGB to GRB, M.S bit first
        msk = n==0 ? 0x8000 : n==8 ? 0x800000 : n==16 ? 0x80 : msk>>1;
        // 1st byte is a high pulse on all lines
        txd[0] = (TXDATA_T)0xffff;
        // 2nd byte has high or low bits from data
        // 3rd byte is low pulse
        txd[1] = txd[2] = 0;
        for (i=0; i<LED_NCHANS; i++)
        {
            if (rgbs[i] & msk)
                txd[1] |= (1 << i);
        }
        txd += BIT_NPULSES;
    }
}
Beware caching
If you are modifying the software, there is a major trap, that I fell into shortly before releasing the code.

Everything was working fine on RPi v3 hardware, then I switched to a Pi Zero, and it was a disaster; the pulse sequences were all over the place, bearing no resemblance to what they should be.

I then tried outputting a simple 8-bit binary sequence, and that was wrong as well; the code steps were:

Copy data into transmit buffer
Byte-swap the buffer data
Transmit the buffer data
Looking at the output on an oscilloscope, the byte-swap function wasn’t working; no matter how I modified the code, it was doing nothing. I then realised there is a golden rule of DMA programming: if your code is behaving illogically, it is probably due to caching.

The transmit buffer has been allocated in uncached video memory, as the DMA controller doesn’t have access to the CPU cache – for more details, see my post on DMA. Since the transmit buffer pointer was defined as non-cached and volatile, the data was copied to it immediately, but then the subsequent byte-swapping will have taken place the CPU’s on-chip cache. Eventually this cache would be written back to the physical memory, but in the short term, there is a mismatch between the two – and the DMA controller will use the copy in physical memory. So the cure is simple; just do the byte-swapping before the data is written to the transmit buffer.

Switching back to the real pulse-generating code, again this was all being done in the transmit buffer:

Prepare data in transmit buffer
Byte-swap the buffer data
Transmit the buffer data
Now, in addition to the byte-swap issue, we also have a caching problem in the data preparation, as it involves lots of bit-twiddling; even if we could persuade the compiler to ignore the cache, the code would run quite slowly due to the absence of caching. The best solution is to prepare all the data in local (cached) memory, then finally copy it across to the uncached memory for transmission:

Prepare data in local buffer
Byte-swap the local data
Copy local data to transmit buffer
Output the transmit buffer data
Source code
The main source file is rpi_pixleds.c, it uses functions from rpi_dma_utils.c and .h, and SMI definitions from rpi_smi_defs.h, available on Github here.

It is essential to modify the PHYS_REG_BASE setting in rpi_smi_defs.h to reflect the RPi hardware version:

// Location of peripheral registers in physical memory
#define PHYS_REG_BASE   PI_23_REG_BASE
#define PI_01_REG_BASE  0x20000000  // Pi Zero or 1
#define PI_23_REG_BASE  0x3F000000  // Pi 2 or 3
#define PI_4_REG_BASE   0xFE000000  // Pi 4
The application is compiled and run using:

gcc -Wall -o rpi_pixleds rpi_pixleds.c rpi_dma_utils.c
 
sudo ./rpi_pixleds [options] [RGB_values]
The options can be in upper or lower case:

-n num    # Set number of LEDs per channel
-t        # Set test mode
Test mode generates a chaser-light pattern for the given number of LEDs, on 8 or 16 channels as specified at compile-time, e.g. for 5 LEDs per channel:

sudo ./rpi_pixleds -n 5 -t
It is also possible to set the RGB value of an individual LED using 6-character hexadecimals, so full red is FF0000, full green 00FF00, and full blue 0000FF. The RGB values for each LED in a channel are delimited by commas, and the channels are delimited by whitespace, e.g.

# All 8 or 16 channels, 5 LEDs per channel, all off
sudo ./rpi_pixleds -n 5
 
# All 8 or 16 channels, 3 LEDs per channel, all off apart from Ch2 LED0 red
sudo ./rpi_pixleds -n 3 0 0 ff0000
 
# 3 active channels, 1 LED per channel, set to half-intensity red, green, blue
sudo ./rpi_pixleds 7f0000 007f00 00007F
 
# 3 active channels, 2 LEDs per channel, set to full & light red, green, blue
sudo ./rpi_pixleds  ff0000,ff2020 00ff00,20ff20 0000ff,2020ff
You will note that it isn’t necessary to specify the number of LEDs per channel when RGB data is given; the code counts the number of RGB values for each channel, and uses the highest number for all the channels.

Compile-time options
The following definitions are at the top of the main source file:

#define TX_TEST         0   // If non-zero, use dummy Tx data
#define LED_D0_PIN      8   // GPIO pin for D0 output
#define LED_NCHANS      8   // Number of LED channels (8 or 16)
#define LED_NBITS       24  // Number of data bits per LED
#define LED_PREBITS     4   // Number of zero bits before LED data
#define LED_POSTBITS    4   // Number of zero bits after LED data
#define BIT_NPULSES     3   // Number of O/P pulses per LED bit
#define CHAN_MAXLEDS    50  // Maximum number of LEDs per channel
#define CHASE_MSEC      100 // Delay time for chaser light test
The main items that might be changed are:

TX_TEST, set non-zero to output a simple binary sequence on data bits 0-2, for checking that SMI works as intended.
LED_NCHANS, to specify either 8 or 16 channels; set to 8 if this number is sufficient.
CHAN_MAXLEDS, to increase the maximum number of LEDs allowed per channel. This is only used for dimensioning the data arrays; the actual number of LEDs per channel is specified at run-time.
CHASE_MSEC, to increase or decrease the delay time for test mode
Possible problems
If the program doesn’t work, here are some issues to check:

PHYS_REG_BASE: make sure you have set this correctly for the RPi version you are using.
Power overload: check that there is an adequate reserve of power, assuming around 34 mA per LED.
Incorrect voltage: ensure that the data lines are being driven to the full supply voltage, usually 5 volts.
Stuck pixels: if you specify too few RGB values for a given channel, then the remaining LEDs will be unchanged, and may appear to be ‘stuck’. If in doubt, use the -n option to set the actual number of LEDs per channel, to ensure all the LEDs receive some data.
Caching problems: if you have modified the pulse-generating code, and it is behaving illogically, then you probably have a caching issue. See the detailed description above.
RPi v4: test mode may only set 1 row of LEDs, then stop. The reason for this is not understood, but it can be cured by running headless as described below.
There can be an intermittent problem with brief flickering of LEDs to other colours when running a fast-changing test such as the chaser-lights, or maybe occasional colour errors on a slowly-changing display, if running in 16-channel mode. This is due to the HDMI display drivers taking priority over the SMI memory accesses, causing jitter in the pulse timing. I suspect it can be cured by changing the priorities, but due to time pressure, I’ve been taking the easy way out, and disabling HDMI output using:

/usr/bin/tvservice -o
Sadly restoring the output (using ‘/usr/bin/tvservice -p’) doesn’t restore the desktop image, so the Rpi is run ‘headless’, controlled using ssh. More work is needed to find an easier solution.

Safety warning: take care when creating a rapidly-changing display, as some people can be adversely affected by flashing lights; research ‘photosensitive epilepsy’ for more information.

Copyright (c) Jeremy P Bentham 2020. Please credit this blog if you use the information or software in it.
