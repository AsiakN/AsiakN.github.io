---
title: Why Localization in Robotics Needs Quaternions
date: 2025-09-22
math: true
---

In robotics, **localization**, knowing where your robot is and how it is oriented in space is critical. Lose track of that, and you may lose your robot. The robot’s position and orientation together are referred to as its **pose**.  

There are several ways to mathematically describe orientation. In robotics, computer graphics, and 3D animation, the most common representations are:  

1. **Euler angles**  
2. **Rotation matrices**  
3. **Quaternions**  

Let’s walk through each of these, and see why quaternions often end up as the preferred choice in robotics.  

---

## 1. Euler Angles  
Euler angles describe orientation using a sequence of three rotations (e.g., roll, pitch, and yaw). According to Euler’s rotation theorem, any 3D rotation can be expressed this way.  

Euler angles are intuitive; “rotate 45° around X” is easy to visualize. However, Euler angles suffer from a critical problem: **gimbal lock** (more on this below).  

## 2. Rotation Matrices  
Rotation matrices use linear algebra to represent orientation. Matrices easy to apply if you’re comfortable with matrix math: just multiply matrices to combine rotations.  

However rotation matrices are **redundant**. A rotation matrix in n-D is an nxn matrix. So for 3D orientation, is a 3x3 matrix which means it uses **nine numbers** to represent just **three degrees of freedom**. Taking a single matrix element is a floating-point nunber, that’s memory-heavy (36 bytes to store orientation alone) and computationally expensive as it requires multiple matrix multiplications are to combine rotations.  


## 3. Quaternions  
Quaternions take a different approach. A quaternion is a complex number in 4-dimensional space. They’re a compact, 4-dimensional extension of complex numbers that represent 3D rotations elegantly. If complex numbers describe 2D rotations, quaternions extend the idea into 3D. Thanks to Hamilton for the discovery.

**Disclaimer**: You may need time to understand quaternion concept fully because it's unintuitive and abstract. Some parts you just to have accept the beauty of the mathematics until you develop an intuition for it later. I found starting with complex numbers in 2D ([resource 1](#) and [resource 2](#) are great starters), then extend the idea to 4D.  

---

## Why Robots Use Quaternions  

**1. No Gimbal Lock**  
Unlike Euler angles, quaternions don’t suffer from gimbal lock—a condition where two rotational axes align, causing a loss of one degree of freedom. In robots, especially **6-DOF underwater vehicles**, this is critical. Imagine a robot navigating dangerous, underexplored terrain: if gimbal lock occurs, it loses the ability to rotate freely, which could compromise the entire mission.  

**2. Efficiency**  
Quaternions store orientation in just four values instead of nine. They’re lightweight and require fewer computations compared to rotation matrices, making them better suited for real-time robotic systems where efficiency matters.  

---

## The Tradeoff  
The main drawback of quaternions? They’re **unintuitive** to end users. Explaining “roll 45°” is straightforward, but a quaternion like `(0.707, 0, 0.707, 0)` doesn’t exactly roll off the tongue.  

Still, for control engineers, the benefits far outweigh the complexity.  


Orientation representation is a complex issue and might sound like abstract math, but in robotics it’s important for precise control. Quaternions strike the balance: compact, efficient, and reliable. They may take some time to grasp, but once you appreciate ir, you’ll see why they’re important for robotics.  


Written by me for me
