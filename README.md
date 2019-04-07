# deepracer
A repo for running deepracer locally

# How to run and use this
Firstly, this is not for the faint of heart. I am trying to build this repo so it's extremely easy for people to get this running but there are a lot of moving parts and it can be a nightmare. Onward!

## The moving parts
Deepracer is made of the following parts:
- The simulation which is built out of ROS Kinetic, colcon, Gazebo and a simulation environment provided by amazon
- Sagemaker which is built out of the sagemaker SDK and a docker image that is run by it using docker-compose (#2.), which uses image (#1.)
- S3 for some communication and model storage
- Cloudwatch for logs

To get each of these components working in a local environment isn't too hard until you realise the docker images rely on tensorflow with certain CPU features, thus you have to re-build them on your CPU. If you don't want to use ANY AWS services, it means you have to emulate S3 and bypass cloudwatch hits, which is what I have done.

To emulate S3 you can use minio and my patches to various components, which are provided as submodules in this repo. The patches also bypass cloudwatch with environment variables.

# Minio
This service is used to emulate S3 and is very easy to setup and use. Go to the (minio download website)[https://min.io/download] and download your version. Then run the command `minio server .\data` and copy your IP location, key id and access key, you will use these in your enviornment variables.

# Building Robomaker
I have provided a docker build file name Robomarker.docker that does all the build so you can refer to that. In summary, it's install the dependencies of ROS Kinetic and Gazebo. Then install the dependencies of the Deepracer simulation environment. If you want to run those commands outside of a Docker build, I have marked each command that requires sudo.

Run `docker build -t deepracer_robomaker:1.0 -f 

# Docker images

Following is about building the images used by sagemaker sdk. I will in future provide these in a docker repo somewhere so you don't have to build them.

## Building sagemaker-tensorflow-scriptmode
`cd sagemaker-tensorflow-container/docker/1.11.0`
Docker build command `docker build -t 520713654638.dkr.ecr.us-west-2.amazonaws.com/sagemaker-tensorflow-scriptmode:1.11.0-cpu-py3 --build-arg py_version=3 -f Dockerfile.cpu .`

You used to require a copy of tensorflow to build it, you can get one by doing `pip download --no-deps tensorflow==1.11.0`. I have *removed this need by modifying the Dockerfile*.

To get a copy of the `framework_support_installable` it needs, you just need to build the directory using `python3 setup.py sdist`, then move the tar to the docker directory.

## Building sagemaker-containers
You need to build this for sagemaker-rl-tensorflow image as to include the patches for S3 and cloudwatch.

To build it go to sagemaker-containers `cd sagemaker-containers`, and run `python3 setup.py sdist`. Then copy the tar.gz over to sagemaker-rl-container.

## Building sagemaker-rl-tensorflow
`cd sagemaker-rl-container`
Stay at the top level of sagemaker-rl-container repo when building the docker file. Make to sure to build sagemaker-containers before this.

To build the docker image run `docker build -t 520713654638.dkr.ecr.us-east-1.amazonaws.com/sagemaker-rl-tensorflow:coach0.11-cpu-py3 --build-arg sagemaker_container=sagemaker_containers-2.4.4.post2.tar.gz --build-arg processor=cpu -f .\coach\docker\0.11.0\Dockerfile.tf .`

If you ever need to quickly rebuild the image with sagemaker-containers, you can run `$(pushd ../sagemaker-containers; python setup.py sdist;popd); $(cp ../sagemaker-containers/dist/*.tar.gz ./); $(docker build -t 520713654638.dkr.ecr.us-east-1.amazonaws.com/sagemaker-rl-tensorflow:coach0.11-cpu-py3 --build-arg sagemaker_container=sagemaker_containers-2.4.4.post2.tar.gz --build-arg processor=cpu -f ./coach/docker/0.11.0/Dockerfile.tf .)`.

## Build and install sagemaker sdk
This one is rather easy. Just `cd sagemaker-python-sdk` and run `pip3 install .`, that will install everything it needs for the SDK to run. You will need to have docker and docker-compose in the path of any scripts that invoke the SDK though.

## Building it all example
These commands may work on your system but serve as an example of each step. I am assuming you are in the repo root directory. These were done on a windows machine in powershell
```
$root = $(pwd)
python -m venv venv
./venv/Scripts/Activate.ps1
cd sagemaker-tensorflow-container
python setup.py sdist
cp ./dist/sagemaker_tensorflow_container-2.0.0.tar.gz ./docker/1.11.0/
cd docker/1.11.0/
docker build -t 520713654638.dkr.ecr.us-west-2.amazonaws.com/sagemaker-tensorflow-scriptmode:1.11.0-cpu-py3 --build-arg py_version=3 -f Dockerfile.cpu .
cd $root
cd sagemaker-containers
python setup.py sdist
cp dist/sagemaker_containers-2.4.4.post2.tar.gz ../sagemaker-rl-container
cd $root
cd sagemaker-rl-container
docker build -t 520713654638.dkr.ecr.us-east-1.amazonaws.com/sagemaker-rl-tensorflow:coach0.11-cpu-py3 --build-arg sagemaker_container=sagemaker_containers-2.4.4.post2.tar.gz --build-arg processor=cpu -f ./coach/docker/0.11.0/Dockerfile.tf .
cd $root
cd sagemaker-python-sdk
pip install -U .
pip install ipython
pip install -U colorama==0.4
cd $root
docker build -t deepracer_robomaker -f docker/Robomaker.docker .
docker run --name dr deepracer_robomaker
#This comes from minio output
$env:AWS_ACCESS_KEY_ID="PLEXW8P0SOZALM05XQ1A"
$env:AWS_SECRET_ACCESS_KEY="Io0Z7xJOYxqZs3UwkZ7GdVfk7+8cw90roK6QKE0N"
$env:AWS_DEFAULT_REGION="us-east-1"
$env:LOCAL="True"
$env:S3_ENDPOINT_URL=$(write-host "Enter the ip of minio server in this variable instead of copying this")
ipython .\rl_deepracer_coach_robomaker.py


```

# Running it all
One word of warning, prepare for it to all break. Try not to get fustrated. If you're having issues, feel free to contact me. Make sure you've built it all, including the docker images so they are available in your local docker repo.

## The moving parts in order
- Robomaker
- Sagemaker - Run `cd rl_coach` and then `ipython rl_deepracer_coach_robomaker.py`

### Starting robomaker
You can run the docker image `docker run --name dr 

You need to start Robomaker first as the python script will not do it. Then the script will run sagemaker, and hence the needed docker images.


# image names
- 1. 520713654638.dkr.ecr.us-west-2.amazonaws.com/sagemaker-tensorflow-scriptmode:1.11.0-cpu-py3
- 2. 520713654638.dkr.ecr.us-east-1.amazonaws.com/sagemaker-rl-tensorflow:coach0.11-cpu-py3

The names need to be those as the internals of sagemaker SDK looks for them.
