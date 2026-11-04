# NeuroPulse-Ambient: Proof-of-Concept Technical Report

*Single-channel wearable BCI for focus tracking | Scope: desk prototype only (Phase 1)*

## 1. Scope and Objectives

This report documents only the working proof-of-concept: hardware used, wiring and pin mapping between boards, ESP32-S3 firmware (.ino) structure, and the offline Python/MNE pipeline used to validate the signal. Custom PCB, enclosure and production costs are out of scope.

**POC goals**

1. Capture a single differential frontal biopotential channel at 256 samples/s.
2. Remove 50 Hz mains interference and motion/muscle artifacts.
3. Compute theta/alpha/beta band power with a real-time FFT on the ESP32-S3.
4. Compute Focus Score = Beta / (Alpha + Theta), show it on a 1.54 inch LCD and stream it over BLE.
5. Validate the same signal offline in Python (MNE) to confirm the on-device maths is correct.

## 2. POC Bill of Materials (Actual)

| Block | Part used | Cost (INR) | Notes |
| --- | --- | --- | --- |
| MCU | ESP32-S3 DevKit | 380 | Dual core, hardware FPU (single precision), BLE 5 |
| Analog front end | MAX30003 SPI breakout | 3000 | 18-bit biopotential ADC; includes gel electrodes and snap leads |
| AFE alternative (tested on bench) | AD8232 module | 400 | Analog output, sampled by ESP32 ADC; noisier, used only for comparison |
| Display | 1.54 inch 240x240 colour LCD (ST7789, SPI) | 500 |  |
| Electrodes | Gel electrodes + snap connectors | 0 | Bundled with the MAX30003 kit |
| Power | Regular USB powerbank | 300 | Battery power keeps the rig isolated from mains |

## 3. System Architecture (POC)

```
[Fp1 / Fp2 gel electrodes] + [Mastoid reference]
        |
        v
[MAX30003: input mux -> instrumentation amp -> 18-bit delta-sigma ADC -> digital HPF/LPF -> FIFO]
        | SPI (FSPI host) + INT1 (data ready)
        v
[ESP32-S3 Core 0: acquisition task -> IIR notch + band-pass -> ring buffer -> FFT -> band power -> Focus Score]
        | FreeRTOS queue (features)
        v
[ESP32-S3 Core 1: UI task (ST7789 on second SPI host) + BLE notify task]
        |
        v
[Tablet / phone app (BLE GATT notify)]
```

Design rule: the two SPI peripherals are on separate hosts (AFE on FSPI, display on HSPI), so a slow screen refresh never blocks AFE reads.

## 4. Electrode Montage and Analog Front End Setup

| Electrode | Site | Connects to |
| --- | --- | --- |
| Active 1 | Fp1 (left forehead, above the eyebrow) | ECGP |
| Active 2 | Fp2 (right forehead) | ECGN |
| Reference | Mastoid (behind the ear) | AFE reference / bias lead on the breakout |

Check the silkscreen on your exact MAX30003 breakout before wiring; lead names vary between vendors.

**Skin preparation (matters more than any filter):** clean with an alcohol wipe, let it dry, press gel electrodes firmly, and keep electrode-skin impedance under about 20 kOhm.

**MAX30003 configuration used**

| Parameter | Setting | Why |
| --- | --- | --- |
| ECG output rate | 256 sps | Matches the 256 point FFT, giving 1 Hz bins |
| PGA gain | 160 V/V (maximum) | EEG is tens of microvolts; maximum gain gives best resolution |
| Internal digital HPF | 0.5 Hz enabled | Removes electrode DC offset and drift |
| Internal digital LPF | 40 Hz | Cuts EMG and most mains energy before the MCU |
| Lead-off / R-to-R detection | Off | Not needed for EEG |

Registers touched over SPI: CNFG\_GEN (0x10, enable ECG channel), CNFG\_ECG (0x15, rate/gain/filters), CNFG\_EMUX (0x14, input mux, leave ECGP/ECGN connected), EN\_INT (0x02, route FIFO-ready to INT1), SYNCH (0x08) and FIFO\_RST (0x0A) to start cleanly, STATUS (0x01) to clear flags, ECG\_FIFO (0x21) for data. Confirm bit positions against the datasheet register tables before committing values.

**Sample to microvolt conversion** (datasheet formula, Vref = 1.0 V):

