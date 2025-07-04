# # Autonomous Vehicle - Lane Following with Obstacle Avoidance (RL)

This project was developed for the "Introduction to Intelligent Robotics" course and aims to make a vehicle, in this case the **Toyota Prius**, able to follow the **road line**, **staying in its lane** and **avoiding obstacles** that appear in front of it. This project must be developed using **Webots and Reinforcement Learning**. Second Semester of the Third Year of the Bachelor's Degree in Artificial Intelligence and Data Science.

<br>

## Programming Language:

<div style = "display: inline_block"><br/>
  <img align="center" alt="python" src="https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white" />
</div><br/>

<br>

## Requirements:

	- python: 3.12.3
	- Webots: R2025a
	- Matplotlib == 3.10.3
	- Gymnasium == 1.1.1
	- Torch == 2.7.0
	- Tensorboar == 2.19.0
	- Numpy == 2.2.6
	- Opencv-python == 4.11.0.86
	- Stable-baselines3 == 2.19.0

You can use other versions of these libraries as long as they are compatible with each other and with the webots installed.

<br>

## Setting up to run the Project

In order to facilitate the use of our code, we provide the requiremts.txt of the virtual environment we used for the project. If you want to use it to facilitate the installation of libraries and dependencies, you should do the following:

1. We recommend the creaton of an enviroment, if you don't want to do this, skip to step [3]

You can create an environment by typing this on your terminal:
```bash
python3 -m venv webots-env
```
This code allows you to create an environment called `webots-env`.
If you don't have `venv` installed you can do this:

```bash
sudo apt install python3-venv python3-full
```
2. Once you have your virtual environment you need to activate it by typing:

```bash
source webots-env/bin/activate
```
We are considering that you created the virtual environment at `~`. If that's not the case you need to write the correct path or move to the correct folder.

3. When you are ready to install the requirements, download the file `requirements.txt` and type this:

```bash
pip install -r requirements.txt
```

If you are doing this on a virtual environment don't forget to activate it.

<br>

## The Project

This project focuses on developing an **autonomous driving agent** with two core capabilities:

-   **Lane following**
    
-   **Obstacle avoidance**
    
The goal is to evaluate how well **Reinforcement Learning (RL)** algorithms like **PPO** and **TD3** perform in realistic driving scenarios, using the **Webots** simulator with camera and LiDAR sensors. RL is used here as an alternative to classical control, which often fails in complex, unpredictable situations.

<br>

### Sensor Setup:

The Toyota Prius model in Webots was equipped with a **minimal set of sensors** required for autonomous navigation, without altering the physical structure of the vehicle. The four key sensors added are:

-   **LiDAR**: Mounted on top of the car, the LiDAR was configured with 24 vertical layers, a wide **horizontal field of view of 2.094 radians**, and a **vertical FoV of 0.5 radians**. It detects obstacles within a **maximum range of 4 meters** and provides a dense depth map of the surroundings. This is used to identify nearby static or dynamic obstacles and assess the navigability of the path ahead.
    
-   **Camera**: Positioned at the front of the vehicle, the camera shares the same **2.094 radians FoV** and a **maximum range of 6 meters**. It faces the ground and is primarily used for detecting road lane markings. The camera feed is processed into grayscale and further transformed into different visual formats (ROI, warped) to support lane-following behaviors.
    
-   **TouchSensor**: A contact sensor placed at the front bumper detects collisions with other objects. The sensor triggers a query to the world manager, which classifies the type of object involved (e.g., static barrel, dynamic obstacle).
    
-   **GPS**: Used to monitor the vehicle’s position relative to a predefined goal or target destination. This is critical for computing the **progress reward** and determining goal completion.
    
Additionally, the **vehicle speed** is accessed programmatically through the Webots Driver API. This provides a real-time scalar value representing the vehicle’s velocity in km/h, which is also included in the observation space.

<br>

### Observation and Action Space:

**Observation Space:**
The agent receives a **partially observable** state representation composed of three main inputs:

