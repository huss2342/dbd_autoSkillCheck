# DBD Auto Skill Check

The Dead by Daylight Auto Skill Check is a tool developed using deep learning techniques (PyTorch) to automatically detect and successfully hit skill checks in the popular game Dead by Daylight. 
This tool is designed to improve gameplay performance and enhance the player's skill in the game. 

## Features
- Real-time detection of skill checks (60fps)
- High accuracy in recognizing **all types of skill checks (with a 98.7% precision, see details of results below)**
- Automatic triggering of great skill checks through auto-pressing the space bar
- **[NEW]** Two different webUI to run the AI model


## Execution Instructions
I have only tested the model on my own computer running Windows 11 with CUDA version 12.3.

Create your own python env (I have python 3.11) and install the necessary libraries using the command :
`pip install numpy mss onnxruntime pyautogui IPython pillow gradio`

Then clone the repo. I provide two different scripts.

### Single prediction Web UI

Run this script to test the AI model with images of the game.

`python run_single_pred_gradio.py`

1) Select the trained AI model (default to `model.onnx` available in this repo)
2) Upload your image
3) Click Submit
4) On the right, it displays the sorted probabilities of the skill check recognition

| SINGLE prediction example 1                    | SINGLE prediction example 2                    |
|------------------------------------------------|------------------------------------------------|
| ![](images/single_pred_1.png "Example repair") | ![](images/single_pred_2.png "Example wiggle") |

### Monitoring Web UI

Run this script and play the game ! It will hit the space bar for you.

`python run_monitoring_gradio.py`

1) Select the trained AI model (default to `model.onnx` available in this repo)
2) Choose debug options. I recommend setting this option to None. If you want to check which screen the script is monitoring, you can select the first option. If the AI struggles recognizing the skill checks, select the second option to save the results, then you can upload the images in a new GitHub issue.
3) Click 'RUN'
4) You can STOP and RUN the script from the Web UI at will, for example when waiting in the game lobby.

Your main screen is now monitored meaning that frames are regularly sampled (with a center-crop) and analysed with the trained AI model.
You can play the game on your main monitor.
When a great skill check is detected, the SPACE key is automatically pressed, then it waits for 0.5s to avoid triggering the same skill check multiple times in a row.

| RUN example with a full white skill check (to hit) | RUN example with an ante-great repair skill check (to hit) |
|----------------------------------------------------|------------------------------------------------------------|
| ![](images/run_2.png "Example run 1")              | ![](images/run_3.png "Example run 2")                      |



**The AI model FPS must run at 60fps (or more) in order to hit correctly the great skill checks.** If you have low values, try this :
- Disable the energy saver settings in your computer settings
- Run the script in administrator

I had this problem of low FPS with Windows : when the script was on the background (when I played), the FPS dropped significantly. Running the script in admin solved the problem.



## What is a skill check

A skill check is a game mechanic in Dead by Daylight that allows the player to progress faster in a specific action such as repairing generators or healing teammates.
It occurs randomly and requires players to press the space bar to stop the progression of a red cursor.

Skill checks can be: 
- failed, if the cursor misses the designated white zone (the hit area)
- successful, if the cursor lands in the white zone 
- or greatly successful, if the cursor accurately hits the white-filled zone 

Here are examples of different great skill checks:

|     Repair-Heal skill check     |       Wiggle skill check        |       Full white skill check        |        Full black skill check         |
|:-------------------------------:|:-------------------------------:|:-----------------------------------:|:-------------------------------------:|
| ![](images/repair.png "repair") | ![](images/wiggle.png "wiggle") | ![](images/struggle.png "struggle") | ![](images/merciless.png "merciless") |

Successfully hitting a skill check increases the speed of the corresponding action, and a greatly successful skill check provides even greater rewards. 
On the other hand, missing a skill check reduces the action's progression speed and alerts the ennemi with a loud sound.

## Project details

### Dataset
We designed a custom dataset from in-game screen recordings and frame extraction of gameplay videos on youtube.
To save disk space, we center-crop each frame to size 320x320 before saving.

The data was manually divided into 11 separate folders based on :
- The visible skill check type : Repairing/healing, struggle, wiggle and special skill checks (overcharge, merciless storm, etc.) because the skill check aspects are different following the skill check type
- The position of the cursor relative to the area to hit : outside, a bit before the hit area and inside the hit area.

**We experimentally made the conclusion that following the type of the skill check, we must hit the space bar a bit before the cursor reaches the great area, in order to anticipate the game input processing latency.
That's why we have this dataset structure and granularity.**

To alleviate the laborious collection task, we employed data augmentation techniques such as random rotations, random crop-resize, and random brightness/contrast/saturation adjustments.

We developed a customized and optimized dataloader that automatically parses the dataset folder and assigns the correct label to each image based on its corresponding folder.
Our data loaders use a custom sampler to handle imbalanced data.

### Architecture
The skill check detection system is based on an encoder-decoder architecture. 

We employ the MobileNet V3 Small architecture, specifically chosen for its trade-off between inference speed and accuracy. 
This ensures real-time inference and quick decision-making without compromising detection precision.
We also compared the architecture with the MobileNet V3 Large, but the accuracy gain was not worth a bigger model size (20Mo instead of 6Mo) and slower inference speed.

We had to manually modify the last layer of the decoder. Initially designed to classify 1000 different categories of real-world objects, we switched it to an 11-categories layer.

### Training

We use a standard cross entropy loss to train the model and monitor the training process using per-category accuracy score.


### Inference
We provide a script that loads the trained model and monitors the main screen.
For each sampled frame, the script will center-crop and normalize the image then feed it to the AI model.

Following the result of the skill check recognition, the script will automatically press the space bar to trigger the great skill check (or not), 
then it waits for a short period of time to avoid triggering the same skill check multiple times in a row.

To achieve real time results, we convert the model to ONNX format and use the ONNX runtime to perform inference. 
We observed a 1.5x to 2x speedup compared to baseline inference.

### Results

We test our model using a testing dataset of ~2000 images:

| Category Index | Category description        | Mean accuracy |
|----------------|-----------------------------|---------------|
| 0              | None                        | 100.0%        |
| 1              | repair-heal (great)         | 99.5%         |
| 2              | repair-heal (ante-frontier) | 96.5%         |
| 3              | repair-heal (out)           | 98.7%         |
| 4              | full white (great)          | 100%          |
| 5              | full white (out)            | 100%          |
| 6              | full black (great)          | 100%          |
| 7              | full black (out)            | 98.9%         |
| 8              | wiggle (great)              | 93.4%         |
| 9              | wiggle (frontier)           | 100%          |
| 10             | wiggle (out)                | 98.3%         |


During our laptop testing, we observed rapid inference times of approximately 10ms per frame using MobileNet V3 Small. 
When combined with our screen monitoring script, we achieved a consistent 60fps detection rate, which is enough for real-time detection capabilities.

In conclusion, our model achieves high accuracy thanks to the high-quality dataset with effective data augmentation techniques, and architectural choices.
**The RUN script successfully hits the great skill checks with high confidence.**


## Acknowledgments

A big thanks to [hemlock12](https://github.com/hemlock12) for the data collection help !

The project was made and is maintained by me ([Manuteaa](https://github.com/Manuteaa)). If you like it, and want to tell it you can simply star this project !
You want to say a thank-you ? Discuss new project(s) collaboration ? Contact me on discord: manuteaa

An issue ? Questions ? Suggestions ? Just open a new issue