`uV = signed18(word >> 6) * 1e6 / (131072 * gain)`\
At gain 160 this is about 0.0477 uV per LSB.

SPI frame format: first byte = (register address << 1) | R/W (1 = read, 0 = write), followed by 24 data bits. FIFO word: bits 23:6 = signed sample, bits 5:3 = ETAG (sample validity: valid, valid end of frame, FIFO empty, FIFO overflow), bits 2:0 = PTAG.

## 5. Pin Mapping, Board to Board

Pins below are the ones used on an ESP32-S3-DevKitC. Avoid GPIO 26 to 37 where possible: on the PSRAM (N8R8) variants GPIO 35, 36 and 37 are reserved, and flash/PSRAM lines use others. Reassign if your devkit differs.

### 5.1 ESP32-S3 to MAX30003 (SPI mode 0, 1 MHz)

| MAX30003 breakout pin | ESP32-S3 GPIO | Function |
| --- | --- | --- |
| VCC | 3V3 | 3.3 V logic and supply |
| GND | GND | Common ground |
| SCLK | GPIO 12 | FSPI clock |
| SDO (MISO) | GPIO 13 | FSPI MISO |
| SDI (MOSI) | GPIO 11 | FSPI MOSI |
| CS | GPIO 10 | Chip select (active low) |
| INT1 (INTB) | GPIO 4 | Data ready, falling edge interrupt |
| INT2 (INT2B) | not used |  |
| FCLK | on-board crystal, otherwise GPIO 5 | The chip needs 32.768 kHz. If the breakout has no crystal, generate it with LEDC (frequency 32768, 50% duty) |

### 5.2 ESP32-S3 to 1.54 inch ST7789 LCD (SPI, 40 MHz)

| LCD pin | ESP32-S3 GPIO | Function |
| --- | --- | --- |
| VCC | 3V3 | Power |
| GND | GND | Ground |
| SCL (SCK) | GPIO 18 | HSPI clock |
| SDA (MOSI) | GPIO 17 | HSPI MOSI |
| RES | GPIO 7 | Reset |
| DC | GPIO 15 | Data/command |
| CS | GPIO 16 | Chip select |
| BLK | GPIO 6 | Backlight (PWM via LEDC for brightness) |

### 5.3 Power

Powerbank, USB to ESP32-S3 USB-C port, with the 3V3 rail feeding both modules. Run the rig on battery during recordings with the laptop USB data cable unplugged (or use a USB isolator) to cut mains-coupled 50 Hz.

## 6. Firmware (.ino) Details

### 6.1 Libraries

| Library | Use |
| --- | --- |
| SPI.h (built in) | Two SPIClass instances |
| arduinoFFT v2 (float) | 256 point FFT |
| TFT\_eSPI with USE\_HSPI\_PORT, or LovyanGFX | ST7789 drawing; pins set in User\_Setup.h |
| NimBLE-Arduino | BLE GATT server |
| FreeRTOS (built in) | Tasks pinned to cores, queues, semaphores |

### 6.2 Constants and globals

```cpp
#include <SPI.h>
#include <arduinoFFT.h>
#include <TFT_eSPI.h>
#include <NimBLEDevice.h>

#define PIN_SCK 12
#define PIN_MISO 13
#define PIN_MOSI 11
#define PIN_CS 10
#define PIN_INT1 4

const float FS = 256.0f;        // sample rate
const int   N_FFT = 256;        // 1 s window
const int   HOP = 128;          // 50% overlap -> 2 feature updates per second
const float GAIN = 160.0f;

SPIClass spiAFE(FSPI);
SemaphoreHandle_t drdySem;
QueueHandle_t featQ;

struct Features { float theta, alpha, beta, focus, focusSmooth; bool artifact; };
```

### 6.3 MAX30003 SPI driver