1.  **Camera (ROI image)**:
    -   The raw camera image is converted to grayscale and resized to 64×64.
    -   A crop of the upper 32 rows — the **Region of Interest (ROI)** — isolates the road ahead.
    -   This 32×64 segment is then **flattened into a 2048-element vector**, emphasizing lane visibility while reducing irrelevant background.
        
2.  **LiDAR Vector**:
    -   The LiDAR returns a **2D range image** with 24 vertical layers and 512 horizontal points, which is **flattened into a 12,288-element vector**.
    -   This provides dense spatial awareness about obstacles in all directions around the vehicle.
        
3.  **Speed Scalar**:
    -   A single float value representing the car’s current speed in km/h is appended to the observation.

The **final observation vector has a dimensionality of 14,337** and captures spatial, visual, and kinematic information.

<br>

**Action Space:**
This is a **continuous control task**, with the following two dimensions:

-   **Steering**: A float in the range **[-0.5, 0.5]**, controlling left/right direction.
    
-   **Throttle/Brake**: A float in the range **[-1.0, 1.0]**
    -   Positive values increase speed (acceleration).
    -   Negative values apply braking proportional to the absolute value.
        

These actions are applied via the Webots Driver interface. Speed is limited to a maximum of 40 km/h, and if throttle is negative, the braking system is activated.

This fine-grained action space enables the agent to make smooth directional and speed adjustments.

<br>

### Environment Setup:
To evaluate learning under different navigation conditions, **two Webots simulation maps** were created:

1.  **Straight Road** – ideal for testing **lane following** without interference.
2.  **Curved Map** – introduces turns and complexity, testing generalization and dynamic control.
    

**World Manager (Supervisor Robot):**
Because vehicles in Webots cannot act as simulation supervisors, a secondary robot called the **World Manager** was implemented to:

-   Handle **resets** between episodes.
-   **Spawn or remove obstacles** (e.g., barrels) with random positioning.
-   **Provide communication** to the vehicle regarding:
    -   Current goal position
    -   Type of collision (static vs dynamic)
        
At the end of each episode, the car writes a reset signal file, which the world manager reads. Upon detection, it:

-   Resets the car’s position and orientation.
-   Randomizes the obstacle setup for the next run (if desired).

This creates a **partially randomized environment** suitable for robust RL training. Determinism can be enabled by disabling obstacle variation.

<br>

### Reward Function:
The reward function was carefully **designed to balance multiple driving objectives** and ensure that the agent learns safe and efficient navigation. It combines **dense feedback** from various sensors to shape behavior:

### Reward Components:

-   **Lane Keeping**:
    -   Using the warped image, the system computes how far the car is from the center of the lane.
    -   Aligned center → +5.0
    -   Small deviations → +2.0 or +0.5
    -   Large deviations → +0.1
    -   No lane detected → -5.0
        
-   **Obstacle Avoidance**:
    -   The LiDAR vector is split into three zones: left, center, right.
    -   Penalties are computed based on **minimum and average distance** in each zone.
    -   Zones closer to obstacles are penalized more (especially the center).
        
-   **Collision**:
    -   Static object → -10.0
    -   Dynamic object → -20.0
    -   Detected via TouchSensor + classification by the world manager.
        
-   **Harsh Braking**:
    -   If braking intensity exceeds 0.6 → penalty of -0.5
        
-   **Off-road Detection**:
    
    -   Based on the grayscale image, the system checks:
        -   If there are enough **dark pixels** (road surface)
        -   If there are enough **bright pixels** (lane lines)
    -   Failing either → penalty of -10.0
        
-   **Progress toward Goal**:
    -   Progress per timestep is computed using GPS.
    -   Δ > 0.2 → +4.0
    -   Δ > 0.1 → +2.0
    -   Δ > 0.001 → +0.5
    -   No movement or moving away → -15.0
        
-   **Goal Completion**:
    -   If within 5 meters of the target → +50.0 terminal reward
        

This **modular reward function** ensures that the agent learns:
-   To stay centered on the lane
-   To avoid obstacles and collisions
-   To brake safely
-   To prioritize forward progress toward the goal

