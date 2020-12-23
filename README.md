# CQT-for-dummies
It took me weeks to understand a little bit how a Constant Q Transform works. Even with various parts of sourcecode I didn't see "it".
Here-by a journey to dissect the algorithm and it's inner workings.

# Sound
* Sound is "changes in the air-pressure". 
* Faster changes in the air-pressure gives higher frequency sound.
* Higher changes of air-pressure gives louder sounds.
When looking at speakers one can see the cone moving in and out, thus creating the changes in air-pressure.

# Microphone, taking a sample
A microphone is a device which measures the air-pressure and converts this to a value.
There are analog-microphones which convert the measured air-pressure into a voltage. 
There are digital-microphones which convert the measured air-pressure into a value which can be read by a computer.

# The samplerate
To get a computer to listen to sound one must take more samples of the values measured by the microphone. To get a more accurate representation of a sound, it's needed to take more samples. It's often important to take samples at a regular interval to be able to interpret the set of samples.
The interval is the time between taking the samples. One can also express this interval as a frequency (example 10 times per second, 4 times per minute).
Quite often the frequency of taking samples is expressed in the unit "Herz" and named "samplerate" or "samplefrequency". The unit "Herz" is "times per second". 

The samplerate is important as this influences the upper range of frequencies which are present in the sampled signal. 
Most digital music are broadcasted at a samplerate of 44.100 Herz or 48.000 Herz. The highest frequency of sound is halve of the used samplerate, thus 44100Herz->22050Herz. This is expressed as the Nyquest-frequency. The two halves are needed to reflect the changes in air-pressure (high->low). 

# The sample buffer
When the aim is to process the samples at the same time when the micrphone is doing it's measurements, a buffer is often used. This buffer is a piece of memory where taken samples are stored consecutively. Applications should only use memory which they actually need. As the samplerate defines the upper range of frequencies in a sampled signal, the "buffer-size and samplerate" determines the lower range of frequencies which can be identified in a sampled signal in the buffer.
When using a samplerate of 44100Herz and a buffer-size of 44100 samples, the range is from 1Herz to 22050Herz.
When using a samplerate of 44100Herz and a buffer-size of 1024 samples, the range is from 21Herz to 22050Herz.
When using a samplerate of 10000Herz and a buffer-size of 1024 samples, the range is from 48Herz to 5000Herz.
This buffer-size is needed to contain the changes in air-pressure (high->low).

* Consequence of too low samplerate with higher frequency sound. (distortion)
* Filters

