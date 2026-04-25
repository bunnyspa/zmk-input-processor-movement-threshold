# zmk-input-processor-threshold

A [ZMK](https://zmk.dev) input processor that filters pointing device output until enough movement has accumulated to confirm intentional input. Accidental contact — a finger resting on or brushing the device — is silently discarded. Once the accumulated movement crosses the threshold, normal output resumes until the device goes idle.

## Getting Started

Assuming that you are using the [zmk-pmw3610-driver](https://github.com/badjeff/zmk-pmw3610-driver).

### `config/west.yml`

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    # --- copy from here ---
    - name: bunnyspa
      url-base: https://github.com/bunnyspa
    # --- to here ---
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: main
      import: app/west.yml
    # --- copy from here ---
    - name: zmk-input-processor-threshold
      remote: bunnyspa
      revision: main
    # --- to here ---
  self:
    path: config
```

### `<keyboard>.conf`

```ini
CONFIG_ZMK_POINTING=y
CONFIG_ZMK_INPUT_PROCESSOR_THRESHOLD=y
```

### `<keyboard>.overlay` / `<keyboard>.dtsi`

```c
#include <input/processors/threshold.dtsi>

/ {
    trackball_listener {
        compatible = "zmk,input-listener";
        device = <&trackball>;
        input-processors = <&threshold 100 2000>;
    };
};
```

If `trackball_listener` is already defined by your board or shield (e.g. in a file you don't own and can't edit directly), override it instead:

```c
#include <input/processors/threshold.dtsi>

&trackball_listener {
    input-processors = <&threshold 100 2000>;
};
```

## Parameters

`<&threshold param1 param2>`

**`param1` — threshold (raw sensor counts)**

The values below are calculated and may not reflect real-world sensor behavior. Use them as a starting point and adjust to feel.

<table>
<thead>
<tr><th rowspan="2">Sensor CPI</th><th colspan="3">34mm trackball</th><th colspan="3">40mm trackball</th></tr>
<tr><th>5°</th><th>10°</th><th>15°</th><th>5°</th><th>10°</th><th>15°</th></tr>
</thead>
<tbody>
<tr><td>400</td><td>23</td><td>47</td><td>70</td><td>27</td><td>55</td><td>82</td></tr>
<tr><td>600</td><td>35</td><td>70</td><td>105</td><td>41</td><td>82</td><td>124</td></tr>
<tr><td>800</td><td>47</td><td>93</td><td>140</td><td>55</td><td>110</td><td>165</td></tr>
<tr><td>1200</td><td>70</td><td>140</td><td>210</td><td>82</td><td>165</td><td>247</td></tr>
<tr><td>1600</td><td>93</td><td>187</td><td>280</td><td>110</td><td>220</td><td>330</td></tr>
</tbody>
</table>

**`param2` — idle-ms**

How long (in milliseconds) the device must be still before the filter resets. If the filter resets while you are still moving, increase it. If accidental touches are not filtered after lifting your hand, decrease it.

## Chaining with other processors

Place `threshold` first. Processors after it only receive events once intentional movement is confirmed — accidental contact is silently dropped before it reaches them.

If you need a processor to activate on any touch regardless of the threshold filter, place it before `threshold`.

```c
input-processors = <
    // processors here see all events, including accidental contact
    &threshold 100 2000
    // processors here only see intentional movement
>;
```

A common setup with `zip_temp_layer`, where layer 2 is a mouse click layer that activates only during intentional movement:

```c
input-processors = <
    &threshold 100 2000
    &zip_temp_layer 2 2000
    &zip_xy_scaler 1 2
>;
```

## Trackpad

This processor has not been tested with a trackpad. It may not work as expected — features like tap-to-click could be affected, since touch events that don't produce enough movement would be silently dropped before they reach other processors.

## How it works

Every REL X/Y input event increments an internal accumulator by `|value|`. While the accumulator is below the threshold, all events are blocked. Once the threshold is crossed, events flow through normally. After the device is idle for `idle-ms` milliseconds, the accumulator resets and filtering resumes.