<br>

## How to run the project:
There are some things you must know to run our code. We have two robots, the Toyota Prius and the WorldManager. Creating a project folder from Webots comes in a predefined format. There will be a folder named `controllers`; in this folder, you should put all the codes you need for your project to work. In our case, in the world's file, we have predefined the controller that each robot uses, which means that inside the `controllers` folder, we need a folder for each robot with the name of the controller and a file with that same name that will run the whole thing.
Taking this into account, we put the file `world_manager.py` in a folder named `world_manager` and everything else in a folder named `prius_train_env.`
As you may have noticed, the folder `prius_train_env` doesn't have a Python file named `prius_train_env` because we can choose what we want the vehicle to do. We have three files: `train.py,` `test.py,` and `continue.py.` If we wish to run the code from the beginning, we can change the name of `train.py` to `prius_train_env.py` and run the project. If we're going to test a model, we must also change the name of the file to the name of the folder, and to continue training the file, we must do the same thing.

```
IRIproject
├── libraries
├── plugins
├── protos
├── controllers
│   ├── prius_train_env
│   │   ├── environment.py
│   │   ├── metrics_callback.py
│   │   ├── observation.py
│   │   ├── reward_helpers.py
│   │   ├── rewards.py
│   │   ├── train.py
│   │   ├── test.py
│   │   └── continue.py
│   └── world_manager
│       └── world_manager.py
├── worlds
│   ├── train_fase_1_straight
│   ├── train_fase_1_curves
│   ├── train_fase_2_obs_straight
│   └── train_fase_2_obs_curves
```

To run the project you can type:

```bash
webots --stout --stderr ~/IRI/worlds/train_fase_1_straight
```
This makes the world start running, along with the controller that is activated as prius_train_env, there is no need to do anything else. the `--stdou` and `--stderr` make the webots prints appear in the terminal as well. If you want the code to run faster, you can manually turn off rendering in webots or add this to the terminal code `--no-rendering`.

<br>

## About the repository:

- Assignment.pdf ➡️Project statement
- ProjectProposal.pdf ➡️Initial Proposal for the Project Development
- Paper ➡️ Paper about the entire Project (finished Project)
- environment.py ➡️ The code of the environment;
- metrics_callback.py ➡️ Code that allow to gather metrics during training;
- observation.py ➡️ Code that gets the observation from the sensors (image and lidar);
- reward_helpers.py ➡️ Functions that help calculating the rewards;
- reward.py ➡️ Code that computed the reward;
- world_manager.py ➡️ Controller of the 'ghost' robot;
- train.py ➡️ The code that trains the model from the beggining;
- teste.py ➡️ Code that testes the a chosen model;
- continue.py ➡️Code that allow to continue the train of a given model.

<br>

## Link to the course: 

This course is part of the **<u>second semester</u>** of the **<u>third year</u>** of the **<u>Bachelor's Degree in Artificial Intelligence and Data Science</u>** at **<u>FCUP</u>** and **<u>FEUP</u>** in the academic year 2024/2025. You can find more information about this course at the following link:

<div style="display: flex; flex-direction: column; align-items: center; gap: 10px;">
  <a href="https://sigarra.up.pt/fcup/pt/ucurr_geral.ficha_uc_view?pv_ocorrencia_id=546536">
    <img alt="Link to Course" src="https://img.shields.io/badge/Link_to_Course-0077B5?style=for-the-badge&logo=logoColor=white" />
  </a>

  <div style="display: flex; gap: 10px; justify-content: center;">
    <a href="https://sigarra.up.pt/fcup/pt/web_page.inicial">
      <img alt="FCUP" src="https://img.shields.io/badge/FCUP-808080?style=for-the-badge&logo=logoColor=grey" />
    </a>
    <a href="https://sigarra.up.pt/feup/pt/web_page.inicial">
      <img alt="FEUP" src="https://img.shields.io/badge/FEUP-808080?style=for-the-badge&logo=logoColor=grey" />
    </a>
  </div>
</div>
