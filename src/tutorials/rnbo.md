# Using Faust in RNBO with codebox~

In this tutorial, we present how [Faust](https://faust.grame.fr) can be used with [RNBO](https://rnbo.cycling74.com), a library and toolchain that can take Max-like patches, export them as portable code, and directly compile that code to targets like a VST, a Max External, or a Raspberry Pi. DSP programs can be compiled to the internal [codebox~](https://rnbo.cycling74.com/codebox) sample level scripting language.
Compiling Faust DSP to codebox~ code will allow to take profit of hundreds of DSP building blocks implemented in the [Faust Libraries](https://faustlibraries.grame.fr), ready to use [Examples](https://faustdoc.grame.fr/examples/ambisonics/), any DSP program developed in more than 200 projects listed in the [Powered By Faust](https://faust.grame.fr/community/powered-by-faust/) page, or Faust DSP programs found on the net.

#### Who is this tutorial for?

The [first section](#using-command-line-tools) assumes a working [Faust](https://github.com/grame-cncm/faust) compiler installed on the machine, so is more designed for regular Faust users. The [second section](#using-the-faust-web-ide) is better suited for RNBO users who want to discover Faust [TODO].  

## Using command line tools

### Generating codebox~ code

Assuming you've [installed the **faust** compiler](https://faust.grame.fr/downloads/), starting from the following DSP **osc.dsp** program:

<!-- faust-run -->
```
import("stdfaust.lib");

vol = hslider("volume [unit:dB]", 0, -96, 0, 0.1) : ba.db2linear : si.smoo;
freq1 = hslider("freq1 [unit:Hz]", 1000, 20, 3000, 1);
freq2 = hslider("freq2 [unit:Hz]", 200, 20, 3000, 1);

process = vgroup("Oscillator", os.osc(freq1) * vol, os.osc(freq2) * vol);
```
<!-- /faust-run -->

The codebox~ code can be generated using the following line (note the use of `-double` option, the default sample format in RNBO/codebox~):

```bash
faust -lang codebox -double osc.dsp -o osc.codebox
```

This will generate a series of functions to init, update parameters and compute audio frames.

### Looking at the generated code

The generated code contains a sequence of parameters definitions with their min, max, step and default values:

```
// Params
@param({min: 2e+01, max: 3e+03, step: 0.1}) freq1 = 1e+03;
@param({min: 2e+01, max: 3e+03, step: 0.1}) freq2 = 2e+02;
@param({min: -96.0, max: 0.0, step: 0.1}) volume = 0.0;
```

Next the declaration of the DSP structure using the `@state` decorator, creating a state that persists across the lifetime of the codebox object. Scalar and arrays with the proper type are created:

```
// Fields
@state fSampleRate_cb : Int = 0;
@state fConst1_cb : number = 0;
@state fHslider0_cb : number = 0;
@state fConst2_cb : number = 0;
@state iVec0_cb = new FixedIntArray(2);
@state fRec0_cb = new FixedDoubleArray(2);
@state fConst3_cb : number = 0;
@state fHslider1_cb : number = 0;
@state fRec2_cb = new FixedDoubleArray(2);
@state fHslider2_cb : number = 0;
@state fRec3_cb = new FixedDoubleArray(2);
@state iVec1_cb = new FixedIntArray(2);
@state iRec1_cb = new FixedIntArray(2);
@state ftbl0mydspSIG0_cb = new FixedDoubleArray(65536);
@state fUpdated : Int = 0;
@state fControl_cb = new FixedDoubleArray(3);
```

Next the DSP init code, which is added in [dspsetup](https://rnbo.cycling74.com/codebox#special-functions), only available in codebox~ where it will be called each time audio is turned on in Max (which is basically every time the audio state is toggled, or the sample rate or vector size is changed). Here the DSP state is initialized using the RNBO current sample rate: 

```
// Init
function dspsetup() {
	fUpdated = true;
	for (let l2_re0_cb : Int = 0; (l2_re0_cb < 2); l2_re0_cb = (l2_re0_cb + 1)) {
		iVec1_cb[l2_re0_cb] = 0;
	}
	for (let l3_re0_cb : Int = 0; (l3_re0_cb < 2); l3_re0_cb = (l3_re0_cb + 1)) {
		iRec1_cb[l3_re0_cb] = 0;
	}
	for (let i1_re0_cb : Int = 0; (i1_re0_cb < 65536); i1_re0_cb = (i1_re0_cb + 1)) {
		iVec1_cb[0] = 1;
		iRec1_cb[0] = ((iVec1_cb[1] + iRec1_cb[1]) % 65536);
		ftbl0mydspSIG0_cb[i1_re0_cb] = sin((9.587379924285257e-05 * iRec1_cb[0]));
		iVec1_cb[1] = iVec1_cb[0];
		iRec1_cb[1] = iRec1_cb[0];
	}
	fHslider0_cb = 0.0;
	fHslider1_cb = 1e+03;
	fHslider2_cb = 2e+02;
	...
    ...   
	fSampleRate_cb = samplerate();
	let fConst0_cb : number = min(1.92e+05, max(1.0, fSampleRate_cb));
	fConst1_cb = (44.1 / fConst0_cb);
	fConst2_cb = (1.0 - fConst1_cb);
	fConst3_cb = (1.0 / fConst0_cb);
}
```

Parameters handling is separated in two functions: `control` is called each time a parameter has changed:

```
// Control
function control() {
	fControl_cb[0] = (fConst1_cb * pow(1e+01, (0.05 * fHslider0_cb)));
	fControl_cb[1] = (fConst3_cb * fHslider1_cb);
	fControl_cb[2] = (fConst3_cb * fHslider2_cb);
}
```

And the actual change is triggered when at least one parameter has changed, controlled by the state of `fUpdated`global variable:

```
// Update parameters
function update(freq1,freq2,volume) {
	fUpdated = int(fUpdated) | (freq1 != fHslider1_cb); fHslider1_cb = freq1;
	fUpdated = int(fUpdated) | (freq2 != fHslider2_cb); fHslider2_cb = freq2;
	fUpdated = int(fUpdated) | (volume != fHslider0_cb); fHslider0_cb = volume;
	if (fUpdated) { fUpdated = false; control(); }
}
```

Finally `compute` process the audio inputs and produces audio ouputs:

```
// Update parameters
/ Compute one frame
function compute() {
	iVec0_cb[0] = 1;
	fRec0_cb[0] = (fControl_cb[0] + (fConst2_cb * fRec0_cb[1]));
	let iTemp0_cb : Int = (1 - iVec0_cb[1]);
	let fTemp1_cb : number = ((iTemp0_cb) ? 0.0 : (fControl_cb[1] + fRec2_cb[1]));
	fRec2_cb[0] = (fTemp1_cb - floor(fTemp1_cb));
	output0_cb = (fRec0_cb[0] * ftbl0mydspSIG0_cb[max(0, min(int((65536.0 * fRec2_cb[0])), 65535))]);
	let fTemp2_cb : number = ((iTemp0_cb) ? 0.0 : (fControl_cb[2] + fRec3_cb[1]));
	fRec3_cb[0] = (fTemp2_cb - floor(fTemp2_cb));
	output1_cb = (fRec0_cb[0] * ftbl0mydspSIG0_cb[max(0, min(int((65536.0 * fRec3_cb[0])), 65535))]);
	iVec0_cb[1] = iVec0_cb[0];
	fRec0_cb[1] = fRec0_cb[0];
	fRec2_cb[1] = fRec2_cb[0];
	fRec3_cb[1] = fRec3_cb[0];
	return [output0_cb,output1_cb];
}
```

With this code in place, the following sequence of operations is done at each sample:

```
// Update parameters
update(freq1,freq2,volume);
// Compute one frame
outputs = compute();
// Write the outputs
out1 = outputs[0];
out2 = outputs[1];
```

Note that the generated code uses the so-called [scalar code generation model](https://faustdoc.grame.fr/manual/compiler/#structure-of-the-generated-code), the default one, where the compiled sample generation code is done in `compute`. 

### Testing the generated codebox~ code

To be tested, the generated code has to be pasted in a codebox~ component in an englobing RNBO patch.

### Known issues

This is a [Work In Progress] and the generated code does not always work as expected:

 - parameters handing is not yet fully functioning, in particular when DSP programs only outputting audio are compiled
 - beware, some DSP produces incorrect audio samples !

## Using the Faust Web IDE [TODO]
