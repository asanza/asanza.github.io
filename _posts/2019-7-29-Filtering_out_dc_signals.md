---
layout: post
title: 'Digitally remove a dc offset'
---

Last week I was working on a gadget to measure some power grid parameters. I was interested on the rms voltage and frequency. I started by writting a small routine for the microcontroller I was using, just to configure the analog digital converter and stream the sampled values to the uart. The values I got are shown in the figure bellow.

![original_signal]({{ site.baseurl }}/images/2019_7_29/original_signal.png)

As you can see, there is an small dc offset in the signal. This dc voltage is expected and is due to voltage offsets in the analog circuitry. These offsets are not really constant, but change at a rate of several parts per millon with the temperature. How much is the temperature change depends on the quality of the analog circuit. Good quality circuits are known for having an stability of a few ppm. Bad quality circuits can vary wildly.

An easy way to remove this offset is just to measure it and substract this value from all measurements. This has the advantage that works for all frequencies up to dc. The disadvantage of this method is that it does compensate for changes in the dc offset due to temperature. 

The other way is of course by using a high pass filter with a cut frequency that allows the signals of interest pass. Contrary to the other approach, it also filters out variations in the dc offset due temperature, but it has the disadvantage that it does not work with dc signals,and that is a little more complicated to implement. 

A digital filter is in reallity not that complicated. 

```
a[0] * y[k] = b[0] * x[k] + b[1] * x[k-1] + b[2] * x[k-2] + ... + b[n] * x[k-n] 
                   - a[1] * y[k-1] - a[2] * y[k-2] - ... - a[n] * y[k-n] 
```


