---
layout: post
title:  "Migrating our robot simulator from MuJoCo 2.3 to 4.0: one number, lots of proof"
date:   2026-11-13 18:30:00 +0000
last_modified_at: 2026-11-13 18:30:00 +0000
categories: robotics simulation
excerpt: Two years pinned on MuJoCo 2.3.7 because of a weld-constraint solver change. How we measured it, fixed it with one attribute, and proved parity with the old version.
---

# Motivation

In April 2023, Tony Zhao and colleagues published
[ACT and ALOHA](https://arxiv.org/abs/2304.13705). This work introduces a cheap
bimanual robot using near-commodity hardware from Trossen Robotics combined with an imitation-learning method.

It is a highly influential paper: thousands of
labs and hobbyists built on it. The simulation side of that work — the
bimanual ViperX arms, the mocap-weld control, the scripted policies, the
recorded demonstrations — was written against **MuJoCo 2.3.7**, and it has
not been updated since the [initial release](https://github.com/tonyzhaozh/act).

But MuJoCo has advanced significantly since then - v4 in particular has additional support for deformable surfaces and filaments (e.g. tying a knot at the top of a rubbish bag).

# Moving from MuJoCo 2.3 to MuJoCo 4 - how and why

## How the robot arms are controlled

Our simulation does not drive the arm motors directly. Instead, we use the same trick that ACT does: command the gripper pose directly in Cartesian space, and let the physics pull the arm there, eliding the need for an Inverse Kinematics (IK) solver and joint-space controller tuning.

1. The software places an invisible body, called a *mocap body*, at the target pose.
2. A **weld** connects this body to the gripper.
3. A weld is an elastic constraint. It pulls the gripper to the target.

A weld has a group of five settings called `solimp` (short for *solver impedance*): dmin, dmax, width, midpoint, power. ACT sets the first three and uses defaults for the last two. `solimp` says how stiff the weld is. A low value makes a soft weld. A high value makes an almost rigid weld.

ACT used soft values — dmin and dmax of 0.25 — and the system was calibrated against this softness. The robot arms were allowed to lag and to hang a little.

Under MuJoCo 4, the old weld lets the gripper rotate away from its commanded orientation, so
the grasps miss.

**Old weld (`solimp=.25 .25`) on MuJoCo 4 — tower fails:**

<video controls muted loop width="100%" src="/images/compare_old_solimp25.mp4"></video>


## How we found the cause

We ran the same simulation task many times and measured the error between each gripper and its target - on 2.3.7 and 4. The result was clear:

- The position error did not change.
- The angle error became two times larger, and up to six times larger during motion.
- The task score fell from a full pass to almost zero.

So only the angle part of the weld had changed. The new solver made the same setting, 0.25, much softer than before.

To be clear - this is not a bug in either MuJoCo or the ACT code. It is a change of behaviour. MuJoCo documents that solver behaviour can change between major versions. The old number and the new number are both correct answers for a given release of MuJoCo.

## The fix

After a grid sweep where we tried many stiffness values, we discovered the new values that allowed the original ACT code to work with MuJoCo 4.

 ```
   before:  solimp=".25 .25 0.001"
   after:   solimp="0.99 0.99 0.001"
             dmin dmax width
 ```

 - dmin: .25 → 0.99 ✏️ changed
 - dmax: .25 → 0.99 ✏️ changed
 - width: 0.001 → 0.001 unchanged

Now the new weld holds the orientation and the tower is built.

**New weld (`solimp=0.99 0.99`) on MuJoCo 4 — tower built:**

<video controls muted loop width="100%" src="/images/compare_new_solimp099.mp4"></video>


With the new values, MuJoCo 4 tracked the target as well as the old one:

- The task score returned to a full pass.
- The angle error matched the old system to less than one degree.
- The old calibration data still worked.

## What we gained - modernity!

- MuJoCo 4.0.1 instead of 2.3.7 - including access to [new features](https://mujoco.readthedocs.io/en/stable/changelog.html) like elasto-plasticity.
- Python 3.14 instead of 3.10.
- Current versions of the tool libraries.
