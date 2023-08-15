# SuperDirt

SuperCollider implementation of the Dirt sampler, originally designed
for the [TidalCycles](https://github.com/tidalcycles/tidal)
environment. SuperDirt is a general purpose framework for playing
samples and synths, controllable over the Open Sound Control protocol,
and locally from the SuperCollider language.

(C) 2015-2020 Julian Rohrhuber, Alex McLean and contributors

SuperDirt is free software: you can redistribute it and/or modify it
under the terms of the GNU General Public License as published by the
Free Software Foundation, either version 2 of the License, or (at your
option) any later version.

This program is distributed in the hope that it will be useful, but
WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
General Public License for more details.

You should have received a copy of the GNU General Public License
along with this library.  If not, see <http://www.gnu.org/licenses/>.

## Requirements

* SuperCollider >= v3.7 (3.6 possible, but see below): https://github.com/supercollider/supercollider
* The Vowel Quark: https://github.com/supercollider-quarks/Vowel
* optional, but recommended (many effect UGens need it): sc3-plugins: https://github.com/supercollider/sc3-plugins/
* For proper usage you need https://github.com/tidalcycles/Tidal

## Installation from SuperCollider
```
Quarks.install("https://github.com/mynkit/SuperDirt.git");
```
Note: this also automatically installs the DirtSamples quark, which contains a large collection of sound files. It downloads them as a zip file. Sometimes, git fails to unpack these samples and they don't get listed. In this case, you have to unpack them "manually".

## Custom

### add effects (distortion, HRTF, etc..)

Add the effect name and arguments in `synths/core-modules.scd`.

```diff_supercollider
+~dirt.addModule('HRTF',
+  { |dirtEvent|
+		dirtEvent.sendSynth('HRTF' ++ numChannels,
+			[
+				theta: ~theta,
+				dis: ~dis,
+				out: ~out
+    ])
+}, { ~theta.notNil });
```

Add the definition of the effect in `synths/core-synths.scd`.

```diff_supercollider
+  ~decoder = FoaDecoderMatrix.newStereo((100).degrad, (3-sqrt(3))/2);
+  SynthDef("HRTF" ++ numChannels, { |out, theta, dis|
+		var signal, in, left, right, t1, t2, t3, t4, soto, naka, farRate, foa;
+		switch(numChannels,
+			2, {
+				in = In.ar(out, numChannels).asArray.sum;
+				// theta is our angle on the X-Y plane and phi is our elevation
+				theta = (theta-1) * pi;
+				// Encode into our foa signal
+				foa = FoaPanB.ar(in, theta, 0);
+				// decode our signal using our decoder defined above
+				signal = FoaDecode.ar(foa, ~decoder);
+				/*soto = 1 - ((1-dis)*(1-dis));
+				naka = (1-dis)*(1-dis);
+				signal = [
+				(soto*signal[0]) + (naka*in*0.95),
+				(soto*signal[1]) + (naka*in*0.95)
+				];*/
+				ReplaceOut.ar(out, signal)
+			},
+			4, {
+				in = In.ar(out, numChannels).asArray.sum;
+				// disが負の場合は180度
+				theta = if(dis<0, theta+1, theta);
+				theta = theta % 2;
+				theta = if(theta>1, theta-2, theta);
+				t1 = (theta+0.25).abs;
+				t2 = (theta-0.25).abs;
+				t3 = (theta+0.75).abs;
+				t4 = (theta-0.75).abs;
+				t3 = min(t3, 2-t3);
+				t4 = min(t4, 2-t4);
+				dis = dis.abs;
+				farRate = if(dis>1, dis, 1);
+				dis = if(dis>1, 1, dis);
+				signal = [
+					if(t1<=0.5, in*(1 - (2*t1)), 0),
+					if(t2<=0.5, in*(1 - (2*t2)), 0),
+					if(t3<=0.5, in*(1 - (2*t3)), 0),
+					if(t4<=0.5, in*(1 - (2*t4)), 0),
+				];
+				soto = 1 - ((1-dis)*(1-dis));
+				naka = (1-dis)*(1-dis);
+				signal = [
+					(soto*signal[0]) + (naka*in*0.95),
+					(soto*signal[1]) + (naka*in*0.95),
+					(soto*signal[2]) + (naka*in*0.95),
+					(soto*signal[3]) + (naka*in*0.95)
+				];
+				ReplaceOut.ar(out, signal)
+			},{
+				// それ以外のチャンネル数のときはなにもせずに返す
+				in = In.ar(out, numChannels);
+				signal = in;
+				ReplaceOut.ar(out, signal)
+			}
+		)
```

### add global effects (reverb, delay, etc..)

Add the effect name and arguments to variable `this.globalEffects` in `classes/DirtOrbit.sc`.

```diff_supercollider
	initDefaultGlobalEffects {
		this.globalEffects = [
			// all global effects sleep when the input is quiet for long enough and no parameters are set.
			GlobalDirtEffect(\dirt_delay, [\delaytime, \delayfeedback, \delaySend, \delayAmp, \lock, \cps]),
			GlobalDirtEffect(\dirt_reverb, [\size, \room, \dry]),
+			GlobalDirtEffect(\schroeder_reverb, [\scReverb, \ice, \damp]),
			GlobalDirtEffect(\dirt_leslie, [\leslie, \lrate, \lsize]),
			GlobalDirtEffect(\dirt_rms, [\rmsReplyRate, \rmsPeakLag]).alwaysRun_(true),
			GlobalDirtEffect(\dirt_monitor, [\limitertype]).alwaysRun_(true),
		]
	}
```

Add the definition of the effect in `synths/core-synths-global.scd`.

```diff_supercollider
+	SynthDef("schroeder_reverb" ++ numChannels, { |dryBus, effectBus, scReverb, ice, damp|
+		var signal = In.ar(dryBus, numChannels);
+		var chain, in, z, y, oct, gate = 1;
+
+		z = DelayN.ar(signal, 0.048);
+		y = Mix.ar(Array.fill(7,{ CombL.ar(z, 0.1, 1, 15) }));
+		// 32.do({ y = AllpassN.ar(y, 0.050, [0.050.rand, 0.050.rand], 1) });
+		32.do({ y = AllpassN.ar(y, 0.02, [0.02.rand, 0.02.rand], 1) });
+		oct = 1.0 * LeakDC.ar( abs(y) );
+		y = SelectX.ar(ice, [y, ice * oct, DC.ar(0)]);
+		signal = ((1-damp)*signal) + (0.2*y*scReverb);
+
+		signal = signal * EnvGen.kr(Env.asr, gate, doneAction:2);
+
+		DirtPause.ar(signal, graceTime:4);
+
+		Out.ar(effectBus, signal);
+	}, [\ir, \ir]).add;
```

## Simple Setup

`SuperDirt.start`

You can pass `port`, `outBusses`, `senderAddr` as arguments.

## Setup with options

For an example startup file, see the file `superdirt_startup.scd`. you can `load(<path>)` this from the SuperCollider startup file.

## Automatic startup
If you want SuperDirt to start automatically, you can load it from the startup file. To do this, open the sc startup file (```File>Open startup file```) and add: ```load("... path to your tidal startup file ...")```. This path you can get by dropping the file onto the text editor.


## Options on startup
- `numChannels` can be set to anything your soundcard supports
- for server options, see `ServerOptions` helpfile: http://doc.sccode.org/Classes/ServerOptions.html

## Options on-the-fly
- add sound files. `~dirt.loadSoundFiles("path/to/my/samples/*")` You can drag and drop folders into the editor and add a wildcard (*) after˘ it.
- you can pass the udp port on which superdirt is listenting and the output channel offsets: `~dirt.start(port, channels)`
- new orbits can be created on the fly (e.g. `~dirt.makeBusses([0, 0, 0])`).
- add or edit SynthDef files to add your own synthesis methods to be called from tidal: https://github.com/musikinformatik/SuperDirt/blob/master/synths/default-synths.scd
- you can live rewrite the core synths (but take care not to break them ...): https://github.com/musikinformatik/SuperDirt/blob/master/synths/core-synths.scd

## Trouble Shooting
 If you run into unspecific troubles and want to quickly reset everything, you can run the following: `SuperDirt.resetEverything`
You can minimize downtime if you have a startup file that automatically starts SuperDirt (see Automatic startup, above).


## Using SuperDirt with SuperCollider 3.6
It is in principle possible to use SuperCollider 3.6, but startup will be much slower by comparison. It is **not** recommended if you expect it to run smoothly.

For reference, we leave here the instructions if you want to try anyway:

The install works differently: don't do `include("SuperDirt")`, but instead download the three quarks to the SuperCollider `Extensions` folder:
- https://github.com/musikinformatik/SuperDirt
- https://github.com/tidalcycles/Dirt-Samples
- https://github.com/supercollider-quarks/Vowel

Note that for automatically loading the sound files, the folder `Dirt-Samples` should have this name (not Dirt-Samples-master e.g.) and should be next to the SuperDirt folder.
