**Institution:** University of Genoa<br>
**Course:** MSc in Robotics Engineering<br>
**Subject:** Experimental Robotics Laboratory<br>
**Author:** ***Alex Thanaphon Leonardi*** <thanaphon.leonardi@gmail.com><br>

# Assignment 1
[Full documentation here](https://thanaphonleonardi.github.io/exprob_assignment1/)

## 1. Introduction
This is the first assignment of the "Experimental Robotics Laboratory" course, for the Robotics Engineering degree, University of Genoa.

The goal of this assignment was to simulate, through an ontology, an environment
comprised of rooms and corridors, and to move a robot around these rooms based
on priority mechanisms. A Finite State Machine (FSM) represents the robot state.

The FSM is implemented through the [ROS SMACH library](http://wiki.ros.org/smach),
while the ontology is implemented through the [ARMOR framework](https://github.com/EmaroLab/armor).

## 2. Running the program
### Installation
This simulation runs on ROS Noetic. Clone this repository into your ROS workspace's src/ folder:
```
git clone https://www.github.com/ThanaphonLeonardi/exprob_assignment1
```
It also requires [ARMOR](https://github.com/EmaroLab/armor) (follow the
[tutorial presented here](https://github.com/EmaroLab/armor/issues/7)) and the
[ArmorPy API](https://github.com/EmaroLab/armor_py_api):
```
git clone https://github.com/EmaroLab/armor_py_api
```
Additionally, make sure to have [xterm](https://invisible-island.net/xterm/) installed:
```
sudo apt-get install xterm
```

### Execution
Launch the simulation:
```
roslaunch exprob_assignment1 assignment1.launch
```

## 3. Description
<img alt="Map of the environment" src="media/img/ontology_map.png" height="500"><br>
The environment is comprised of locations called **rooms** and **corridors**,
connected to each other by **doors**, and a **robot** that can occupy each of
these locations one at a time.<br>

- A **room** is an entity that can contain the **robot** and it is characterised
by having only *one* **door** connecting it to another location.<br>
- A **corridor** is a **room** with *two or more* **doors**.<br>

Each location is additionally characterised by a numeric data property called
*visitedAt* that represents the last moment it has been visited by the **robot**,
expressed in Unix time (since epoch).<br>

Before describing the **robot**'s behaviour, the concepts of *urgency* and
*reachability* should be introduced:
- a location is considered *reachable* if it is adjacent to the **robot**.
- a location is considered *urgent* if it has not been visited by the **robot**
for a certain amount of seconds, expressed by the *urgencyThreshold* property of
the **robot**.<br>

The **robot** exhibits the following behaviour: first, it looks for the next
*reachable* location to visit, following a priority list:
  1. urgent locations
  2. corridors
  3. rooms

Then, it goes to that location and waits for some time (to simulate work); after
which a new location is searched for and so on.

During its entire operation, the robot constantly monitors its own battery. If
the battery level falls below the first threshold (battery *low*), then the robot
will finish up its current work and then immediately after go charge itself. If,
however, the battery level falls below the second threshold (battery *critical*),
then the robot will immediately stop its work and find a charging station.

A location is considered a **charger** if it allows the **robot** to charge. In this
simulation, only the **corridor E** is considered a **charger**.

### Demo Video
![A Demonstration Video](media/gifs/demo.gif)<br>
Top left: **behaviour**<br>
Top right: **owl_interface**<br>
Left: **planner**<br>
Right: **controller**<br>
Bottom left: **robot_state**

This video shows what happens when the battery goes below the *LOW* threshold.
The planner sends the goal 'E' to the controller, which moves the robot and
starts to charge it.

## 4. Behind The Scenes
The simulation incorporates several elements: a Finite State Machine governs the
flow of events, while a component diagram describes the interactions among the
ROS nodes involved.

### Finite State Machine
Automatically generated image by [smach_viewer](http://wiki.ros.org/smach_viewer):
<br>
![State Machine diagram](media/img/exprob_ass1_smach_viewer.png)
<br>
Manually drawn:
<br>
![The Finite State Machine of the robot](media/img/state_diagram.png)<br><br>
The robot starts in state **CHARGING** with a battery level determined by the
global variable *STARTING_BATTERY_LEVEL* defined in **robot_state.py**, and
positioned in location 'E' (which can be changed from the ontology, if desired).<br>
If the battery is low, the robot will immediately charge. If the battery is full,
the robot will transition to state **PLAN_GOAL** where
it will look for the next location to move towards.<br>

Once a new goal is planned, the robot transitions to state
**MOVING** where it will move to change its location. If, while moving, the
robot detects critically low battery, the state will transition back to **CHARGING**.
However, since charging can only happen in location 'E', if the robot is not
in the right place, then a new plan will immediately be created to bring the
robot to 'E', and the robot will once again attempt to charge.<br>
If, instead, everything goes well and the robot reaches its final destination,
then it will transition to state **WAITING** where the robot will simulate
work by waiting in the new location for a random number of seconds.

If during this wait the battery gets critically low, then the robot will
immediately attempt to charge. If, instead, everything goes well and the robot
completes its work, then the transition to state **PLAN_GOAL** will be made and
the robot will look for a new goal.

### Component Diagram
![Component Diagram of the system](media/img/compdiag2.png)<br>
Here, the structure of the ROS nodes can be seen, along with the topics,  
services and action servers that link them together.

#### Component: OWL Interface
This component interfaces with the OWL ontology representing both the map and the
current situation (robot battery, timestamps, etc.). It exposes multiple topics
and services to allow other nodes to easily interact with the ontology
whilst hiding the implementation details.

The [ArmorPy API](https://github.com/EmaroLab/armor_py_api) is used
to facilitate operations.

#### Component: Robot State
This component interfaces the robot's state with the other nodes. Specifically, it
shares information regarding the robot pose, the next goal and the battery. It
also accepts pose updates to update the current pose in the ontology, and accepts
requests to change the battery mode to *charging* or *discharging*.

#### Component: Controller
This component receives the plan through an action server (which is comprised of only one goal in this
simulation) from the **behaviour** component (which in turn received the plan from
the **planner** component) and moves the robot through it.
Once the final destination is reached, or upon failure, the **behaviour** module
is notified.

This component also manages the robot's battery charging.

In this simulated program, only adjacent locations are considered *reachable*,
so the controller only ever has one waypoint to go towards. Therefore,
this component simulates work by busy waiting.

#### Component: Planner
This component receives the next goal from the **behaviour** component (which,
in turn, received it from the **owl_interface** component) and calculates the
shortest path to reach that goal, i.e. a set of viapoints.
These viapoints are then sent to the **controller** component through **behaviour**.

In this simulated program, only adjacent movements are made, therefore the planner does not
actually have to compute anything and instead simulates work by busy waiting.

#### Component: Behaviour
This component is the heart of the robot and manages its underlying Finite State
Machine (FSM).
The [ROS SMACH library](http://wiki.ros.org/smach) is used to represent the FSM.
States are non-concurrent and updated at a rate defined by the
*STATE_UPDATE_PERIOD* global variable.

### Temporal/Sequence Diagram
![Temporal/Sequence Diagram](media/img/temporal_diagram.png)
This diagram is kept purposefully simple. The important thing is to highlight the relationship between each node and their interactions. A detailed overview of the specific topics, services and action servers used can be read in the [full documentation](https://thanaphonleonardi.github.io/exprob_assignment1/).

### Extras
The **roslaunch** file is set by default to print to the terminal. The program is
capable of **logging** and can be set to do so, to the default ROS logs, by
modifying the launch file accordingly.

## 5. System Analysis
### Assumptions
1. The robot starts with full battery in state CHARGING and in location 'E', even though these can be
tweaked from the ontology and from the behaviour.py script.
2. The location 'E', even though it is special as it is the charging room, is still considered a possible "normal" goal for the robot.
3. Even if the battery reaches 0, the robot continues working. This is purposeful,
as it allows the simulation to function even with high battery discharge rates,
for analysis purposes.
4. The FSM is non-concurrent, so the robot can only be in one state at any given time.
5. The robot can freely *teleport* between reachable rooms, as there is no physical model
to actually control and move.
### Current Limitations and Future Improvements
#### 1. Path Planning Algorithm
Currently, only adjacent rooms are considered as viable targets. Therefore, no
path-planning algorithm has been implemented. The only moment in which the robot
would need one is if the battery becomes *low* and the robot is in room 'R2' or
'R4'.
In this case, a random goal is chosen, because in the current map this will
inevitably lead the robot to a corridor which is then adjacent to 'E'.
However, for bigger maps and more complex movements, a path-planning algorithm
should be implemented, such as the [A* Search Algorithm](https://en.wikipedia.org/wiki/A*_search_algorithm).
#### 2. Complex Battery Management
Currently, the robot is not that robust with regard to its battery management,
and the battery could run out before the robot has had the time to reach a
charging station.
Especially for bigger maps, the thresholds for *low* or *critical* battery levels should
depend on the robot's distance from charging stations, instead of just being
some predetermined values. Also, the robot should have some sort of *low power mode*,
for example when it is idling.
These improvements would allow for optimal use of the robot's battery.
#### 3. Actual "Work"
In this simulation, no real "work" is done by the planner or the controller.
Of course, some real task should be given to the robot, and the controller
expanded to include such a task.
