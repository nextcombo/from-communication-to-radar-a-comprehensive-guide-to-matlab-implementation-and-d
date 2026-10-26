# From Communication to Radar: A Comprehensive Guide to MATLAB Implementation and Dual-Scenario Application of OCDM Technology [with MATLAB Code]

1. OCDM Technology: What is it? Why is it important? Before we proceed with the formal analysis of code, let's first clarify the core positioning of OCDM technology:
OCDM (Orthogonal Chirp Division Multiplexing) is a multicarrier transmission technique based on chirp signals. Compared to traditional OFDM (Orthogonal Frequency Division Multiplexing), it achieves signal multiplexing through "orthogonality of chirp pulses," offering significant advantages in terms of anti-multipath fading, reducing peak-to-average power ratio (PAPR), and enhancing spectral efficiency. It can also naturally integrate "communication - radar" dual functions, making it very suitable for scenarios such as 5G/6G, vehicle-to-everything, Internet of Things, etc.
The core of this article's code is the DFTnT/IDFnT transformation (Discrete Fourier Transform - Nyquist Transform and its inverse transformation) - this is the "soul" of OCDM modulation and demodulation. The code includes custom subfunctions FastDFnT and FastInvDFnT, which we will analyze later.
2. Scenario One: OCDM High-Speed Data Communication System
The first code module fully implements the entire process from "bit stream → QAM modulation → OCDM modulation → channel transmission → OCDM demodulation → QAM demodulation → performance evaluation," allowing for an intuitive view of OCDM's ability to resist interference and bit error performance in communication scenarios.
1. Core Process Decomposition (with corresponding code modules) (1) Parameter initialization: Define the "basic rules" of the system first, setting key parameters that directly determine the system performance limit:
ModOrder = 16;  % 16QAM modulation (each symbol carries 4 bits of information)
N = 1024;  % OCDM sub-carrier number (determines the granularity of time-frequency resources)
GI = 128;  % Loop prefix length (counteracts multipath delay spread)
NumBlocks = 64;  % OCDM symbol block count (controls data volume)
EbN0dB = 20;  % Bit signal-to-noise ratio (core metric for evaluating anti-noise capability)
Channel = 'awgn';  % Channel type ('awgn' Gaussian White Noise/'multipath' multipath)

• 16QAM modulation: Represents data points in the constellation diagram with 16 discrete points, each corresponding to 4 bits (e.g., "0000" corresponds to a point), is a common choice for balancing "spectral efficiency" and "anti-interference capability."

• Loop prefix (GI): Copies 128 samples from the tail of an OCDM symbol to the head to avoid intersymbol interference (ISI) between adjacent symbols, a classic method in wireless communications to counter multipath effects.
NumBits = NumBlocks * N * log2(ModOrder); % Total number of bits (64*1024*4=262144bits)
QAMMod function: maps a binary bitstream into 16-QAM constellation points (complex values) ensuring "unit average power" to avoid signal power fluctuations affecting performance.
1  
OCDM modulation (IDFnT transformation):
txOCDM = FastInvDFnT(txSymMat, N); % Core: Inverse DFnT transformation, generates the time-domain signal of OCDM.
Here, FastInvDFnT is a custom subroutine that implements IDFnT through the fast algorithm of "chirp multiplication → FFT → chirp multiplication". It converts the frequency-domain QAM symbols into time-domain orthogonal chirp signals, ensuring that multiple chirp signals do not interfere with each other during transmission.
1  
Adding prefixes and serialization:
txWithCP = \[txOCDM(end-GI+1:end, :); txOCDM]; % Headers added with cyclic prefix s = txWithCP(:); % Serialization of matrix (converted to 1D vector for simulation of actual channel transmission)
•  AWGN Channel (No Multipath, e.g., Open Field):  
snr\_dB = EbN0dB + 10*log10(log2(ModOrder)); % Convert to Signal-to-Noise Ratio (QAM each symbol contains log2(ModOrder) bits) sr = awgn(s, snr\_dB, 'measured'); % Add Gaussian White Noise

• Multipath Channel (e.g., Urban High-rise environment, signals reflected multiple times before reaching the receiver):  
PathDelays = [0;4;9]; % Multipath delays (0/4/9 samples) PathGains = [1; 0.4*exp(1j*0.7); 0.2]; % Multipath amplitude and phase (first path has the strongest gain) h = zeros(L,1); for i = 1:numel(PathDelays) &#x20; d = PathDelays(i)+1; &#x20; if d <= L, h(d) = h(d) + PathGains(i); endendsr = conv(s,h); sr = sr(1:L); % Convolution to simulate multipath fading sr = awgn(sr, snr\_dB, 'measured'); % Add noise

