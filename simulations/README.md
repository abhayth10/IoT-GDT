# IoT-GDT Simulations

## Overview

This folder contains MATLAB/Simulink simulations for testing and validating the IoT soil monitoring system under realistic environmental conditions. The simulations model the Solan/Nauni field site in Himachal Pradesh, India, and provide virtual sensor data streams for system validation before field deployment.

## Purpose

- **System Validation**: Test sensor algorithms and control logic without hardware
- **Climate Modeling**: Simulate realistic environmental conditions for the Himalayan region
- **Algorithm Development**: Develop and tune data processing algorithms
- **Performance Prediction**: Estimate system behavior under various environmental scenarios
- **Field Testing Baseline**: Establish baseline expectations before actual field deployment

### Prerequisites
- MATLAB R2020a or later (https://in.mathworks.com/downloads/web_downloads/13472755)
- Simulink installed(installed from checkbox in matlab setup)
- Basic MATLAB knowledge
(Dia)

## Folder Structure
simulations/
├── code/                          # MATLAB initialization and helper functions
│   ├── init_nauni.m              # Site parameters and simulation setup
│   ├── diurnal_env.m             # Generates realistic diurnal temperature/humidity patterns
│   └── sensor_on.m               # Manages sensor duty cycle (60s awake / 30min sleep)
├── model/                         # Simulink models
│   └── nauni_fieldtesting.slx    # Main Simulink simulation model
└── README.md    

**Typical Workflow**:
```matlab(run in matlab terminal)
% 1. Initialize simulation parameters
init_nauni;

% 2. Open Simulink model
open('nauni_fieldtesting.slx');

% 3. Run simulation
sim('nauni_fieldtesting.slx', struct('StopTime', num2str(StopTime)));

% 4. Analyze results in X-Y scope



```

## Site Configuration (Solan/Nauni) - meta data 

### Geographic Information
- **Latitude**: 30.85° N
- **Longitude**: 77.18° E
- **Altitude**: 1350 m above sea level
- **Region**: Himachal Pradesh, India

### Typical November-December Climate (Baseline)

| Parameter | Value | Notes |
|-----------|-------|-------|
| **Temperature Mean** | 15°C | Moderate autumn conditions |
| **Temperature Swing** | ±11°C | Diurnal variation (5°C to 23°C) |
| **Humidity Mean** | 50% RH | Post-monsoon transition |
| **Humidity Swing** | ±15% RH | 45% to 85% RH daily |
| **Initial Soil Moisture** | 60% VWC | Moderate soil water content |
| **Soil Moisture Range** | 15-70% VWC | Min (dry) to max (saturated) |

### Environmental Processes

#### Temperature Cycle
- **Minimum**: ~00:00 hr
- **Maximum**: ~11:00 
- **Model**: Sinusoidal diurnal pattern
- **Equation**: $T(t) = T_{mean} + T_{amp} \sin\left(\frac{2\pi(h-5)}{24}\right)$
  - Where $h$ = hour of day (0-24)

#### Relative Humidity Cycle
- **Maximum**: Pre-dawn (~00:00-06:00)
- **Minimum**: Mid-day (~10:00-15:00)
- **Anti-phase**: RH rises as temperature falls
- **Model**: Sinusoidal pattern offset from temperature
- **Equation**: $RH(t) = RH_{mean} + RH_{amp} \sin\left(\frac{2\pi(h+7)}{24}\right)$

#### Soil Moisture Dynamics
- **Evaporation**: 0.2% VWC/hour under normal conditions
- **Irrigation Events**: 6% VWC added every 36 hours
- **Constraints**: Moisture bounded between 15% (dry) and 70% (saturated)
- **Rate**: Evaporation rate = 0.2 ÷ 3600 %VWC/hour

## Sensor Models

### Sensor Dynamics

All sensors are modeled with **first-order lag** (low-pass filter) to simulate realistic sensor response characteristics:

- **Time Constant (τ)**: 2 seconds
- **Sample Time (Ts)**: 1 second
- **Transfer Function**: $\frac{1}{1 + \tau s}$
- **Discrete Implementation**: $a = e^{-T_s/\tau}$

This creates a realistic 2-second response delay in sensor readings.

### Sensor Noise

Sensor measurements include realistic noise (1-sigma values):
 These ranges were noticed during field trial where each parameter varied by following values.
| Sensor | Noise Level | Unit |
|--------|-------------|------|
| **Temperature** | ±0.1 | °C |
| **Humidity** | ±0.5 | %RH |
| **Soil Moisture** | ±0.7 | %VWC |

## MATLAB Scripts

### `init_nauni.m` - Simulation Initialization

**Purpose**: Sets all site parameters, simulation settings, and environmental conditions

**Key Variables**:
```matlab
% Site location
site.lat = 30.85; site.lon = 77.18; site.alt_m = 1350;

% Simulation timing
Ts = 1;                    % Sample time [s]
simDays = 1;               % Simulation duration [days]
StopTime = simDays*86400;  % Total seconds

% Environmental conditions (Solan, November)
T_mean = 15;               % Mean temperature [°C]
T_amp = 11;                % Diurnal swing [°C]
RH_mean = 50;              % Mean humidity [%]
RH_amp = 15;               % Humidity swing [%]

% Sensor dynamics (first-order lag ~2s)
tau_s = 2;                 % Time constant [s]
a = exp(-Ts/tau_s);       % Lag coefficient

% Sensor noise (1-sigma)
noise.T = 0.1;             % Temperature [°C]
noise.RH = 0.5;            % Humidity [%RH]
noise.VWC = 0.2;           % Soil moisture [%VWC]

% Soil moisture model
VWC_init = 60;             % Initial content [%]
VWC_min = 15; VWC_max = 70; % Bounds [%]
evap_per_hour = 0.15;      % Evaporation rate [%VWC/hour]
rain_pulse = 6;            % Irrigation amount [%VWC]
rain_every_hours = 36;     % Irrigation interval [hours]
evap_rate = evap_per_hour/3600;  % Evaporation [%VWC/s]

% Soil temperature (slower thermal response)
Tsoil_mean = T_mean - 1;   % Soil cooler than air
Tsoil_amp = 5;             % Smaller swing
Tsoil_lag_h = 2;           % Peak ~4h after air peak
tau_s_soil = 6;            % Slower sensor response [s]
a_soil = exp(-Ts/tau_s_soil);
noise.Tsoil = 0.05;        % Smoother [°C]
```

**Usage**:
```matlab
% Run before starting Simulink
init_nauni;  % All parameters now available in workspace
```

**Quick Customization**:
- Change duration: `simDays = 7;`
- Winter test: `T_mean = 10; T_amp = 12;`
- Drought test: `rain_pulse = 0;`
- High noise: `noise.T = 0.5; noise.RH = 2.5;`

### `diurnal_env.m` - Environmental Pattern Generator

**Purpose**: Creates realistic 24-hour temperature and humidity cycles

**Function**:
```matlab
[T_air_raw, RH_raw, T_soil_raw] = diurnal_env(...
    t, T_mean, T_amp, RH_mean, RH_amp, 
    Tsoil_mean, Tsoil_amp, Tsoil_lag_h)
```

**How it works**:
- Converts simulation time to hour of day (0-24)
- **Temperature**: $T(t) = T_{mean} + T_{amp} \sin\left(\frac{2\pi(hr-5)}{24}\right)$ (peaks ~3 PM)
- **Humidity**: $RH(t) = RH_{mean} + RH_{amp} \sin\left(\frac{2\pi(hr+7)}{24}\right)$ (peaks pre-dawn, inverse of temp)
- **Soil Temp**: Same as air but delayed and damped (realistic thermal lag)

**Called by**: Simulink model at each timestep to generate continuous synthetic sensor data

### `sensor_on.m` - Power Management Function
Used to match duty cycles of sensors.(Here On for 1 min after every 30 min)
in case it is changed on the device, please make changes in `sensor_on.m`

**Purpose**: Manages sensor duty cycle to simulate battery-powered operation

**Function**:
```matlab
on = sensor_on(t)  % Returns 0 (active) or 1 (sleep)
period = 31*60;   % total cycle: 31 minutes = 60s on + 30min off (Make changes in period and awake)
awake  = 60;      % 60 seconds active
```

**Cycle**: 31-minute period
- **60 seconds**: Active (measuring, returning 0)
- **30 minutes**: Sleep mode (no measurement, returning 1)

**Repeated pattern ensures**: Sensor only transmits data 3% of the time, saving power

## Simulink Model

### `nauni_fieldtesting.slx`

**Model Overview**: 
The main Simulink model integrates all simulation components into a unified environment testing platform.

**Expected Components** (typical structure):
1. **MATLAB function Block**: Creates temperature, humidity, and moisture profiles
2. **Sensor Blocks**: Implements sensor dynamics (lag), noise injection, and quantization
3. **Visualization**: Real-time scopes for monitoring during simulation
4. **Analysis Tools**: Export capabilities for post-processing in MATLAB


```![alt text](image-1.png)

## Running Simulations



### Running on Your Local Machine

#### Step 1: Download and Set Up

```bash
# Clone or download the repository
git clone https://github.com/gramdishatrust/VARTA-IoT
cd VARTA-IoT/simulations
```

#### Step 2: Open MATLAB

```matlab
% In MATLAB Command Window, navigate to simulations folder

% Add code folder to path
addpath(genpath('code/'));
``'
#### Step 3: Initialize Simulation

```matlab
% Load all parameters for Solan/Nauni site
init_nauni;

% Verify parameters loaded if needed
disp('Simulation ready. Current settings:')
fprintf('Duration: %.1f hours\n', StopTime/3600);
fprintf('Temperature: %.1f ± %.1f °C\n', T_mean, T_amp);
fprintf('Humidity: %.1f ± %.1f %%\n', RH_mean, RH_amp);
```

#### Step 4: Run the Simulation

```matlab (In command window)
% open the model on Simulink
open('path of nauni_fieldtesting.slx');

% Run simulation (from model - click Run button or Ctrl + T)

```

#### Step 5: View Results

view results on the X-Y graph in the SImulink model
![alt text](image.png)

### Quick Testing Scenarios

#### Scenario 1: Basic 24-Hour Test
```matlab
init_nauni;
simDays = 1;
StopTime = 86400;  % 1 day
sim('File location of nauni_fieldtesting.slx'); in matlab command window
% Verify diurnal pattern appears correctly
```

#### Scenario 2: One Week with Irrigation
```matlab
init_nauni;
simDays = 7;
StopTime = simDays * 86400;
rain_pulse = 6;        % Irrigation enabled
rain_every_hours = 36;
sim('File location of nauni_fieldtesting.slx');  in matlab command window
% Check moisture cycles with irrigation
```

#### Scenario 3: Drought Conditions (No Rain)
```matlab
init_nauni;
simDays = 5;
StopTime = simDays * 86400;
rain_pulse = 0;        % No irrigation
sim('File location of nauni_fieldtesting.slx');  in matlab command window
% Watch soil moisture drop monotonically
```

#### Scenario 4: Winter Conditions with High Noise
```matlab
init_nauni;
T_mean = 8;            % Cold
T_amp = 12;            % Larger daily swing
RH_mean = 80;          % Wet
noise.T = 0.3;         % 3x normal noise for robustness testing
noise.RH = 1.5;
simDays = 3;
StopTime = simDays * 86400;
sim('File location of nauni_fieldtesting.slx');  in matlab command window
% Verify algorithms handle noise
```

#### Scenario 5: Extended Monitoring (2 weeks)
```matlab
init_nauni;
simDays = 14;
StopTime = simDays * 86400;
sim('File location of nauni_fieldtesting.slx');   in matlab command window
% Long-term behavior assessment
```

