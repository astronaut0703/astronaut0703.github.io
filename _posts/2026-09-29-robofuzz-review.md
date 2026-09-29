---
title: "[Paper Review] RoboFuzz: Fuzzing Robotic Systems over Robot Operating System (ROS) for Finding Correctness Bugs"
date: 2026-09-29 12:00:00 +0900
categories: [Paper Review]
tags: [robofuzz, ros, ros2, fuzzing, cps, px4, robotics]
description: "A review of RoboFuzz (ESEC/FSE 2022), a fuzzing framework that finds correctness bugs in ROS-based robotic systems — bugs where the robot does not crash but behaves incorrectly."
---

A review of RoboFuzz (ESEC/FSE 2022), a fuzzing framework that finds correctness bugs in ROS-based robotic systems — bugs where the robot does not crash but behaves incorrectly.

> **RoboFuzz: Fuzzing Robotic Systems over Robot Operating System (ROS) for Finding Correctness Bugs**  
> Seulbae Kim, Taesoo Kim (Georgia Institute of Technology)  
> ESEC/FSE 2022 · [Paper](https://dl.acm.org/doi/10.1145/3540250.3549164)

## 1. What security threat was this paper responding to?

ROS (Robot Operating System) is an open-source middleware widely used to build robots, and it has become a de facto standard in industry, the military, and research. Despite "OS" in its name, it is not an operating system that directly controls computer hardware. It runs on top of an operating system such as Linux and provides the communication, libraries, and tools needed for robot development.

The paper's motivation is as follows.
Because so many robots use the same ROS, if ROS itself has a flaw, or if developers widely misuse ROS in the same way, a large number of robots are put at risk all at once. ROS 1 is the representative example. ROS 1 had no authentication, so anyone who knew the IP and port could connect to a robot, eavesdrop on its internal messages, and hijack its behavior. In fact, scanning the internet for two months found more than 100 ROS systems in 28 countries exposed as-is.

ROS 2 was designed with these problems in mind. However, existing research and solutions stayed at network security such as authentication and authorization, or at unit and regression testing. There was no way to **systematically find not-yet-known bugs** that undermine whether a robot **behaves correctly**. This paper tries to fill exactly that gap. In particular, it focuses on **"correctness bugs"**, where the program does not crash but the robot moves incorrectly.

The paper divides correctness bugs into three types.

| Type | In plain terms |
| --- | --- |
| Violation of physical laws | The robot's state makes no physical sense (e.g., the position changed but the velocity is 0) |
| Violation of specifications | The robot does not respect the limits written in its documentation or datasheet (e.g., it exceeds a joint limit) |
| Discrepancy between simulator and real robot | The same robot behaves very differently in the simulator and in the real world |

## 2. The paper's threat model

One thing should be made clear first. This paper has no section called "threat model." So the content below is what I collected from the body of the paper about what it assumed when testing.

- **Fuzzing targets.** Unless the user specifies otherwise, every node that subscribes to (receives data from) at least one topic becomes a fuzzing target. Mutated messages are sent to these nodes (§3.2).
- **Kind of input.** Only well-formed messages are used: messages with the correct type and the correct shape. The paper calls this "positive testing" (§7).
- **Inputs fed to the PX4 drone.** Three kinds: ① target position/attitude/velocity/acceleration (setpoint) messages, ② control commands corresponding to remote-controller stick values, and ③ configuration parameters changed at runtime (§4.2).
- **What is not covered.**
    - Flaws in lower communication layers such as DDS, for example a system crashing on a malformed message (§7, future work)
    - Physical attacks applied directly to the robot (§7, future work)
    - Fault injection such as turning off GPS with a PX4 shell command (§5.3)
- **Relationship to network security.** The paper itself says it is **orthogonal** to network security research (§8).

In one line: this is not a security threat model that assumes what an attacker can do. It is a test model that checks **"Does the robot move in a way that contradicts physical laws, specifications, or the real world even when only well-formed inputs are given?"**

## 3. Research methodology

### 3.1 Background: What does ROS 2 look like?

![ROS 2 architecture](/assets/img/%5BRoboFuzz%5Dimg1.png)

ROS 2 is divided into the application layer written by the developer (1) and the internal layers beneath it that handle data delivery (2–6).

| No. | Layer | What it does |
| --- | --- | --- |
| ② | RCL API (rclcpp, rclpy) | APIs the developer calls directly from C++ and Python |
| ③ | RCL | Core library shared regardless of language. Implements concepts such as nodes and topics |
| ④ | RMW | Layer that wraps DDS implementations from different vendors so they can be used the same way |
| ⑤ | DDS | Communication middleware that actually sends and receives messages |
| ⑥ | ROSIDL | Type system that defines message types, generates code, and checks whether types match |

A ROS application consists of three things.

- Node: a process that does one job
- Topic: a channel through which nodes exchange data
- Message: the data that travels over a topic

When a node publishes a message to a topic, the nodes subscribed to that topic receive it. Each topic accepts exactly one message type.

![ROS nodes, topics, and messages](/assets/img/%5BRoboFuzz%5Dimg2.png)

The paper gives three reasons why robots are hard to test, and proposes a design for each (§2.1).

| Difficulty | The paper's response |
| --- | --- |
| Every robot has different hardware and a different environment | It focuses on what they have in common, that a robot is ultimately "a collection of processes exchanging data," and fuzzes based on the **message flow** |
| There are too many possible states | It adds robot-specific feedback to fuzzing to guide exploration |
| Sensor and motor values contain noise | It runs the simulator and the real robot **at the same time** |

### 3.2 Overall flow

![RoboFuzz overview](/assets/img/%5BRoboFuzz%5Dimg3.png)

Fuzzing is a method of finding bugs by repeatedly feeding a program inputs that are changed little by little. RoboFuzz repeats this process in the following order.

1. System Inspector: runs the robot and figures out which nodes and topics exist and which message type each topic uses.
2. Message Mutator: changes (mutates) messages little by little and sends them to nodes.
3. Hybrid Executor: sends the same message to the simulator and the real robot at the same time, and records the state of both.
4. Oracle Handler: if nothing is wrong, it scores how far this input pushed the robot toward a "state close to a bug." If the score rises compared to before, the input is saved as material for the next mutation.

### 3.3 Key idea ①: Type-preserving message mutation

A ROS topic (channel) only accepts messages of a fixed type. A message whose type does not match is rejected at the sending stage. So whenever RoboFuzz changes a message, it always makes sure only values that fit that type are produced.

- It takes the mutation methods of the fuzzer AFL (American Fuzzy Lop) (bit flips, byte flips, arithmetic operations, etc.) and maps an appropriate method to each ROS type.
- It adds methods that generate special values and boundary values that robot code often mishandles, such as NaN, INF, and the maximum value of a 32-bit unsigned integer.
- Users can also add their own mutation methods.
- It controls the timing of sending. For a robot that moves on its own after receiving something like a target coordinate once, only one message is sent. For a robot that must receive input continuously, like drone control signals, messages are sent in succession. How many to send, how often to send them, and how long to observe can be configured.

### 3.4 Key idea ②: Running the simulator and the real robot together

![Hybrid execution with simulator and real robot](/assets/img/%5BRoboFuzz%5Dimg4.png)

Testing with a simulator alone misses two things. One is the difference that arises because the simulator cannot imitate reality exactly, and the other is values that do not exist in the simulator at all, such as motor temperature. So RoboFuzz sends the same message to the simulator and the real robot at the same time. This way, it can capture values that are visible on only one side, as well as the moment when values present on both sides diverge. The environment inside the simulator was made as identical as possible to the real test site, down to the GPS coordinates, trees, and buildings.

### 3.5 Key idea ③: Oracles that judge what is "correct"

An oracle is the criterion for judging "Is this result correct?" To borrow the paper's framing, conventional bugs reveal themselves by crashing the program, but correctness bugs often do not show up right away. So what counts as correct has to be defined separately. RoboFuzz provides a framework for building oracles, and separate oracles were built for each robot.

- Common to all robots
    - Physical constraint oracle: position and orientation values must be physically possible. For example, the z position cannot be NULL, NaN, or INF.
    - Sanitizer oracle: checks the error signals raised by ASan/UBSan when the system terminates.
- PX4 drone
    - Parameter oracle: checks whether parameters go outside the ranges written in the documentation.
    - Collision oracle: detects collisions with a contact sensor.
    - Flight mode oracle: checks whether each mode behaves as documented. For example, moving sideways in takeoff mode is a bug.
- TurtleBot3 (wheeled robot)
    - Specification oracle: checks whether the maximum velocity and the LiDAR distance range are exceeded.
    - Target velocity oracle: if a velocity within the limits was commanded, the robot must reach that velocity.
- MoveIt 2 (PANDA robot arm)
    - Inverse kinematics oracle: checks whether it fails to find a way to move the arm to the target position even though one exists.
    - Joint limit oracle: checks whether the joint angle limits in the datasheet are exceeded.
    - Physical consistency oracle: checks whether values are normal numbers, and whether joint velocity is reported as 0 while the arm is moving.
- ROS internal layers

    ![ROS internal layer oracles](/assets/img/%5BRoboFuzz%5Dimg5.png)

    - API consistency oracle: compares whether the C++ API (rclcpp) and the Python API (rclpy), when doing the same thing, call the same internal functions and produce the same result.
    - Type system oracle: checks five rules. Putting in a value of a different type, a value larger than the maximum, a value smaller than the minimum, or a value of a different type into an array element must fail; everything else must succeed.

### 3.6 Key idea ④: Feedback using robot knowledge

I think this is the most central idea of the paper.

Recent greybox fuzzers usually pick good inputs based on code coverage, that is, "how much new code did the input execute?" But a robot is a system whose state changes according to the data it receives, rather than according to the order in which code is executed. So even when the robot moves in many different ways, the code that is executed is almost the same. To complement code coverage, the paper uses knowledge about robots to score "how far this input pushed the robot into an abnormal state."

| Feedback | Idea | Example of application |
| --- | --- | --- |
| Redundant sensor disagreement | If two sensors measuring the same thing drift apart, the robot is unstable | Difference between the two IMUs on the Pixhawk 4 (Bosch BMI-055, TDK ICM-20689) |
| Control error | If the controller cannot reduce the gap between the target and the actual value, control is failing | Per-joint control error of the PANDA robot arm |
| Distance to limits | The closer a value is to a physical limit, the closer it is to a specification violation | Guides fuzzing toward the errors the specification oracle tries to catch (§4 gives no separate per-target example) |
| Simulator–real robot difference | Same input, but the states of the two robots differ | Result of the simultaneous execution in 3.4 |

The following robot-specific feedback was also added.

- **PX4:** Position estimation error. The distance between the raw GPS position and the position estimated by fusing multiple sensors (EKF).
- **TurtleBot3:** Rotation angle disagreement. The difference between the rotation angle computed from the IMU and the rotation angle computed from wheel rotation.

### 3.7 Results

![Bugs found by RoboFuzz](/assets/img/%5BRoboFuzz%5Dimg6.png)

- RoboFuzz is implemented in about 5.5k lines of Python and runs on ROS 2 Foxy.
- Targeting PX4, TurtleBot3, MoveIt 2, Turtlesim, ROSIDL, and rclcpp/rclpy, a separate fuzzing instance was created for each kind of input the target receives, and each was run for 12 hours.
- It found 30 new bugs, of which 25 were acknowledged by developers and 6 were fixed. 13 of the 30 (43%) are bugs in the ROS internal layers, so they can affect other robots that use ROS.
- All bugs were confirmed by replaying the recorded inputs, and the paper reports no falsely detected bugs (false positives).

Here are three of the bug cases introduced in the paper.

- **Bug where the PX4 drone loses control (#01–02):** The documentation defines a maximum horizontal acceleration `MPC_ACC_HOR_MAX` (default 5 m/s²). However, this limit is not applied to acceleration setpoints. So an abnormal value like `a_y = 3000 m/s²` goes straight into the controller and corrupts its internal state. Even if the setpoint is changed back to 0 afterward, the drone cannot return to a stable state.
- **Bug where TurtleBot3 ignores valid commands (#09–10):** The documented maximum velocity is 0.22 m/s. The firmware limits this to 0.2108 m/s and converts it to a wheel velocity value of 266.37. This value is smaller than the limit of 337 hardcoded in the motor driver, so it passes. But the motor's specified limit is 265, so the motor silently fails to carry out the command. In the end, the user sent a valid command, but the robot does not move.
- **Type mistake in Turtlesim (#16):** The return type of the `normalizeAngle` function, which normalizes an angle to −π to π, is wrongly declared as float. So when a very large angular velocity is given, the orientation value becomes NaN, and the turtle's position becomes physically impossible.

![Evaluation of feedback and coverage](/assets/img/%5BRoboFuzz%5Dimg7.png)

- **Effect of robot-knowledge feedback:** With feedback turned on, bug #06 occurred 9 times; with it turned off and only random mutation, it occurred 2 times.
- **Limits of code coverage:** PX4's branch coverage stopped at 21% within the first 10 minutes and barely increased over 12 hours. Meanwhile, the drone exhibited many different behaviors, including 9 specification violations.
- **Comparison with PGFuzz, an existing drone fuzzer:**
    - When the 21 rules used by PGFuzz were turned into RoboFuzz oracles and run, RoboFuzz found 26 of the 36 rule violations reported by PGFuzz. The remaining 10 required fault injection such as turning off GPS, so they were not inputs RoboFuzz handles.
    - Conversely, PGFuzz reportedly missed all of the new bugs RoboFuzz found. The paper gives three reasons: the rules manually extracted from documentation did not capture all specifications, PGFuzz cannot generate precisely timed continuous inputs, and its static analysis for finding relationships between inputs and parameters misses implicit relationships.

## 4. What this paper is trying to say

A robot is a cyber-physical system in which software and noisy hardware operate together. So conventional fuzzing approaches, which mainly look for memory-safety bugs, cannot catch bugs where the robot does not crash but quietly moves incorrectly (violations of physical laws, violations of specifications, and simulator–real robot discrepancies). This paper proposes three things.

1. A robot's behavior can be summarized as the message flow between nodes, so test it with fuzzing that mutates and injects messages.
2. Take what counts as correct from physical laws, specification documents, and the comparison between the simulator and the real robot, and turn it into oracles.
3. Code coverage alone is not enough for fuzzing to select good inputs, so complement it with robot knowledge such as sensor disagreement and control error. Using this method, the paper found 30 new bugs, including in the ROS internal layers, showing that the proposal is actually effective.

## 5. Questions I had after reading this paper

### 1. Who is responsible for input validation? And why didn't they fuzz with values that slipped past validation?

In kernel security, the principle is to validate values coming from user space at the boundary, so this question came to mind while reading the paper.

- The message mutator produces only type-correct values, on the assumption that "the type checking of the ROS type system (ROSIDL) works correctly" (§3.3).
- Yet this paper reports 8 bugs in that very type system (#18–25). For example, 65535 gets into an int8 array, and putting −32 into a uint8 array silently turns it into 224.
- In §4.6, the paper itself says that **ROS nodes often trust the type system and skip type checks or range checks**.

If so, the truly dangerous point is "what happens to a node that receives a strange value that quietly slipped past type checking." But this paper's mutator does not generate such values. The type system bugs were found with an oracle that checks type rules, and the paper contains no evaluation of how nodes behave when they receive those values. So this question remains: "What would have come out if the values that can slip through because of the type system bugs had been fed back into the mutator to fuzz the nodes?"

### 2. Why were sensor inputs excluded from the fuzzing targets?

The only inputs RoboFuzz fuzzes are ROS messages. Physical attacks were left as future work, and the 10 cases that required fault injection such as turning off GPS were not found. Yet this paper's feedback, namely the disagreement between the two IMUs, position estimation error, and rotation angle disagreement, all directly reflect anomalies in sensor values.

My thought is that if sensor inputs had also been included as fuzzing targets, they would have fit well with this feedback... I'm curious why they only went as far as ROS messages.

## 6. Shortcomings, lessons learned, and the research I want to do

### Shortcomings

1. It starts from a security problem, but because there is no threat model, it is impossible to judge how dangerous the found bugs are from a security standpoint.

    The introduction raises a security problem with ROS 1's lack of authentication and the 100+ systems exposed on the internet. But the body has no threat model, and §8 says its direction differs from network security research. For example, PX4 #01–02 is a serious bug in which a single abnormal acceleration setpoint leaves the drone unable to recover. However, who can send messages on that topic, and with what privileges, is not analyzed. So it is impossible to tell whether this bug is an accident (safety) problem or a security problem that an attacker can trigger remotely. It would have been more convincing if, for each of the 30 bugs, it had laid out who can trigger it and how.

### Lessons learned

1. Bugs that don't crash the program are a question of "where do you get the ground truth from."

    In kernel fuzzing, signals such as crashes or KASAN reports usually serve directly as the criterion for judging a bug. But in CPS, nothing crashes even if a drone flies faster than its specification or a motor silently ignores a command. This paper organizes the sources of ground truth into three.

    Physical laws (if the position changed, the velocity is not 0), specification documents (parameter ranges, datasheets), and the comparison of two worlds (simulator and real robot). I found it interesting that these three map one-to-one onto the three types of correctness bugs. By tracing the numbers in the TurtleBot3 case myself, from the documentation (0.22 m/s), to the firmware and motor driver (0.2108 m/s, hardcoded limit 337), to the motor specification (265), I came to see that CPS bugs arise less from a mistake in a single line of code and more from places where the assumptions of multiple layers do not agree with each other.

2. How to use knowledge about robots as a compass for fuzzing.

    While studying CPS on my own, redundant sensors, feedback control, and EKF were concepts I had learned separately.

    This paper turns them into a score that measures "how far this input pushed the robot toward a state close to a bug." The two IMUs normally output nearly the same values, but in extreme situations like a collision or flipping over, they diverge greatly. A growing control error means the controller is failing to track its target. A position estimate drifting away from GPS means the estimation is breaking down. Coverage stopped at 21%, but according to the paper, this score tended to increase over time in most runs.

    Seeing this, I learned that "executing new code" is not the only criterion for exploration, and that understanding the physics of the target system is itself the ability to design a good fuzzer.