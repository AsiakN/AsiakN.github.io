---
title: Why Your Robot's IMU Is Lying to You
date: 2026-02-20
math: true
---

![Merry-Go-Round Lever Arm Animation](merry_go_round.gif)

Imagine you're standing on a spinning merry-go-round in a playground.

**Standing at the center:** You spin in place. You feel no pull outward. Your feet stay planted. You're rotating, but your body isn't being dragged anywhere.

**Standing at the edge:** Now you feel a strong pull outward (centripetal acceleration). The faster it spins, the harder you have to grip the railing. If someone suddenly speeds up or slows down the spin, you also lurch forward or backward (tangential acceleration).

Both you-at-the-center and you-at-the-edge are on the *same* spinning platform, but your *experience of motion is completely different* depending on how far you stand from the center.

Now replace the merry-go-round with a drone making a yaw turn, yourself with an accelerometer, and the center with the drone's center of mass. If the IMU sits right at the center of mass, it measures only the drone's true movement. But if it's bolted 10 cm off to the side — like standing at the edge of the merry-go-round — it also feels that outward pull and that lurching. It reports those as real accelerations.

The flight controller then thinks: *"I'm accelerating sideways!"* and tries to correct for motion that doesn't actually exist at the center of mass. The vehicle drifts, oscillates, or fights itself.

That's the **lever arm effect**: the farther a sensor sits from the center of rotation, the more "phantom motion" it feels from every twist and turn of the vehicle.

## The Physics

When a body rotates about its center of mass, any point at a distance **r** from the CoM experiences additional accelerations:

```
a_measured = a_CoM + α × r + ω × (ω × r)
```

Where:
- **a_CoM** — translational acceleration at the center of mass
- **α × r** — tangential acceleration from angular acceleration (Euler term)
- **ω × (ω × r)** — centripetal acceleration from angular velocity
- **r** — lever arm vector from CoM to sensor
- **ω** — angular velocity
- **α** — angular acceleration (dω/dt)

The centripetal term always points inward (toward the axis of rotation), the tangential term is perpendicular to the lever arm in the plane of angular acceleration.

### How Bad Can It Get?

Consider a UUV with an IMU mounted 0.3 m forward of the CoM. During a yaw maneuver at ω = 0.5 rad/s:

- Centripetal acceleration: ω² × r = 0.25 × 0.3 = **0.075 m/s²**
- Over 10 seconds of integration, that's a velocity error of **0.75 m/s** and a position error of **3.75 m**

And it only gets worse with faster rotation or larger offsets. The centripetal term grows with ω², so a vehicle doing aggressive maneuvers accumulates error fast.

## Compensation

The obvious fix is mechanical: mount the IMU as close to the CoM as possible. When that's not feasible, you subtract the spurious acceleration in software:

```
a_corrected = a_measured - α × r - ω × (ω × r)
```

This needs an accurate lever arm vector **r** (measure it during assembly), good gyroscope data, and a way to get α — either by differentiating ω numerically or estimating it in your filter.

The cleaner approach is to fold the lever arm into your state estimator. An EKF or UKF can include **r** as a known parameter in the measurement model, so the filter accounts for rotational coupling when fusing IMU data with other sensors. In ROS 2, [robot_localization](http://docs.ros.org/en/rolling/p/robot_localization/) does exactly this — its EKF and UKF nodes accept sensor offsets directly.

Don't forget the GNSS antenna. It's almost never co-located with the IMU, and if you don't account for that lever arm in your fusion algorithm, position solutions will oscillate during turns.

## What Drives the Error

- Lever arm distance — bigger offset, bigger phantom accelerations
- Angular rate — centripetal error grows with ω²
- Angular acceleration — tangential error scales linearly
- Gyro quality — noisy gyros make compensation noisy
- Integration time — uncompensated position error grows cubically
