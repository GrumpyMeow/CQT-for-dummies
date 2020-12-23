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
* Window-functions


# Code 1
I had found this code at [Ledcube](https://bitbucket.org/JoD/ledcube/src/master/frequency_printer.pde) for a Pinguino. The code is very optimized and runs very fast.

This code uses integer-calculations which significatly improves the performance.

This code uses two buffers: a normal buffer and a 8-times-downsampled-buffer. This technique reduces the need for a large buffer-size.

This code uses dividers to skip over samples: Div[k]. 

With pure sine-waves i do see that this implementation works. But with more complex sounds i don't find the results very clear.
My guess is that because of the "skipping over samples" lower frequency harmonics appear.

```
fixedpoint hamming(int m, int k) {
    return ALPHA - (BETA * approxCos(int2PI * m / NFreq[k]) >> PRECISION);
}

void cqt() {
    unsigned int k, i, indx;
    int windowed, angle;
    float real_f, imag_f;
    fixedpoint real, imag;
    for (k = 0; k < lowfreq_endIndex; ++k) {
	indx = nrInterrupts % NRSAMPLES - 1 + 8 * NRSAMPLES;
	real = ALPHA - (BETA * signal_lowfreq[indx % NRSAMPLES] >> PRECISION);
	imag = 0;
	for (i = 1; i < NFreq[k]; ++i) {
	    windowed = hamming(i, k) * signal_lowfreq[(indx - i * Div[k]) % NRSAMPLES];
	    angle = twoPiQ * i / NFreq[k];
	    real += windowed * approxCos(angle) >> PRECISION;
	    imag += windowed * approxSin(angle) >> PRECISION;
	}

	real_f = real / (float) SCALE;
	imag_f = imag / (float) SCALE;
	freqs[k] = logf(powf(real_f * real_f + imag_f * imag_f, 0.5) / NFreq[k] + 0.1) * amplitude / 32;
    }
    for (; k < FREQS; ++k) {
	indx = nrInterrupts % NRSAMPLES - 1 + 8 * NRSAMPLES;
	real = ALPHA - (BETA * signal[indx % NRSAMPLES] >> PRECISION);
	imag = 0;
	for (i = 1; i < NFreq[k]; ++i) {
	    windowed = hamming(i, k) * signal[(indx - i * Div[k]) % NRSAMPLES];
	    angle = twoPiQ * i / NFreq[k];
	    real += windowed * approxCos(angle) >> PRECISION;
	    imag += windowed * approxSin(angle) >> PRECISION;
	}
	real_f = real / (float) SCALE;
	imag_f = imag / (float) SCALE;
	freqs[k] = logf(powf(real_f * real_f + imag_f * imag_f, 0.5) / NFreq[k] + 0.1) * amplitude / 32;
    }
}

void preprocess_filters() {
    //CDC.printf("calculating twoPiQ...\n");
    float nn = powf(2, log(F16 / (double) F0) / log(2) / 16.0);
    twoPiQ = int2PI * (nn + 1) / (nn - 1) / 2;

    int i;
    //printf("calculating the border frequencies...\n");
    for (i = 0; i < FREQS + 1; ++i) {
	Freq[i] = (F0 * powf(nn, i) + F16 / powf(nn, FREQS - i)) / 2;
    }

    //CDC.printf("calculating the index until which the signal_lowfreq samples should be used\n");
    lowfreq_endIndex = 0;
    while (FSAMPLE / (Freq[lowfreq_endIndex + 1] - Freq[lowfreq_endIndex]) >= NRSAMPLES && lowfreq_endIndex < FREQS) {
	++lowfreq_endIndex;
    }

    int samplesLeft = MAXTOTALSAMPLES;

    //CDC.printf("calculating sample frequency dividers...\n");
    for (i = 16; i > lowfreq_endIndex; --i) {
	Div[i - 1] = 1 + FSAMPLE / ((Freq[i] - Freq[i - 1]) * samplesLeft / i);
	NFreq[i - 1] = FSAMPLE / (Freq[i] - Freq[i - 1]) / Div[i - 1];
	samplesLeft -= NFreq[i - 1];
    }
    for (; i > 0; --i) {
	Div[i - 1] = 1 + FSAMPLE / LOWFREQDIV / ((Freq[i] - Freq[i - 1]) * samplesLeft / i);
	NFreq[i - 1] = FSAMPLE / LOWFREQDIV / (Freq[i] - Freq[i - 1]) / Div[i - 1];
	samplesLeft -= NFreq[i - 1];
    }
}
```

