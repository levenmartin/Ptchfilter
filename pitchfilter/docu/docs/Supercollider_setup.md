
## Supercollider
/**
* Diese Funktion erstellt und spielt einen Synthesizer-Sound mit verschiedenen Filtern.<br>
*
* @param {float} bandwidth - Die Bandbreite des Bandpassfilters.<br>
* @param {float} sig - Das Signal was welches vom Master_node an alle gesendet wird<br>
* @param {float} input - Das Input Signal, welches die Frquenz für den Bandpassfilter gibt.<br>
*  @param {float} amp - Die Amplitude des Signals vom eigenem Input, damit nur ein Sound erzeugt wird wenn man auch einen Input gibt.<br>
* @return {Synth} - Ein Synthesizer-Objekt, das den erzeugten Sound abspielt.<br>
```
{
	var sig;
	var sound;
	var input;
	var bandwidth = 0.1; 
	var freq;
	var amp;

	sig = SoundIn.ar(1);
	input = SoundIn.ar([0]); 
    freq = Pitch.kr(input, ampThreshold: 0.02, median: 7).cpsmidi.round(1).midicps;
	freq = freq.clip(100, 20000);
	amp = Amplitude.ar(input, mul:10.0);
    // Bandpassfilter mit dynamischer Frequenz einstellen

    sound = BPF.ar(sig, freq, bandwidth);
	sound = HPF.ar(sound,40);
	sound = LPF.ar(sound,20000);
    Out.ar(0, sound*amp);
}.play;
```

Der Hoch-und Tiefpassfilter haben den Zweck Störgeräusche zu verhindern, falls das Signal vom Input zu groß oder Klein wird.
```
sound = HPF.ar(sound,40);
sound = LPF.ar(sound,20000);

```

Hier wird die Tonhöhe des Inputsignals berechnet und in Hz umgewandelt, sowie ein Minimum und Maximum dafür festgelegt.

```
freq = Pitch.kr(input, ampThreshold: 0.02, median: 7).cpsmidi.round(1).midicps;
freq = freq.clip(100, 20000);
```