```cpp
uint32_t afeRead(uint8_t reg) {
  spiAFE.beginTransaction(SPISettings(1000000, MSBFIRST, SPI_MODE0));
  digitalWrite(PIN_CS, LOW);
  spiAFE.transfer((reg << 1) | 1);
  uint32_t v = (uint32_t)spiAFE.transfer(0) << 16;
  v |= (uint32_t)spiAFE.transfer(0) << 8;
  v |= spiAFE.transfer(0);
  digitalWrite(PIN_CS, HIGH);
  spiAFE.endTransaction();
  return v;
}

void afeWrite(uint8_t reg, uint32_t val) {
  spiAFE.beginTransaction(SPISettings(1000000, MSBFIRST, SPI_MODE0));
  digitalWrite(PIN_CS, LOW);
  spiAFE.transfer(reg << 1);
  spiAFE.transfer((val >> 16) & 0xFF);
  spiAFE.transfer((val >> 8) & 0xFF);
  spiAFE.transfer(val & 0xFF);
  digitalWrite(PIN_CS, HIGH);
  spiAFE.endTransaction();
}

void afeBegin() {
  afeWrite(0x00, 0);            // SW_RST
  delay(100);
  afeWrite(0x10, CNFG_GEN_VAL); // enable ECG channel
  afeWrite(0x15, CNFG_ECG_VAL); // 256 sps, gain 160, HPF 0.5 Hz, LPF 40 Hz
  afeWrite(0x14, CNFG_EMUX_VAL);// inputs connected
  afeWrite(0x02, EN_INT_VAL);   // FIFO-ready to INT1
  afeWrite(0x08, 0);            // SYNCH
}

// Convert one FIFO word into microvolts; returns false if the tag says no valid sample
bool afeDecode(uint32_t w, float &uV) {
  uint8_t etag = (w >> 3) & 0x07;
  if (etag != 0 && etag != 2) return false;   // 0 valid, 2 valid + end of frame
  int32_t raw = (int32_t)(w >> 6);
  if (raw & 0x20000) raw -= 0x40000;          // sign-extend 18 bits
  uV = raw * 1e6f / (131072.0f * GAIN);
  return true;
}
```

### 6.4 Interrupt and acquisition

```cpp
void IRAM_ATTR onDrdy() {
  BaseType_t hp = pdFALSE;
  xSemaphoreGiveFromISR(drdySem, &hp);
  if (hp) portYIELD_FROM_ISR();
}

void setup() {
  Serial.begin(115200);
  pinMode(PIN_CS, OUTPUT); digitalWrite(PIN_CS, HIGH);
  spiAFE.begin(PIN_SCK, PIN_MISO, PIN_MOSI, PIN_CS);
  drdySem = xSemaphoreCreateBinary();
  featQ = xQueueCreate(4, sizeof(Features));
  afeBegin();
  attachInterrupt(PIN_INT1, onDrdy, FALLING);
  xTaskCreatePinnedToCore(dspTask, "dsp", 8192, NULL, 3, NULL, 0); // Core 0: signal maths
  xTaskCreatePinnedToCore(uiTask,  "ui",  8192, NULL, 2, NULL, 1); // Core 1: graphics + BLE
}
void loop() { vTaskDelete(NULL); }
```

### 6.5 On-device filters (functions)

All filters are 2nd-order IIR biquads in Direct Form II Transposed, with coefficients generated at startup using the RBJ audio-EQ cookbook formulas, so the cutoffs can be changed without recomputing tables by hand.

| Function | Purpose | Settings |
| --- | --- | --- |
| `biquadNotch(f0, Q, fs)` | Remove 50 Hz mains | f0 = 50, Q = 30 (about 1.7 Hz notch). Optional second notch at 100 Hz for the harmonic |
| `biquadHighpass(fc, Q, fs)` | Remove residual drift | fc = 1 Hz, Q = 0.707 |
| `biquadLowpass(fc, Q, fs)` | Limit EMG and aliasing leftovers | fc = 40 Hz, Q = 0.707 |
| `biquadProcess(&bq, x)` | One sample update | State z1, z2 per instance |
| `removeDC(x)` | Running mean subtraction, alpha = 0.001 | Quick start-up settling |

Cascade order per sample: DC removal, notch (50 Hz), high-pass, low-pass.

```cpp
struct Biquad { float b0,b1,b2,a1,a2,z1=0,z2=0; };

Biquad biquadNotch(float f0, float Q, float fs) {
  float w = 2*PI*f0/fs, al = sinf(w)/(2*Q), a0 = 1+al;
  Biquad b;
  b.b0 = 1/a0; b.b1 = -2*cosf(w)/a0; b.b2 = 1/a0;
  b.a1 = -2*cosf(w)/a0; b.a2 = (1-al)/a0;
  return b;
}

inline float biquadProcess(Biquad &q, float x) {
  float y = q.b0*x + q.z1;
  q.z1 = q.b1*x - q.a1*y + q.z2;
  q.z2 = q.b2*x - q.a2*y;
  return y;
}
```

