# VARTA-IoT Simulations

## Overview

This folder contains MATLAB/Simulink simulations for testing and validating the VARTA system under realistic environmental conditions. The simulations model the Solan/Nauni field site in Himachal Pradesh, India, and provide virtual sensor data streams for system validation before field deployment.

## Purpose

- **System Validation**: Test sensor algorithms and control logic without hardware
- **Climate Modeling**: Simulate realistic environmental conditions for the Himalayan region
- **Performance Prediction**: Estimate system behavior under various environmental scenarios
- **Field Testing Baseline**: Establish baseline expectations before actual field deployment

## Folder Structure

```
simulations/
├── code/                          # MATLAB functions and initialization scripts
│   ├── init_nauni.m              # Site-specific initialization and parameters
│   └── diurnal_env.m
    └── sensor_on.m          # Diurnal cycle generation function
├── model/                         # Simulink models
│   └── nauni_fieldtesting.slx    # Main Simulink simulation model
└── README.md                      # This file
```

### Prerequisites
- MATLAB R2020a or later(https://in.mathworks.com/downloads/web_downloads/13472755)
- MATLAB online can also be used
- Simulink
- <img width="2519" height="778" alt="image" src="https://github.com/user-attachments/assets/2b5bc45a-e5d4-4a67-b0b3-6159324f6bf7" />




**Typical Workflow**:
```matlab
% 1. Initialize simulation parameters
init_nauni;

% 2. Open Simulink model
open('nauni_fieldtesting.slx');

% 3. Run simulation
sim('nauni_fieldtesting.slx', struct('StopTime', num2str(StopTime)));

% 4. Analyze results in workspace
```

## Site Configuration (Solan/Nauni)

### Geographic Information (meta data)
- **Latitude**: 30.85° N
- **Longitude**: 77.18° E
- **Altitude**: 1350 m above sea level
- **Region**: Himachal Pradesh, India

### Typical November Climate (Baseline)

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
- **Minimum**: ~00:00 (pre-dawn)
- **Maximum**: ~11:00 (mid-afternoon)
- **Model**: Sinusoidal diurnal pattern
- **Equation**: $T(t) = T_{mean} + T_{amp} \sin\left(\frac{2\pi(h-5)}{24}\right)$
  - Where $h$ = hour of day (0-24)

#### Relative Humidity Cycle
- **Maximum**: Pre-dawn (~00:00-06:00)
- **Minimum**: Mid-day (~12:00-15:00)
- **Anti-phase**: RH rises as temperature falls
- **Model**: Sinusoidal pattern offset from temperature
- **Equation**: $RH(t) = RH_{mean} + RH_{amp} \sin\left(\frac{2\pi(h+7)}{24}\right)$

#### Soil Moisture Dynamics
- **Evaporation**: 0.2% VWC/hour under normal conditions
- **Irrigation Events**: 6% VWC added every 36 hours
- **Constraints**: Moisture bounded between 8% (dry) and 45% (saturated)
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

| Sensor | Noise Level | Unit |
|--------|-------------|------|
| **Temperature** | ±0.1 | °C |
| **Humidity** | ±0.5 | %RH |
| **Soil Moisture** | ±0.2 | %VWC |

## MATLAB Scripts

### `init_nauni.m` - Simulation Initialization

**Purpose**: Define all site parameters, simulation settings, and environmental conditions

**Key Variables**:
```matlab
% Site location
site.lat = 30.85;          % Latitude (°N)
site.lon = 77.18;          % Longitude (°E)
site.alt_m = 1350;         % Altitude (meters)

% Simulation timing
Ts = 1;                    % Sample time (seconds)
simDays = 3;               % Simulation duration (days)
StopTime = simDays*86400;  % Total simulation time (seconds)

% Environmental conditions
T_mean = 16;               % Mean temperature (°C)
T_amp = 7;                 % Temperature amplitude (°C)
RH_mean = 65;              % Mean relative humidity (%)
RH_amp = 20;               % Humidity amplitude (%)

% Sensor characteristics
tau_s = 2;                 % Sensor time constant (seconds)
a = exp(-Ts/tau_s);       % First-order lag coefficient

% Noise specifications
noise.T = 0.1;             % Temperature noise (1-sigma, °C)
noise.RH = 0.5;            % Humidity noise (1-sigma, %RH)
noise.VWC = 0.7;           % VWC noise (1-sigma, %VWC)

% Soil moisture model
VWC_init = 28;             % Initial volumetric water content (%)
VWC_min = 8;               % Minimum VWC (dry soil)
VWC_max = 45;              % Maximum VWC (saturated)
evap_per_hour = 0.2;       % Hourly evaporation rate (%VWC/hour)
rain_pulse = 6;            % Water added per irrigation (%)
rain_every_hours = 36;     % Irrigation interval (hours)
evap_rate = evap_per_hour/3600;  % Evaporation rate (%VWC/second)
```

**Usage**:
```matlab
% Run at start of Simulink session
init_nauni;  % Loads all parameters into MATLAB workspace
```

**Customization Points**:
- Adjust `simDays` to change simulation duration
- Modify `T_mean`, `T_amp`, `RH_mean`, `RH_amp` for different seasonal conditions
- Change `VWC_init`, `evap_per_hour`, `rain_pulse` for different moisture scenarios
- Tune `noise.*` values to test sensor robustness

### `diurnal_env.m` - Diurnal Cycle Generator

**Purpose**: Generate realistic hourly temperature and humidity patterns

**Function Signature**:
```matlab
[T_raw, RH_raw] = diurnal_sht40(t, T_mean, T_amp, RH_mean, RH_amp)
```

**Parameters**:
- `t`: Time in seconds
- `T_mean`: Mean temperature (°C)
- `T_amp`: Temperature amplitude (°C)
- `RH_mean`: Mean relative humidity (%)
- `RH_amp`: Humidity amplitude (%)

**Outputs**:
- `T_raw`: Unfiltered temperature value (°C)
- `RH_raw`: Unfiltered relative humidity value (%)

**Behavior**:
- Converts time `t` to hour of day: $h = \text{mod}(t/3600, 24)$
- Generates smooth sinusoidal patterns with physically realistic phase relationships
- Temperature peaks ~3 hours after humidity minimum
- Can be called at each simulation timestep to generate continuous sensor data

### `sensor_on.m` - Create on-off cycles for sensors

**Purpose**: Reduce battery consumption as sensors work for 60 second after every 30 min

**Function Signature**:
'function on' = sensor_on(t)   % t is simulation time in seconds
period = 31*60;   % total cycle: 31 minutes = 60s on + 30min off
awake  = 60;      % 60 seconds active
Change values of period and awake to change duty cycle of your sensors

## Simulink Model

### `nauni_fieldtesting.slx`
<img width="2717" height="1376" alt="image" src="https://github.com/user-attachments/assets/ee985554-c825-4e71-ad87-a75c081e6390" />


**Model Overview**: 
The main Simulink model integrates all simulation components into a unified environment testing platform.

**Expected Components** (typical structure):
1. **Environmental Generator Block**: Creates temperature, humidity, and moisture profiles
2. **Sensor Blocks**: Implements sensor dynamics (lag), noise injection, and quantization
3. **Visualization**: Real-time scopes for monitoring during simulation
4. **Analysis Tools**: Export capabilities for post-processing in MATLAB



## Running Simulations



### Basic Workflow

1. **Open MATLAB**
   in the simulations folder or add it to path:
   ```matlab (in command window)
   addpath(genpath('code/'));  % Add code folder
   ```

3. **Initialize parameters**: (in command window)
   ```matlab
   init_nauni;
   ```

4. **Open and run Simulink model**:                  (in command window)
   ```matlab
   open('file location of nauni_fieldtesting.slx');
   sim('file location of nauni_fieldtesting.slx');
   ```

5. **Access simulation results**
   Double click on X-Y scope to view graphs
  

### Advanced Usage

#### Change Simulation Duration                        (in command window)
```matlab
init_nauni;
StopTime = 7 * 86400;  % Change to 7 days
sim('file location of nauni_fieldtesting.slx');
```

#### Test Different Environmental Conditions             (in command window)
```matlab
init_nauni;
T_mean = 12;      % Winter conditions
T_amp = 10;       % Larger daily swing
RH_mean = 80;     % Higher humidity
RH_amp = 15;
sim('file location of nauni_fieldtesting.slx');
```

#### Increase Sensor Noise for Robustness Testing          (in command window)
```matlab
init_nauni;
noise.T = 0.5;    % 5x normal noise
noise.RH = 2.5;
noise.VWC = 3.5;
sim('file location of nauni_fieldtesting.slx');
```

#### Disable Irrigation to Test Drought Conditions           (in command window)
```matlab
init_nauni;
rain_pulse = 0;   % No irrigation
rain_every_hours = inf;  % Prevent any rain events
sim('file location of nauni_fieldtesting.slx');
```