• Multipath Channel is implemented through "Convolution": The transmitted signal is convolved with the channel impulse response (h), simulating the effect of signals being transmitted through different paths.

 (4)Receiver: Recover "Original Bits" from "Damaged Signal": The receiver is the inverse process of the sender, focusing on mitigating channel interference and restoring original information:
1  
De-cyclic prefix and OCDM demodulation:
1. QAM Demodulation and Bit Recovery:
rxSym = rxSymMat(:); % Serialization of rxBits
qamdemod = @(rxSym, ModOrder, 'OutputType', 'bit', 'UnitAveragePower', true) qamdemod(rxSym, ModOrder, 'OutputType','bit', 'UnitAveragePower', true); % QAM demodulation

2. Performance Assessment: The system's performance is evaluated using two core metrics: Bit Error Rate (BER) and Root Mean Square Error (EVM).
BER = mean(rxBits ~= txBits); % Bit Error Rate (Number of erroneous bits divided by total bits)
EVMrms = sqrt(mean(abs(rxSym - txSym).^2) / mean(abs(txSym).^2)); % Root Mean Square Error

3. BER: It is the "lifeline" of a communication system. For example, if BER = 1e-5, it means that there are only one error per ten thousand bits transmitted, which is exceptionally low at an EbN0 of 20dB.
EVM: Measures the deviation of received symbols from the original symbols (in percentage), with an EVM < 5% generally considered to indicate good system performance, and the results in this paper typically fall within 1%-3%.