### 6.6 Windowing, FFT and band power

```cpp
float ring[N_FFT];  int head = 0, sinceHop = 0;
float vReal[N_FFT], vImag[N_FFT];
ArduinoFFT<float> FFT(vReal, vImag, N_FFT, FS);

float bandPower(const float *mag, int loHz, int hiHz) {  // 1 Hz per bin at N=256, fs=256
  float p = 0;
  for (int k = loHz; k <= hiHz; k++) p += mag[k] * mag[k];
  return p;
}

void computeFeatures(Features &f) {
  for (int i = 0; i < N_FFT; i++) {           // unroll ring buffer oldest to newest
    vReal[i] = ring[(head + i) % N_FFT];
    vImag[i] = 0;
  }
  FFT.windowing(FFTWindow::Hann, FFTDirection::Forward);
  FFT.compute(FFTDirection::Forward);
  FFT.complexToMagnitude();
  f.theta = bandPower(vReal, 4, 7);
  f.alpha = bandPower(vReal, 8, 12);
  f.beta  = bandPower(vReal, 13, 30);
  f.focus = f.beta / (f.alpha + f.theta + 1e-9f);
}
```

### 6.7 Artifact rejection (blink, jaw clench, motion)

Per window, before trusting the Focus Score, the DSP task checks three conditions and sets `artifact = true` if any fails:

1. **Amplitude:** peak-to-peak above about 150 uV (blinks reach hundreds of microvolts at Fp1/Fp2).
2. **Sample-to-sample jump:** `abs(x[n]-x[n-1])` above a slew limit (motion and electrode pops).
3. **EMG ratio:** power in 30 to 40 Hz divided by total 4 to 30 Hz power above a threshold (jaw clench).

When flagged, the previous Focus Score is held and the UI shows a small warning dot, so muscle movement never corrupts the score.

### 6.8 DSP task (Core 0)

```cpp
void dspTask(void *) {
  Biquad nf = biquadNotch(50.0f, 30.0f, FS);
  Biquad hp = biquadHighpass(1.0f, 0.707f, FS);
  Biquad lp = biquadLowpass(40.0f, 0.707f, FS);
  float focusEma = 0; bool first = true;
  for (;;) {
    xSemaphoreTake(drdySem, portMAX_DELAY);
    // drain FIFO until the tag reports empty
    for (;;) {
      uint32_t w = afeRead(0x21);
      float uV;
      if (!afeDecode(w, uV)) break;
      float x = removeDC(uV);
      x = biquadProcess(nf, x);
      x = biquadProcess(hp, x);
      x = biquadProcess(lp, x);
      ring[head] = x; head = (head + 1) % N_FFT;
      if (++sinceHop >= HOP) {
        sinceHop = 0;
        Features f; computeFeatures(f);
        f.artifact = detectArtifact();
        if (!f.artifact) {                 // EMA smoothing, alpha = 0.2
          focusEma = first ? f.focus : 0.2f*f.focus + 0.8f*focusEma;
          first = false;
        }
        f.focusSmooth = focusEma;
        xQueueSend(featQ, &f, 0);
      }
    }
  }
}
```

### 6.9 UI and BLE task (Core 1)

- `drawAmbient(focusNorm)` fills the screen with a gradient: low focus is soft teal/blue, high focus shifts toward green. `focusNorm` is the Focus Score divided by a 30 second eyes-open baseline captured at start-up.
- `drawBands()` draws three small bars for theta, alpha and beta (debug view used during the POC).
- Rendering uses a sprite or DMA buffer at about 20 fps, so graphics never touch Core 0 timing.
- BLE: NimBLE server with one service and one notify characteristic carrying 3 floats (focus, alpha, beta) plus a flags byte, sent twice per second (matches the hop rate).
- Serial CSV debug stream (`t_ms, uV_filtered, theta, alpha, beta, focus, artifact`) was also enabled, which is how the Python validation data was captured.

### 6.10 Raw-data logging firmware mode

A compile-time switch `#define RAW_LOG 1` bypasses all on-device filtering and streams raw microvolt samples with timestamps at 115200 or 921600 baud over USB serial (256 samples/s needs under 5 KB/s). This raw log is the ground truth for the Python pipeline in section 7.

## 7. Python Pipeline (MNE-Python) for Offline Validation

