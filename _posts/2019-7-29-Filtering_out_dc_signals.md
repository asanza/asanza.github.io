---
layout: post
title: 'Digitally remove a dc offset'
---

Last week I was working on a gadget to measure some power grid parameters. I was interested on the rms voltage and frequency. I started by writting a small routine for the microcontroller I was using, just to configure the analog digital converter and stream the sampled values to the uart. The values I got are shown in the figure bellow.

![original_signal]({{ site.baseurl }}/images/2019_7_29/original_signal.png)

As you can see, there is an small dc offset in the signal. This dc voltage is expected and is due to voltage offsets in the analog circuitry. These offsets are not really constant, but change at a rate of several parts per millon with the temperature. How much is the temperature change depends on the quality of the analog circuit. Good quality circuits are known for having an stability of a few ppm. Bad quality circuits can vary wildly.

An easy way to remove this offset is just to measure it and substract this value from all measurements. This has the advantage that works for all frequencies up to dc. The disadvantage of this method is that it does compensate for changes in the dc offset due to temperature. 

The other way is of course by using a high pass filter with a cut frequency that allows the signals of interest pass. Contrary to the other approach, it also filters out variations in the dc offset due temperature, but it has the disadvantage that it does not work with dc signals,and that is a little more complicated to implement. 

A digital filter is in reallity not that complicated. At its core is defined by the following equation:

```
a[0] * y[k] = b[0] * x[k] + b[1] * x[k-1] + b[2] * x[k-2] + ... + b[n] * x[k-n] 
                          - a[1] * y[k-1] - a[2] * y[k-2] - ... - a[n] * y[k-n] 
```

Where a[0], b[0], a[1], b[1] ... a[n], b[n] are the filter coefficients, and 
x[k], y[k], x[k-1], y[k-1] ... x[k-n], y[k-n] are the actual and past values of
the input signal (x) and the output signal(y).

If the coefficients a[1], a[2], a[3] ... a[n] are zero, the filter is called a
FIR (finite impulse response) filter. If not zero, then the filter is called an 
IIR (infinite impuse response) filter. There is a lot of interesting properties 
of fir and iir filters, but for now is enough to know that for a given frequency
response, iir filters reach better performance with less coefficients. This means
that you need to perform less sums and multiplications, which is good in a
microcontroller.

I decided to use the iir filter, because I have other things that I want to do
and don't want to use all the cpu cycles just on the filter. So now the task was
to find the filter coefficients that best match the frequency response I wanted.

There are several ways to find the filter coefficients, and proper filter design
is an art of finding the best tradeoffs between frequency response, phase delay, 
ripple, attenuation and so on. But I would say the vast majority of situations, 
like mine, you don't have really high spectations and just can manage with a simple
filter.

For these situations, (and complexer ones) there are python and its library for
digital signal processing. The script bellow calculates the filter coefficients
for an small filter with three coefficients. 

```python
import numpy as np
import matplotlib.pyplot as plt
from scipy.signal import lfilter, firwin, freqz, iirfilter, freqs, chirp

# Parameter needed for the filter design
fs = 1000.0           # Sampling frequency in Hz
fc = 4                # Cut frequency of the filter
N  = 2                # Filter order


# Here starts the filter calculations.
# First calculate the filter cut frequency in radians
Wn = [2*np.pi*fc]

# We create a simple test signal (just to have some visuals on the working of the filter)
samples = 1024 + 64    
t = np.arange(0, samples/fs, 1/fs)
signal = -650 + np.cos(2*np.pi*50*t) # + 0.8*np.cos(2*np.pi*10*t) + 0.5*np.sin(2*np.pi*1000*t)

# This is the magic stuff. Call iirfilter function to design the filter.
# Look in the iirfilter documentatio for details
b, a = iirfilter(N, Wn, rp=0.01, rs=40, btype='highpass', analog=False, ftype='ellip', fs=fs)

# Because we want the filter to work in an embedded system,
# we scale the coeficients to fit into an int16_t
b *= 2**15
a *= 2**15

b = np.round(b)
a = np.round(a)

# print calculated filter coeficients
print("Filter Coeficients")
print("b: " + str(b))
print("a: " + str(a))

# Draw the frequency response of the filter
w, h = freqz(b, a, 1500, fs=fs)

fig, ax1 = plt.subplots()
ax1.semilogx(w / (2*np.pi), 20 * np.log10(np.maximum(abs(h), 1e-5)))
ax1.set_xlabel('Frequency [Hz]')
# Make the y-axis label, ticks and tick labels match the line color.
ax1.set_ylabel('Amplitude [dB]', color='b')
ax1.tick_params('y', colors='b')
ax2 = ax1.twinx()
ax2.semilogx(w/(2*np.pi), np.angle(h) * 180 / np.pi)
ax2.set_ylabel('Phase [degrees]', color='r')
ax2.tick_params('y', colors='r')
fig.tight_layout()
ax1.grid(which='both', axis='both')
plt.show()

# Filter the test signal and plot it.
filtered_signal = lfilter(b, a, signal)

plt.plot(t[1024:len(signal)], signal[1024:len(signal)])
plt.ylabel('Amplitude')
plt.xlabel('Time [s]')
plt.grid(which='both', axis='both')
plt.title('Original signal')
plt.show()

plt.plot(t[1024:len(filtered_signal)], filtered_signal[1024:len(filtered_signal)])
plt.ylabel('Amplitude')
plt.xlabel('Time [s]')
plt.grid(which='both', axis='both')
plt.title('Filtered signal')
plt.show()
```


```c
#include <stdint.h>
#include <string.h>
#include <stdio.h>

/* storage for old values of the input signal */
static int32_t regx[3] = {0};
/* storage for old values of the filtered signal */
static int32_t regy[3] = {0};

/* Resets the filter state */
void iir_init( void )
{
	memset(regx, 0, sizeof(regx));
	memset(regy, 0, sizeof(regx));
}

/* apply the filter */
void iir_apply(int32_t* a, int32_t* b, int32_t* x, int size)
{
  int64_t y;
  
  for( int i = 0; i < size; i++ )
  {
    regx[2] = regx[1];
    regx[1] = regx[0];
    regx[0] = x[i];

    regy[2] = regy[1];
    regy[1] = regy[0];

    y  = (int64_t) x[i] * b[0] + (int64_t) regx[1] * b[1] + (int64_t) regx[2] * b[2]; 
    y = y - (int64_t) regy[1] * a[1] - (int64_t) regy[2] * a[2];
    y = ct/a[0];

    x[i] = (int32_t) y;

    regy[0] = x[i];
  }
}
```