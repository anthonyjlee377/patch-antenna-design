# 10 GHz Patch Antenna

## 1. Overview

First antenna design intended to operate at 10GHz, built as a course project for Antenna Engineering (Yonsei University, EEE4140). 

- Learned the general flow of antenna design, fabrication, and measurement.
- Learned the concept of time-domain gating while trying to correct VNA data.
- This project also inspired the addition of a time-domain gating method to SPITE, my own open-source RF simulation library.
- **Lessons for future designs:** account for SMA connector and soldering effects at the design stage, and perform VNA calibration more carefully to reduce measurement artifacts.


## 2. Simulation Results (10 GHz)

**Materials:** Copper, FR4

| Metric | Value |
|---|---|
| S11 | -25.32 dB at 10 GHz (-26.68 dB at 9.972 GHz) |
| Normalized input impedance | 0.86 + 0.057j |
| Directivity | 8.067 dBi |
| Radiation efficiency | -2.04 dB |
| Total efficiency | -2.05 dB |

A MATLAB skeleton script for the EAGLE script exportwas provided by Prof. Insang Yoo as part of the course.
Fabrication files were generated with Autodesk EAGLE.

## 3. VNA Measurement Results

The raw S11 measurement showed severe ripples between 8 and 12 GHz, with a minimum value of -23.4 dB at 11.46 GHz. These ripples were attributed to calibration and connector mismatches.

Band-limited time-domain gating was applied to remove the delayed reflections:

```matlab
%==========================================================================
% Original data
%==========================================================================
clear; clc;
close all;
set(0, "DefaultFigureWindowStyle", "docked")

% s-parameter data
s_param = sparameters('JeongseokLee.s1p');
figure
rfplot(s_param);  % s parameter values

% Turn given data into a vector
frequencies = s_param.Frequencies / 1e9;  % converted to GHz
mask = (frequencies >= 8) & (frequencies <= 12);  % frequency range 
frequencies = frequencies(mask);
S11 = squeeze(s_param.Parameters(1,1,:));  % S11 parameters extracted and turned into a vector 
S11 = S11(mask);

% Find minimum S11 and its frequency
S11_dB = 20*log10(abs(S11));
[S11_min_dB, idx] = min(S11_dB(:));
min_freq = frequencies(idx);
fprintf('Minimum value of %.3f dB at %.3f GHz\n\n', S11_min_dB, min_freq);
hold on;
plot(min_freq, S11_min_dB, 'ro', 'MarkerSize', 8, 'MarkerFaceColor', 'r');
text(min_freq, S11_min_dB, sprintf('  %.2f dB @ %.2f GHz', S11_min_dB, min_freq), ...
    'VerticalAlignment', 'bottom','HorizontalAlignment', 'right', 'FontSize', 9);

%==========================================================================
% Post processing to estimate S11 without ripples
%==========================================================================
% Window to reduce edge-truncation artifacts, needed to work around limited data (8-12 GHz)
N = length(S11);
a = 0.05;
W = tukeywin(N, a);  % window in the frequency domain
S11_windowed = S11 .* W;

% Time domain
s11 = ifft(S11_windowed);  % approximated impulse response in the time domain
df = (frequencies(2)-frequencies(1))*1e9;  %  frequency step in Hz
fs = df * N;  % sampling rate in the dual domain
t = (0:N-1)/fs;  % seconds

figure;
plot(t*1e9, abs(s11));  % ns
xlabel('Time (ns)'); ylabel('|Response|');
title('Analytic impulse response (8–12 GHz band-limited)');

% Time gate to filter out delayed responses (reflections)
N = length(s11);
gate = zeros(N,1);
gate_cutoff = 20*1e-9;  % remove reflections after 20ns
idx_cutoff = round(gate_cutoff * fs);
gate(1:idx_cutoff) = 1;

s11_gated = s11 .* gate;  % delay removed
S11_gated = fft(s11_gated) ./ W;  % divide out the window applied earlier
S11_gated_dB = 20*log10(abs(S11_gated));  % dB

figure;
plot(frequencies, S11_gated_dB);
xlim([8+4*a,12-4*a]);
hold on;
yline(-10, '--r', 'LineWidth', 1.2, 'Label', '-10 dB');
xlabel('Frequency (GHz)'); ylabel('|S_{11}| (dB)');
title('S_{11} after time domain gating (8–12 GHz)');

[S11_min_dB, idx] = min(S11_gated_dB(:));
min_freq = frequencies(idx);
fprintf('Minimum value of %.3f dB at %.3f GHz after time domain gating\n', S11_min_dB, min_freq);
hold on;
plot(min_freq, S11_min_dB, 'ro', 'MarkerSize', 8, 'MarkerFaceColor', 'r');
text(min_freq, S11_min_dB, sprintf('  %.2f dB @ %.2f GHz', S11_min_dB, min_freq), ...
    'VerticalAlignment', 'top','HorizontalAlignment', 'center', 'FontSize', 9);
```

## 4. Results After Time-Domain Gating

Ripples were substantially reduced. The minimum S11 after gating was -20.65 dB at 11.46 GHz. However, the S11 value at 10GHz worsened to above -8 dB after gating, indicating a shift in the antenna's effective operating frequency.