Purpose: prove the biological signal is real (alpha on eye closure), verify the ESP32 filters and FFT match a reference implementation, and tune thresholds. Environment: Python 3.10+, `mne`, `numpy`, `scipy`, `pandas`, `pywt` (PyWavelets), `pyserial`, `matplotlib`.

### 7.1 Module layout

```
neuropulse/
  io_serial.py      capture from the ESP32 and save CSV
  load.py           CSV to mne.io.RawArray
  preprocess.py     filters, artifact marking, epoching
  features.py       band power, ratios, entropy, wavelets
  smooth.py         smoothing methods
  validate.py       experiment tests and plots
```

### 7.2 Step-by-step flow and functions

| Step | Function | Library call and settings |
| --- | --- | --- |
| 1. Capture | `record_serial(port, baud, seconds)` | pyserial; save `time_ms, uV` to CSV with event markers typed on the keyboard (rest, eyes\_closed, math) |
| 2. Build MNE object | `make_raw(csv_path)` | `mne.create_info(['Fp1-Fp2'], sfreq=256, ch_types='eeg')`, then `mne.io.RawArray(data_in_volts, info)`; convert uV to V by x 1e-6 |
| 3. Annotations | `add_events(raw, markers)` | `mne.Annotations(onset, duration, description)` then `raw.set_annotations()` |
| 4. Inspect raw | `plot_raw_and_psd(raw)` | `raw.plot()`, `raw.compute_psd(fmax=60).plot()`; confirm the 50 Hz peak before filtering |
| 5. Notch filter | `apply_notch(raw)` | `raw.notch_filter(freqs=[50, 100], method='fir')` or `method='spectrum_fit'` when the mains peak drifts |
| 6. Band-pass | `apply_bandpass(raw)` | `raw.filter(l_freq=1.0, h_freq=40.0, method='iir', iir_params=dict(order=4, ftype='butter'))` for a causal-like match with the ESP32; FIR (`method='fir'`, zero-phase) used as a reference |
| 7. Artifact marking | `mark_artifacts(raw)` | Single channel, so no ICA or EOG regression. Custom thresholds: peak-to-peak above 150 uV, gradient limit, and EMG band ratio. Results written as `BAD_` annotations via `mne.Annotations` |
| 8. Epoching | `make_epochs(raw)` | `mne.make_fixed_length_epochs(raw, duration=2.0, overlap=1.0, reject_by_annotation=True, preload=True)` |
| 9. Baseline correction | `epochs.apply_baseline` not used; instead each epoch is demeaned | Avoids introducing steps the ESP32 does not do |
| 10. PSD | `compute_psd_epochs(epochs)` | `epochs.compute_psd(method='welch', fmin=1, fmax=40, n_fft=256, n_overlap=128, window='hann')` |
| 11. Band power | `band_powers(psd, freqs)` | Integrate with `scipy.integrate.trapezoid` over theta 4-7, alpha 8-12, beta 13-30; also relative power (band / total 4-30) |
| 12. Focus score | `focus_score(theta, alpha, beta)` | `beta / (alpha + theta)`, same formula as firmware |
| 13. Smoothing | see 7.3 | Compare against the EMA used on-device |
| 14. Compare | `compare_esp_vs_python(csv_esp, raw)` | Pearson correlation and Bland-Altman of Focus Score per window |

### 7.2.1 Artifact Handling : Motion, Facial Muscle, Eye Blinks and ECG

With one channel there is no ICA, SSP or EOG regression, so the POC pipeline detects each artifact by its own signature and then rejects the segment (or, for ECG, subtracts a template) instead of unmixing sources. This expands step 7 of the table above.