Simultaneously, code plotting constellations intuitively displays signal quality:
figure('Name','OCDM Constellation','Color','w');plot(rxSym, '.', 'MarkerSize', 4); % Received symbol (scatter plot)
ref = qammod((0:ModOrder-1).', ModOrder, 'InputType','integer','UnitAveragePower',true);
plot(ref, 'o', 'MarkerSize', 6, 'LineWidth', 1.2); % Original reference constellation points
• In the constellation diagram, the closer the received scatter plot is to the reference points, the less noise interference, indicating better system performance.

Scene Two: OCDM Multi-Object Radar System (Distance - Velocity Joint Estimation)
The second code module showcases the "communication - radar" integration capability of OCDM — utilizing the time-frequency characteristics of OCDM signals to achieve joint estimation of distance and velocity for multiple targets, which holds significant practical value in scenarios such as autonomous vehicles (collision avoidance radar), security monitoring, etc.

1. Core principle: How does a radar "see" a target? The core logic of an OCDM radar is:
1. Sending OCDM signals;
2. Upon encountering a target, the reflected (echo) signal is received;
3. The echo signal carries information about the target: the distance corresponds to the signal delay (propagation time × speed of light / 2), and the velocity corresponds to the Doppler shift (signal frequency change due to target movement);
4. Through the "distance - Doppler two-dimensional FFT", extract the delay and frequency shift from the echo signal, thereby calculating the target's distance and velocity.
2. Break down the core code process (1) Parameter initialization: The "physical foundation" of radar systems first defines the physical parameters and OCDM system parameters of the radar.
c0  = physconst('LightSpeed'); % Speed of light (3e8 m/s, key for calculating distance) fc = 24e9; % Carrier Frequency (24GHz, commonly used in millimeter wave radar, balanced between resolution and penetration) lambda = c0 / fc; % Wavelength (1.25cm, used for calculating speed) N = 128; % OCDM sub-chirp number M = 64; % OCDM symbol block count (radar observation times) B = 20e6; % Signal bandwidth (20MHz, determining distance resolution) T = N / B; % Effective symbol duration Tcp = T/4; % Loop prefix duration Ts = T + Tcp; % Total symbol duration (radar pulse repetition period)
Key Performance Indicators (Code automatically calculated):
range\_res = c0 / (2 * B);  % Distance resolution (7.5m, higher bandwidth leads to higher resolution)
(2) Multi-target Scene Setting: Simulate real-world environment code settings for three targets, covering different distances, speeds, and reflection intensities:
tgt\_R = [140, 200, 320]; % Target distance (140m/200m/320m)
tgt\_V = [15, -30, 0]; % Target velocity (15m/s approaching/30m/s receding/static)
tgt\_alpha = [1.0, 0.7, 0.5]; % Target reflection coefficient (the larger the value, the stronger the echo)
• The velocity is negative indicating the target "recedes" from the radar (Doppler shift is negative), positive indicates it's "approaching" (Doppler shift is positive).
(3) Backscatter Construction: The backscatter signal is the "data source" of the OCDM radar. The code simulates real echoes by "delay shifting + Doppler modulation":
RxOCDM = zeros(N, M);
Circular Shift: A fast method to achieve signal delay. For example, a target of 140m corresponds to a delay tau = 2 * 140 / 3e8 ≈ 933ns, which is converted into the number of sampling points n_delay ≈ 18.
•  
Doppler modulation: Simulates the phase shift caused by target motion through the complex exponential exp(1j*2*pi*fd*(m-1)*Ts), where fd is the Doppler shift (fd = 2*V/lambda).
(4) Distance-Velocity Joint Estimation: The core algorithm code realizes target parameter estimation through "channel elimination→two-dimensional FFT".
1  
Channel-influence elimination (Hadamard product):
Ftx = fft(TxOCDM, N, 1); % Sending signals in the frequency domain: Frx = fft(RxOCDM, N, 1); % Received signal's frequency domain H_est = Frx / (Ftx + 1e-12); % Estimated channel response (to eliminate the fixed effects of transmission/reception channels)
Adding 1e-12 is to avoid a denominator of 0, ensuring the calculation is stable.
1  
2D FFT and Peak Detection:
% Distance to IFFT (Extracting delay information) R\_mat = ifft(H\_est, NPer, 1); % NPer=4\*N (zero-padding to improve resolution) % Speed towards FFT (extract Doppler frequency shift information) RD\_mat = fftshift(fft(R\_mat, MPer, 2),2); % fftshift shifts negative frequencies to the left \[iR,iV] = ind2sub(size(RD\_mat), idx); est\_range = range\_axis(iR); % Estimated distance est\_vel = vel\_axis(iV); % Estimated speed
Zero-padding (NPer=4*N): No actual data is increased, only the frequency resolution of FFT is improved to make the target location more accurate.
•  
fftshift: shifts the negative frequency of Doppler shift (away from the target) to the left side of the coordinate axis, and the positive frequency (close to the target) to the right side, which is in line with intuitive cognition.
(5) Visualization: The radar detection results are intuitively displayed through code generation. Two types of graphs are used to present the results:
1  
3D Distance - Doppler Map:
Figure: 'Name', 'OCDM Range-Doppler 3D', 'Color', 'w'; surf(range_axis, vel_axis, RD_mag.', 'EdgeColor', 'none'); xlabel('Range (m)'); ylabel('Velocity (m/s)'); zlabel('Magnitude (dB)');
• 3D surface's "peak" corresponds to the target position, with higher peaks indicating stronger target echoes.
1
2D heatmap (with target markers):
figure('Name', 'OCDM Range-Doppler Heatmap', 'Color', 'w'); imagesc(range_axis, vel_axis, RD_mag.'); axis xy;hold on;for p=1:P, [~, ir] = min(abs(range_axis - tgt_R(p))); [~, iv] = min(abs(vel_axis - tgt_V(p))); plot(range_axis(ir), vel_axis(iv), 'rx', 'LineWidth', 2, 'MarkerSize', 10); % Red cross marks true targets end
• In the heatmap, areas with brighter colors correspond to targets, with red crosses marking true targets for visual verification of estimation accuracy (estimated errors in this paper are generally less than 1m/0.5m/s).
Simulation Results
Completed

## Images

![From Communication to Radar: A Comprehensive Guide to MATLAB Implementation and Dual-Scenario Application of OCDM Technology [with MATLAB Code]](01.jpg)
![From Communication to Radar: A Comprehensive Guide to MATLAB Implementation and Dual-Scenario Application of OCDM Technology [with MATLAB Code]](02.jpg)
![From Communication to Radar: A Comprehensive Guide to MATLAB Implementation and Dual-Scenario Application of OCDM Technology [with MATLAB Code]](03.jpg)
![From Communication to Radar: A Comprehensive Guide to MATLAB Implementation and Dual-Scenario Application of OCDM Technology [with MATLAB Code]](04.jpg)
![From Communication to Radar: A Comprehensive Guide to MATLAB Implementation and Dual-Scenario Application of OCDM Technology [with MATLAB Code]](05.jpg)
![From Communication to Radar: A Comprehensive Guide to MATLAB Implementation and Dual-Scenario Application of OCDM Technology [with MATLAB Code]](06.jpg)

<!-- PAYMENT_BLOCK -->

## Get this dataset

- Payment link (Stripe): [Buy now](https://buy.stripe.com/3cs8yP7sY87d0vu9AB)
- Payment QR code: ![Payment QR code](pay.png)

<!-- /PAYMENT_BLOCK -->
