---
layout: post
title: "Building a 2-DOF Rocket Trajectory & Nozzle Sizing Simulator in MATLAB"
date: 2026-06-20 12:00:00 -0500
categories: [Aerospace, Simulation, MATLAB]
tags: [Flight Dynamics, Propulsion, Numerical Integration, Isentropic Flow]
excerpt: "A technical deep dive into developing a numerical flight simulator that bridges atmospheric trajectory physics with 1D isentropic nozzle design for preliminary launch vehicle sizing."
---

## Overview

During the preliminary design phase of any launch vehicle, engineers must answer two fundamental questions: *How much fuel do we need to reach our target altitude?* and *What shape should the engine nozzle be?* To explore these concepts, I built a 2-Degree-of-Freedom (2-DOF) trajectory and aerodynamic simulator in MATLAB. Unlike basic algebraic calculators that rely on static "loss margins," this script utilizes numerical integration to "fly" a virtual rocket through a standard atmosphere, dynamically calculating drag, mass depletion, and gravity turns millisecond by millisecond. It then uses the launch site's specific environmental data to calculate the ideal expansion ratio for the engine's nozzle.

Here is a breakdown of the math, the physics, and the code architecture behind the simulator.

## 1. The Core Optimization Engine

The primary goal of the script is to calculate the required propellant mass to reach a highly specific target apogee. To do this, I implemented a `while` loop that acts as a numerical optimizer.

Instead of solving the Tsiolkovsky Rocket Equation analytically—which is impossible to do perfectly when factoring in dynamic atmospheric drag—the script makes an initial guess for propellant mass (`m_prop_guess`) and runs a flight simulation.

    while abs(apogee_achieved - target_apogee_m) > tolerance
        % ... [Flight Integration Loop Runs Here] ...
        
        if apogee_achieved < target_apogee_m
            m_prop_guess = m_prop_guess + dm_step; % Need more fuel
        elseif apogee_achieved > target_apogee_m
            m_prop_guess = m_prop_guess - dm_step; % Too much fuel
            dm_step = dm_step / 2; % Refine the step size
        end
    end

If the rocket falls short of the target, the script adds a mass step (`dm_step`) and re-flies the rocket. If it overshoots, it subtracts fuel and halves the step size to hone in on the exact required mass, converging typically within 15-20 iterations.

## 2. Dynamic Environmental Modeling

Aerodynamic drag is highly dependent on air density ($\rho$), which decreases exponentially as altitude increases. To model this accurately, the simulation uses the International Standard Atmosphere (ISA) barometric formula.

The physics equations governing the atmosphere at a given altitude ($z$) are:

$$T = T_0 - L \cdot z$$

$$P = P_0 \left( \frac{T}{T_0} \right)^{\frac{g_0}{R_s L}}$$

$$\rho = \frac{P}{R_s T}$$

Where $T_0$ is sea-level temperature, $L$ is the temperature lapse rate ($0.0065 \text{ K/m}$), $P_0$ is sea-level pressure, and $R_s$ is the specific gas constant for dry air.

By translating this into MATLAB, the simulator dynamically calculates the exact air density for the rocket's current altitude at every time step ($dt = 0.01\text{ s}$), accounting for the specific elevation of the launch site (e.g., launching from Denver vs. Sea Level).

    % Calculate dynamic air density (rho) based on current true altitude
    true_altitude_m = launch_elev_m + z;
    temperature = 288.15 - 0.0065 * true_altitude_m; % Kelvins
    pressure_pa = 101325 * (temperature / 288.15)^(9.81 / (287.05 * 0.0065)); 
    rho = pressure_pa / (287.05 * temperature); % kg/m^3

## 3. 2-DOF Kinematics & The Gravity Turn

To simulate realistic orbital or long-range suborbital trajectories, the rocket cannot fly strictly vertically. The simulator models a **Pitchover Maneuver** to initiate a gravity turn.

Once the rocket clears the launch tower height (`pitchover_alt_m`), the flight computer commands a slight tilt (`pitch_angle_deg`). After this initial kick, the rocket naturally follows its velocity vector.

We must resolve the forces—Thrust ($F_T$), Drag ($F_D$), and Gravity ($F_g$)—into horizontal ($X$) and vertical ($Z$) components using the flight angle ($\theta$). The aerodynamic drag equation is:

$$F_D = \frac{1}{2} \rho A C_d v^2$$

In the code, strict sign convention is maintained to ensure drag always opposes the velocity vector, and gravity acts purely in the negative $Z$ direction.

    % 2D Kinematics & Strict Sign Convention
    v_mag = sqrt(vx^2 + vz^2);
    F_drag_total = 0.5 * rho * area_m2 * Cd * v_mag^2; 

    F_drag_x = -F_drag_total * cos(flight_angle); 
    F_drag_z = -F_drag_total * sin(flight_angle); 
    F_gravity_z = -m_current * g0;

    F_thrust_x = thrust * cos(flight_angle);
    F_thrust_z = thrust * sin(flight_angle);

Using Euler integration, the net forces are divided by the instantaneous mass of the rocket ($m(t) = m_{dry} + m_{propellant} - \dot{m}t$) to find acceleration, which is then integrated to update velocity and position.

## 4. Thermodynamics: Sizing the Engine Nozzle

The final module of the simulator bridges software and hardware by designing the physical geometry of the rocket nozzle. To maximize thrust, the pressure of the exhaust gases exiting the nozzle ($P_e$) should perfectly match the ambient atmospheric pressure ($P_a$) at the launch site.

Using 1D Isentropic Flow equations for a compressible gas, we first calculate the Exit Mach Number ($M_e$) based on the engine's chamber pressure ($P_c$):

$$M_e = \sqrt{ \frac{2}{\gamma - 1} \left[ \left(\frac{P_c}{P_a}\right)^{\frac{\gamma - 1}{\gamma}} - 1 \right] }$$

Once $M_e$ is known, we calculate the **Expansion Ratio** ($\epsilon = A_e / A_t$), which dictates exactly how much larger the exit bell must be compared to the throat constriction:

$$\frac{A_e}{A_t} = \frac{1}{M_e} \left[ \frac{2}{\gamma + 1} \left( 1 + \frac{\gamma - 1}{2} M_e^2 \right) \right]^{\frac{\gamma + 1}{2(\gamma - 1)}}$$

I encapsulated this thermodynamic math into a dedicated MATLAB function at the end of the script:

    function [exp_ratio, M_e] = calculate_ideal_nozzle(Pc_psi, Pa_pa)
        gamma = 1.2; % Specific heat ratio for typical exhaust
        Pc_pa = Pc_psi * 6894.76; % Convert PSI to Pascals
        
        % Find Exit Mach Number
        M_e = sqrt( (2 / (gamma - 1)) * ( (Pc_pa / Pa_pa)^((gamma - 1)/gamma) - 1 ) );
        
        % Find Area Expansion Ratio (Area_exit / Area_throat)
        term1 = 2 / (gamma + 1);
        term2 = 1 + ((gamma - 1) / 2) * M_e^2;
        exp_ratio = (1 / M_e) * (term1 * term2)^((gamma + 1) / (2 * (gamma - 1)));
    end

By dynamically feeding the exact barometric pressure of the launch elevation (e.g., the thinner air of Denver vs. the thick air of Cape Canaveral) into this function, the script outputs the exact geometric ratios required for CAD modeling.

## Future Developments

The next phase of this project involves writing a script to automatically export the generated nozzle expansion ratio into a `.csv` coordinate curve, allowing for automated spline generation and revolving in CAD software.