| Artifact | Signature at Fp1/Fp2 | Detection | Handling | Python function | On-device equivalent |
| --- | --- | --- | --- | --- | --- |
| Eye blink | Single bump of 100 to 300 uV lasting 200 to 400 ms; energy mostly 0.5 to 8 Hz, so it overlaps theta | Band-pass copy 1 to 10 Hz, then `find_peaks` on the absolute value (threshold about 75 uV, 0.3 s minimum spacing) | Annotate 0.2 s either side as `BAD_blink` and drop those epochs | `detect_blinks()` | Peak-to-peak above 150 uV in the window |
| Facial or jaw muscle (EMG) | Broadband, spiky bursts above 20 Hz lasting 0.2 s to several seconds | Band-pass copy 20 to 40 Hz, rolling RMS (0.5 s), robust z-score against the median and MAD; also 30 to 40 Hz power divided by 4 to 30 Hz power | Annotate as `BAD_emg` and drop epochs; never filter it away, since it overlaps beta | `detect_emg()` | EMG band ratio threshold |
| Motion, cable pull, electrode pop | Sudden step or spike, slow baseline swing, or a flat/saturated line when the electrode lifts | Derivative z-score, slew limit (about 40 uV per sample), `annotate_amplitude` for peak and flat segments, and 50 Hz share of total power as a contact-quality check | Annotate as `BAD_motion` or `BAD_flat`; short slow drifts are removed per epoch with `scipy.signal.detrend` | `detect_motion()` | Sample-to-sample slew limit |
| ECG (heartbeat) | Periodic QRS spikes at about 1 to 1.7 Hz, a few microvolts; largest when the reference sits on the mastoid and the electrode spacing is wide; weak with a symmetric frontal pair | Band-pass copy 5 to 35 Hz, R-peak detection with `mne.preprocessing.find_ecg_events` using the recording channel as the ECG source | Average a QRS template over at least 50 beats, scale it per beat by least squares and subtract; skip if the template is under 5 uV peak-to-peak (below the noise floor) | `detect_ecg()`, `remove_ecg_template()` | None in the POC firmware; checked offline only |

**Order of operations**

1. Run `notch_filter` and the 1 to 40 Hz band-pass on the continuous recording first, so filter ringing is not spread around later cuts.
2. Make light-filtered copies (1 to 10 Hz for blinks, 20 to 40 Hz for EMG, 5 to 35 Hz for ECG) used only for detection; the main data is never altered by them.
3. Remove ECG by template subtraction before any rejection, so heartbeat spikes do not trigger false motion flags.
4. Detect blinks, EMG, motion and flat/saturated segments, and write each as a `BAD_` annotation, merged with the existing event annotations.
5. Epoch with `reject_by_annotation=True`, plus `reject=dict(eeg=150e-6)` and `flat=dict(eeg=0.5e-6)` as a second safety net.
6. Report the percentage of epochs rejected per session; above about 30% the session is treated as unusable and the electrodes are re-seated.

**Starting thresholds (tuned per user during the POC, not universal)**

| Parameter | Value |
| --- | --- |
| Peak amplitude reject | 150 uV peak-to-peak |
| Flat-line reject | under 0.5 uV over 1 s |
| Blink peak height | 75 uV (1 to 10 Hz copy) |
| Blink padding | 0.2 s each side |
| EMG robust z-score | above 4.0 |
| Slew limit | 40 uV per sample |
| ECG minimum beats for a template | 50 |

**Skeleton of the detectors**

```python
import numpy as np, mne
from scipy.signal import find_peaks, butter, sosfiltfilt

def _bp(x, lo, hi, fs=256, order=4):
    sos = butter(order, [lo, hi], btype='band', fs=fs, output='sos')
    return sosfiltfilt(sos, x)

def _mask_to_annotations(mask, fs, label, pad=0.0):
    edges = np.diff(np.r_[0, mask.astype(int), 0])
    on, off = np.where(edges == 1)[0], np.where(edges == -1)[0]
    return mne.Annotations(on / fs - pad, (off - on) / fs + 2 * pad, label)

def detect_blinks(raw, thresh_uv=75, pad=0.2):
    fs = raw.info['sfreq']
    y = _bp(raw.get_data()[0] * 1e6, 1, 10, fs)
    pk, _ = find_peaks(np.abs(y), height=thresh_uv, distance=int(0.3 * fs))
    return mne.Annotations(pk / fs - pad, 2 * pad, 'BAD_blink')

def detect_emg(raw, win=0.5, z=4.0):
    fs = raw.info['sfreq']
    hf = _bp(raw.get_data()[0] * 1e6, 20, 40, fs)
    n = int(win * fs)
    rms = np.sqrt(np.convolve(hf ** 2, np.ones(n) / n, mode='same'))
    med = np.median(rms)
    mad = np.median(np.abs(rms - med)) * 1.4826 + 1e-12
    return _mask_to_annotations((rms - med) / mad > z, fs, 'BAD_emg')

def detect_motion(raw, slew_uv=40, pad=0.1):
    fs = raw.info['sfreq']
    x = raw.get_data()[0] * 1e6
    jump = np.r_[0, np.abs(np.diff(x))] > slew_uv
    return _mask_to_annotations(jump, fs, 'BAD_motion', pad)

def remove_ecg_template(raw, half=0.3):
    fs = int(raw.info['sfreq']); h = int(half * fs)
    ev, _, _ = mne.preprocessing.find_ecg_events(
        raw, ch_name=raw.ch_names[0], l_freq=5, h_freq=35, verbose=False)
    x = raw.get_data()[0].copy()
    idx = [s for s in ev[:, 0] if h <= s < len(x) - h]
    if len(idx) < 50:
        return raw
    tmpl = np.mean([x[s - h:s + h] for s in idx], axis=0)
    if np.ptp(tmpl) < 5e-6:
        return raw                      # below the noise floor, nothing to remove
    for s in idx:
        seg = x[s - h:s + h]
        x[s - h:s + h] = seg - (seg @ tmpl) / (tmpl @ tmpl) * tmpl
    return mne.io.RawArray(x[None, :], raw.info)

def mark_artifacts(raw):
    raw = remove_ecg_template(raw)
    amp, _ = mne.preprocessing.annotate_amplitude(
        raw, peak=dict(eeg=150e-6), flat=dict(eeg=0.5e-6), min_duration=0.01)
    ann = raw.annotations
    for a in (detect_blinks(raw), detect_emg(raw), detect_motion(raw), amp):
        ann = ann + a
    raw.set_annotations(ann)
    return raw
```

