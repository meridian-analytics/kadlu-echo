Welcome to `kadlu-echo`, a Python package for simulating the underwater propagation
of acoustic signals.

# Description

We use the [kadlu](https://meridian-analytics.github.io/kadlu/) package to calculate the 
transmission loss between the sound source and the receiver as a function of the 
sound frequency. Then, we take the Fourier Transform to obtain the response function 
in the time domain. Finally, we convolve this response function with the signal of 
interest to simulate the propagation effects of the underwater environment, which 
include reverberations and echos (due to reflections of the water surface and the 
seafloor) as well as refraction due to variations in sound speed with depth.

# Requirements

`kadlu-echo` requires [kadlu](https://meridian-analytics.github.io/kadlu/)>=2.3 and all its dependencies. 
The [example script](scripts/example_script.py) also requires [soundfile](https://pysoundfile.readthedocs.io/en/latest/) 
(for saving the waveforms to wav files) and [ketos](https://docs.meridian.cs.dal.ca/ketos/)
(for visualizing the spectrograms).


# Usage

Below, we demonstrate the key functionalities of the `kadlu-echo` package through a simple example.

We begin by computing the frequency-domain response of a simplified underwater acoustic environment with 
the following properties,

 * uniform water depth of 130 m
 * hard seafloor made of basalt
 * sound source situated 5 m below the surface
 * receiver situated 5 m above the seafloor at a distance of 1000 m from the source
 * uniform sound speed of 1480 m/s

Also, we choose to consider only frequencies below 500 Hz (the computation becomes increasingly 
slow for higher frequencies).

```python
from echo import echo

mag, phase, axes = echo.compute_freq_response(load_bathymetry=130, 
                        seafloor=echo.seafloor_dict['basalt'], ssp=1480,
                        source_depth=5, receiver_depth=125, receiver_distance=1000, 
                        freq_max=500, plot_freq=[200], plot_dir='.')
```

The numpy arrays `mag` and `phase` contain the (linear) transmission loss and the complex phase angle, 
respectively, at each integer frequency values from 1 to 500 Hz. In addition, the call to this method 
produces a plot of the transmission loss at 200 Hz, which looks like this,

![Transmission Loss](res/TL_5m_200Hz.png)

Since the transmission loss computation can be rather slow, it can be useful to save the outputs for later
use. For this purpose, `kadlu-echo` provides the methods `save_freq_response` and `load_freq_response`, as 
well as `save_time_response`.

Next, we transform the frequency response to the time domain as follows,

```python
freq = axes['frequency']

mag = np.squeeze(mag)
phase = np.squeeze(phase)

# transform
time, resp = echo.freq2time(freq=freq, mag=mag, phase=phase, window_len=3.0)

# interpolate
fresp = echo.interp_time_response(time, resp)
```

The time-domain response function thus obtained may be visualized as follows,
```python
import matplotlib.pyplot as plt

x = np.linspace(-1.0, 1.0, 2000)
y = fresp(x)
plt.plot(x, y)
plt.xlabel('Time (s)')
plt.title('Response function of the underwater acoustic environment')
plt.show()
```
One clearly sees reverberations and distinct echos caused 
by reflections off the surface and the hard bottom.

![Time-domain response](res/response.png)

Finally, we create synthetic NARW upcall and convolve it with the response function 
to simulate the effects of the underwater sound propagation,

```python
from upcall import upcall

#create the upcall
wf, sr = upcall()

#convolve with the response funciton
wf_new = echo.convolve(signal=wf, rate=sr, response_function=fresp, conv_siz=3.0)

# add some white noise to the waveforms
wf += np.random.normal(loc=0, scale=0.1*np.std(wf), size=len(wf))
wf_new += np.random.normal(loc=0, scale=0.1*np.std(wf_new), size=len(wf_new))

# plot the waveforms
x = np.arange(len(wf)) / sr
plt.plot(x, wf, label='Original')
plt.plot(x, wf_new, label='New')
plt.xlabel('Time (s)')
plt.title('Sound propagation applied to a synthetic NARW upcall')
plt.legend()
plt.show()
```

![Waveforms](res/waveform.png)

If you have [ketos](https://docs.meridian.cs.dal.ca/ketos/) installed, you can also 
visualize the sound clips as spectrograms, like this

```python
from ketos.audio.waveform import Waveform
from ketos.audio.spectrogram import MagSpectrogram
from ketos.data_handling.parsing import load_audio_representation

spec_config = load_audio_representation('spec_config.json', name="spectrogram")

spec = MagSpectrogram.from_waveform(Waveform(wf, sr), **spec_config)
spec_new = MagSpectrogram.from_waveform(Waveform(wf_new, sr), **spec_config)

spec.plot()
spec_new.plot()
plt.show()
```

![Original upcall](res/spec_orig.png)

![Propagated upcall](res/spec_new.png)


