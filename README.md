# 3D Print Failure Detector

## What is 3D Print Failure Detector?
3D Print Failure Detector is an automatic print failure detection system for 3D printers and runs on an RPi and another 64-bit computer. It uses Octoprint and Octolapse to get timelapse images, which 3DPS uses to determine if a failure occurred.

## How does it work?
Octolapse generates amazing timelapse images where, from the camera's viewpoint, only the 3D model changes. These images are sent to the 64-bit computer (using lsyncd/rsync), where the arrival of a new image (with inotifywait) triggers the Python scripts. By comparing the previous layer image to the current layer image and through the use of OpenCV, a score in the form of a Normalised Root Mean-Squared Error (NRMSE) value is calculated, which represents how similar the two images are, with values over 1 representing a significant deviation. This value stays consistently below 1 after shadow thresholding during a simple 3D print.

This was extended, so that the current image is compared to the image from 5 layers prior. This is named the "deviance" and is calculated in the same way as the score. This value aims to represent how much the print has deviated from past layers. This value is quite high when the current layer has less of a change than the layer from 5 layers prior. An example would be anything with a large base, the base itself is large, anything on top of the base would be smaller in comparison, meaning the deviance would be high. Once 5 layers into the part on top of the base, the deviance will decrease as there is less of a change.
Using both the score and deviance, it is possible to detect when a 3D print has either detached from the bed or a part has broken off, even when the filament has either ran out or clogged.
* Detachment - Score > 1.0 AND Deviance > 1.0
* Breakage - Score Diff > 0.2 AND Deviance Diff > 0.2
* Filament run-out/clog - Score < 0.2 AND Deviance < 0.2

Another implemented method besides image is The Spaghetti Detective - a spaghetti detection model from https://github.com/TheSpaghettiDetective/TheSpaghettiDetective. This allows using a machine learning model to examine each snapshot took by Octolapse and detects spaghetti failure. The Spaghetti detection model has been integrated into the `printcontrol.py`. If a spaghetti is detected, same procedure to pause the printing will be invoked.

## How to use

### Installation on computer side
**Note:** The Spaghetti Detective only supports 64 bit operating systems!

Clone this repo to your home directory:
```
$ git clone https://github.com/paulokinho/3d-print-failure-detection
```

In terminal, navigate to the root directory of this repo, run the `run` script with `-i` flag for dependency installation:
```
$ ./run -i
```
This may take quite some time as a number of packages are being installed.

You can press ctrl-c to exit once you see "Setup complete!" in your terminal.

To make sure 3DPrintFailureDetector works, you also need to make sure that you have **OpenCV** installed with version **>= 3.4.6**. The commands in `run` to install OpenCV are commented out by default as they might not work. Do check if you have the right version installed. I will not include the steps to install OpenCV here.

### Setting up Octoprint and Octolapse on the RPi
While waiting for the installation to finish, you can start to work on the RPi. Follow the tutorial below to setup Octoprint on your RPi: https://octoprint.org/download/ both model 3 and 4 should be fine.

After you are done, set up Octolapse following this tutorial:
https://github.com/FormerLurker/Octolapse/wiki/Installation

Next, open `api_keys.py` with your preferred text editor, and replace the URL and API_KEY with the ones for your Octoprint.

### Setting up rsync
reference: https://www.digitalocean.com/community/tutorials/how-to-use-rsync-to-sync-local-and-remote-directories-on-a-vps

First, make sure that both your computer and RPi are connected to the same network. Both machines should have rsync installed. Set up SSH access from your computer to the RPi. Check this if you don't know how to do: https://itsfoss.com/ssh-into-raspberry/

The `run` script you ran just now should have created a `snapshots` directory for you. In your `3DPrintFailureDetector` directory, execute the following command:
```
$ rsync -a [YOUR_RPI_USERNAME]@[YOUR_RPI_IP]:/home/[YOUR_RPI_USERNAME]/.octoprint/data/octolapse/snapshots ./snapshots
```
Enter password for your RPi as it might require.

### Using
Before, starting your print, start the run script on your computer:
```
$ ./run
```
On you Octoprint, properly set the Octolapse configuration that you prefer, it is recommended to set nozzle position when taking snapshots to a corner that is out of the view of the camera. Don't forget to turn on octolapse at the end.

Then just start the printing through octoprint. When the snapshots are taken, the images will be synchronized to the snapshots folder on your computer, then the python scripts will be invoked.