**Caveats found in the POC**

- MNE's `annotate_muscle_zscore` is not used: it expects several channels and a 110 to 140 Hz band, which a 256 sps single-channel recording cannot provide. The on-chip 40 Hz low-pass also hides most EMG energy, so to validate EMG detection record a session with the MAX30003 low-pass set to bypass (Nyquist is still 128 Hz).
- Blink energy overlaps theta, so filtering or wavelet thresholding would distort the Focus Score; rejecting the window is the safer choice. A bipolar Fp1-Fp2 pair partly cancels symmetric blinks, so amplitudes can be lower than the thresholds above; check on your own recordings.
- `find_eog_events` offers an alternative blink detector (1 to 10 Hz filter, threshold in volts) if you prefer MNE's own routine over the custom `detect_blinks()`.
- Template subtraction assumes beats are at least 0.3 s apart; very fast heart rates (children can exceed 120 bpm) need a shorter half-window of about 0.2 s.

### 7.3 Smoothing methods tried

| Function | Method | Parameters | Result |
| --- | --- | --- | --- |
| `smooth_ema(x, alpha)` | Exponential moving average | alpha = 0.2 | Chosen for firmware: one multiply-add, no buffer |
| `smooth_moving_avg(x, w)` | Boxcar moving average | w = 5 windows | Works, but lags and needs a buffer |
| `smooth_median(x, k)` | `scipy.signal.medfilt` | k = 5 | Good at rejecting single-window spikes |
| `smooth_savgol(x, w, p)` | `scipy.signal.savgol_filter` | window 7, poly order 2 | Preserves peaks; offline only |
| `smooth_welch_avg(epochs)` | Average PSDs over several epochs before the ratio | 5 epochs | More stable ratio than smoothing after the ratio |

### 7.4 Feature extraction functions

| Function | Feature | How |
| --- | --- | --- |
| `band_powers()` | Absolute and relative theta, alpha, beta | Welch PSD plus trapezoid integration |
| `engagement_index()` | Beta / (Alpha + Theta) | Primary feature, matches firmware |
| `theta_beta_ratio()` | Theta / Beta | Alternative attention marker, compared offline only |
| `peak_alpha_frequency()` | Frequency of max PSD in 8-12 Hz | `freqs[np.argmax(psd[alpha_mask])]` |
| `spectral_edge_freq(psd, 0.95)` | 95% spectral edge | Cumulative sum of PSD |
| `spectral_entropy()` | Shannon entropy of normalised PSD | `-sum(p*log2(p))` |
| `hjorth_params(x)` | Activity, mobility, complexity | Variance of x, dx and ddx; cheap time-domain features |
| `wavelet_dwt_bands(x)` | DWT band energies | `pywt.wavedec(x, 'db4', level=5)`; at 256 Hz the detail levels map roughly to 64-128, 32-64, 16-32 (beta), 8-16 (alpha-ish) and 4-8 Hz (theta) |
| `wavelet_morlet_tfr(epochs)` | Time-frequency view | `mne.time_frequency.tfr_morlet(epochs, freqs=np.arange(4, 31), n_cycles=freqs/2, return_itc=False)` or `tfr_array_morlet` for NumPy input |

The DWT and Morlet features were explored to see whether they separate focus from rest better than FFT band power; they were not ported to the ESP32, which keeps the FFT approach for cost and simplicity.

### 7.5 Example skeleton

```python
import numpy as np, pandas as pd, mne
from scipy.integrate import trapezoid

BANDS = {'theta': (4, 7), 'alpha': (8, 12), 'beta': (13, 30)}

def make_raw(csv_path, sfreq=256):
    df = pd.read_csv(csv_path)
    data = df['uV'].to_numpy()[None, :] * 1e-6
    info = mne.create_info(['Fp1-Fp2'], sfreq, 'eeg')
    return mne.io.RawArray(data, info)

def preprocess(raw):
    raw.notch_filter([50, 100], method='fir')
    raw.filter(1.0, 40.0, method='iir',
               iir_params=dict(order=4, ftype='butter'))
    return raw

def band_powers(epochs):
    psd = epochs.compute_psd(method='welch', fmin=1, fmax=40,
                             n_fft=256, n_overlap=128)
    p, f = psd.get_data(return_freqs=True)
    out = {}
    for name, (lo, hi) in BANDS.items():
        m = (f >= lo) & (f <= hi)
        out[name] = trapezoid(p[:, 0, m], f[m], axis=-1)
    out['focus'] = out['beta'] / (out['alpha'] + out['theta'])
    return out
```

## 8. Validation Experiments Run in the POC

| Test | Procedure | Pass criterion |
| --- | --- | --- |
| Noise floor | Short the inputs (or input via a 1 kOhm resistor pair), record 60 s | Flat spectrum, no mains spike beyond the noise floor |
| Mains rejection | Record with electrodes attached; PSD before and after the notch | 50 Hz peak reduced by 20 dB or more on-device |
| Eyes open vs eyes closed (Berger effect) | 30 s each, alternating, 3 repeats | Clear alpha rise (8-12 Hz) on eye closure |
| Blink and jaw check | Deliberate blinks and jaw clenches | Artifact flag fires; Focus Score held |
| Rest vs mental arithmetic | 60 s relaxed, then 60 s serial subtraction | Beta rises relative to alpha plus theta |
| ESP32 vs Python parity | Same raw log through both pipelines | High correlation of per-window Focus Score; any gap explained by causal vs zero-phase filtering |
| Real-time load | Monitor FFT plus screen redraw timing | No dropped FIFO samples, display about 20 fps |

## 9. Known Limitations and Lessons From the POC

- **Front-end suitability:** The MAX30003 and AD8232 are ECG-oriented chips. They worked for this proof of concept, but EEG amplitude is far smaller than ECG, so noise and drift are the main risks. A purpose-built EEG front end (for example ADS1299 class) is the usual upgrade path.
- **Single frontal channel:** A bipolar Fp1-Fp2 pair cancels activity common to both sites, so alpha/theta/beta power is lower than with a single electrode referenced to the mastoid. Test both montages before fixing a design.
- **Focus Score is a proxy:** Beta/(Alpha+Theta) is a simple engagement index, not a clinical measure of attention. It varies strongly between people, so per-user baseline normalisation is required.
- **Frontal electrodes pick up ocular and facial muscle activity,** so artifact rejection is not optional.
- **Cheap gel electrodes dry out;** signal quality degrades after 30 to 60 minutes.
- **Wi-Fi/BLE noise:** keep the radio idle during recordings, or batch BLE sends, when characterising the analog chain.
- **Powerbank:** some bank models inject switching noise; if the spectrum shows spurs, add a ferrite bead and 100 uF plus 100 nF decoupling at the AFE supply.

## 10. Software Dependency Summary

| Environment | Packages |
| --- | --- |
| Arduino IDE / arduino-esp32 core | ESP32 Arduino core 2.x or 3.x, arduinoFFT v2, TFT\_eSPI (or LovyanGFX), NimBLE-Arduino |
| Python | mne, numpy, scipy, pandas, pywt, pyserial, matplotlib |
