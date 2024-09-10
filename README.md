# MLC-LLM for Orange Pi 5 Pro

Sources:
- https://llm.mlc.ai/docs
- https://llm.mlc.ai/docs/install/gpu.html#orange-pi-5-rk3588-based-sbc
- https://blog.mlc.ai/2024/04/20/GPU-Accelerated-LLM-on-Orange-Pi
- https://github.com/Chrisz236/llm-rk3588

Please open an issue if you find something that can be improved in this guide

## Prepare the hardware
While this guide is tailored to the Orange Pi 5 Pro it could also be used as a generic guide for other boards using rockchip. Make sure your hardware is properly cooled with a fan and heatsinks as the hardware may overheat or thermal throttle.
## Install Ubuntu for Rockchip
- Download the latest version of ubuntu for your device [here](https://joshua-riek.github.io/ubuntu-rockchip-download/)
- Burn the OS image to a micro SD card using balenaetcher or any other image writing software
- Boot the OS on your device. Optionally connect via SSH for convivence. The default username and password is ubuntu/ubuntu. Once you login for the first time you will be forced to change the password

## Install OS to SSD or eMMC (Optional but recommended)
In order to not experience disk read/write speed bottlenecks when using mlc-llm I highly recommend using a M.2 SSD or eMMC storage module.
- Once booted to the OS off the micro SD card run 

```bash
sudo fdisk -l
```
   to find the storage you will be installing the OS on (Highlighted in red is the eMMC module I will be installing my OS on)
   <img width="539" alt="Pasted image 20240910125904" src="https://github.com/user-attachments/assets/c1fbc06f-944f-465b-bade-50967160d4f0">

- Once you found the storage to install the OS on run the following command
```bash
  sudo ubuntu-rockchip-install /dev/mmcblk0
```
Where "/dev/mmcblk0" is the storage you wish to install the OS on as shown by fdisk. After the process is done you will need to power off the device, remove the micro SD card and turn the system back on. 

## Install OpenCL GPU Drivers
- Download and install `libmali-g610.so`
```bash
  cd /usr/lib && sudo wget https://github.com/JeffyCN/mirrors/raw/libmali/lib/aarch64-linux-gnu/libmali-valhall-g610-g6p0-x11-wayland-gbm.so
```
- Check if file `mali_csffw.bin` exist under path `/lib/firmware`, if not download it with command:
```bash
  cd /lib/firmware && sudo wget https://github.com/JeffyCN/mirrors/raw/libmali/firmware/g610/mali_csffw.bin
```
- Download OpenCL ICD loader and manually add libmali to ICD
```bash
  sudo apt update
sudo apt install mesa-opencl-icd
sudo mkdir -p /etc/OpenCL/vendors
echo "/usr/lib/libmali-valhall-g610-g6p0-x11-wayland-gbm.so" | sudo tee /etc/OpenCL/vendors/mali.icd
```
- Download and install `libOpenCL`
```bash
  sudo apt install ocl-icd-opencl-dev
```
- Download and install dependencies for Mali OpenCL
```bash
  sudo apt install libxcb-dri2-0 libxcb-dri3-0 libwayland-client0 libwayland-server0 libx11-xcb1
```
- Download and install clinfo to check if OpenCL successfully installed
```bash
  sudo apt install clinfo
```
- Run clinfo and check the results. We are looking for a mali device like shown below.![Pasted image 20240910132233](https://github.com/user-attachments/assets/e9913f73-8d7b-4186-83e2-859bfd2e15f6)


## Install Conda - miniconda (optional but recommended)
While not necessary using conda will make managing dependencies for your build environment.
- Grab the latest installer script for your hardware [here](https://docs.anaconda.com/miniconda/) for the Orange Pi 5 Pro I will be using "Miniconda3 Linux-aarch64 64-bit" as the device is using an arm64 CPU.
- Run the following, change the installer script url accordingly
```bash
mkdir -p ~/miniconda3
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-aarch64.sh -O ~/miniconda3/miniconda.sh
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
rm ~/miniconda3/miniconda.sh
```
- add/initialize conda in your shell
 for bash:
```bash
~/miniconda3/bin/conda init bash
```
for zsh:
```zsh
~/miniconda3/bin/conda init zsh
```

- close and restart your shell to activate and start using conda. You should see (base) next to your username ![Pasted image 20240910134503](https://github.com/user-attachments/assets/fe7c8885-55a2-417c-9318-69942e9a9fce)

- Update conda
```bash
conda update --yes -n base -c defaults conda
```
- **Set libmamba as the dependency solver.** The default dependency solver in conda could be slow in certain scenarios, and it is always recommended to upgrade it to libmamba, a faster solver. Install:
```bash
conda install --yes -n base conda-libmamba-solver
```
set as default dependency solver:
```bash
conda config --set solver libmamba
```

## Compile TVM-Unity (Optional but recommended)
See instructions here for up to date instructions: https://llm.mlc.ai/docs/install/tvm.html

mlc-llm provides pre-compiles tvm packages which should be fine for our uses but in my personal testing I was unable to get them to work (specifically on the orange pi 5 pro). If you skip this section and encounter errors related to the "tvm" module I recommend attempting to compile it yourself.

### Some prerequisites
Since these packages are required when building mlc-llm I install them with apt install but you could optionally install them to your build environment with conda. Either way make sure these packages are install prior to attempting to compile TVM-Unity
- Doxygen
- tqdm
- Graphviz
- build-essential (ubuntu specific package. Install the equivalent for your OS)
- zlib1g-dev
- libfl-dev
- clang
#### PIP prerequisites
The following python modules are required in order for us to run mlc-llm and test our tvm-unity build. If you are not using conda you will need to install and use python3-pip
- numpy
- decorator
- psutil
- typing_extensions
- packaging
- attrs
### Using conda
The following commands are for setting up a build environment for compiling TVM-Unity. If you skipped the conda installation you can attempt to install these dependencies using apt install. If manually installing these packages with apt install make sure you are installing the correct versions as by default it will install the latest version of each package.

Run the following to create and setup the conda environment
```bash
# make sure to start with a fresh environment
conda env remove -n tvm-build-venv
# create the conda environment with build dependency
conda create -n tvm-build-venv -c conda-forge \
    "llvmdev>=15" \
    "cmake>=3.24" \
    git \
    python=3.11
# enter the build environment
conda activate tvm-build-venv
```

- Clone the github repo for tvm-unity (actually mlc-ai/relax temporarily, check link above to see if tvm-unity is recommended now) I recommend cloning in the home directory (~)
```bash
# clone from GitHub
git clone --recursive https://github.com/mlc-ai/relax.git tvm-unity && cd tvm-unity
# create the build directory
rm -rf build && mkdir build && cd build
# specify build requirements in `config.cmake`
cp ../cmake/config.cmake .
```

- configure the cmake.config file which instructs cmake how we want to compile the project. Ensure you are in the build directory created earlier when running these commands.
```bash
echo "set(CMAKE_BUILD_TYPE RelWithDebInfo)" >> config.cmake
echo "set(USE_LLVM \"llvm-config --ignore-libllvm --link-static\")" >> config.cmake
echo "set(HIDE_PRIVATE_SYMBOLS ON)" >> config.cmake
```
- Set how mlc-llm will run the llm. The options are Cuda (Nvidia), Metal (Apple), Vulkan (AMD) and OpenCL which is what we will be using. If you are using something other than OpenCL make sure you set that to "ON"
```bash
echo "set(USE_OPENCL ON)" >> config.cmake
```
- once the config.cmake file is created and configured to our liking we can run the following to build the project. Again ensure you are in the build folder from earlier:
```bash
cmake .. && cmake --build . --parallel $(nproc)
```

### Verify TVM compiled correct
You may see several warnings or errors during compilation but this should be ok as long as the build doesn't end with several error messages like the screenshot below which is due to missing libfl-dev ![Pasted image 20240910152125](https://github.com/user-attachments/assets/6eac5357-e851-4396-ae45-d554505053b3)

if you encounter any errors such as the error shown above remediate the errors by installing the required packages and restart the process including and following this command in the guide above. Make sure you are in the tvm-unity directory when running the command below.
```bash
rm -rf build && mkdir build && cd build
```

Example of output when built successfully: ![Pasted image 20240910170713](https://github.com/user-attachments/assets/c51bd9fc-f5b1-425e-9495-1088dc34b98c)


Add the tvm-unity path to your python path variable. Edit the path if your tvm-unity folder is located somewhere else:
```bash
export PYTHONPATH=/home/ubuntu/tvm-unity/python:$PYTHONPATH
```
After adding the path to your python path variable run the following. Make sure you have the python modules installed from the pip prerequisites section!
```bash
python -c "import tvm; print(tvm.__file__)"
```
which should output something similar to: ![Pasted image 20240910173413](https://github.com/user-attachments/assets/ed4880f6-1e75-4f95-bc7f-244c4a4e838d)


Now run the following to check the build options used: 
```bash
python -c "import tvm; print('\n'.join(f'{k}: {v}' for k, v in tvm.support.libinfo().items()))"
```
We should then be able to see opencl set to "ON"

<img width="304" alt="Pasted image 20240910173823" src="https://github.com/user-attachments/assets/e9097e30-664b-4a76-b80e-e13ce060a6f9">



## Compile mlc-llm from source

### Package prerequisites
There is only one other package required here compared to the tvm-unity section. If you installed the packages included in the tvm-unity section then you only need to install git-lfs
- git-lfs
- Doxygen
- tqdm
- Graphviz
- build-essential (ubuntu specific package. Install the equivalent for your OS)
- zlib1g-dev
- libfl-dev
- clang
#### PIP prerequisites
The following python modules are required in order for us to run mlc-llm. If you are not using conda you will need to install and use python3-pip. 
- numpy
- decorator
- psutil
- typing_extensions
- packaging
- attrs
- Pydantic
- shortuuid
- fastapi
- requests
- tqdm
- prompt_toolkit
- uvicorn

### Conda environment (optional but recommended)
If you have conda installed from earlier perform the following to setup a build environment for mlc-llm. Note that if you aren't using conda you need to have the packages installed on your OS with the recommended versions.
- Create the environment 
```bash
conda create -n mlc-chat-venv -c conda-forge \
    "cmake>=3.24" \
    rust \
    git \
    python=3.11
```
- activate the environment
```bash
conda activate mlc-chat-venv
```

### Clone and compile
- clone the mlc-llm github repo. I recommend cloning it to your home directory to make things easy:
```bash
git clone --recursive https://github.com/mlc-ai/mlc-llm.git && cd mlc-llm/
```
- create the build directory
```bash
mkdir -p build && cd build
```
- generate the config.cmake with the following script. Note if you are not using conda you will likely have to replace "python" with "python3"
```bash
python ../cmake/gen_cmake_config.py
```
- for the first option of the script, the tvm location, if you followed the compile tvm-unity instructions from earlier you should use that directory here. Otherwise you can attempt to use the default provided and just leave this option blank. Here in the example below I am using a custom compiled tvm ![Pasted image 20240910175631](https://github.com/user-attachments/assets/91853fcf-bf9c-4b5d-8688-3bae6e57a229)

- Use CUDA should be no
- Use CUTLASS should be no
- Use CUBLAS should be no
- Use ROCm should be no
- Use Vulkan should be no
- Use Metal should be no
- Use OpenCL should be Yes
- Use OpenCLHostPtr can be yes or no. From what I understand this has something to do with using storage for opencl cache of some kind but dont have a great understanding of what exactly this means/does.
- Use FlashInfer should be no
Here is my output after running the script:
![Pasted image 20240910180223](https://github.com/user-attachments/assets/bb2e8efb-5ca3-4dfb-a824-ce8ae9c987b3)


Now you are ready to compile. Run the following to build the project. Again keep an eye out for errors and warnings . Warnings and errors may not prevent the project from compiling 
```bash
cmake .. && cmake --build . --parallel $(nproc) && cd ..
```
Here is my output after a successful build:
![Pasted image 20240910181146](https://github.com/user-attachments/assets/18289722-ec4c-4bb3-967a-d75ccb0c60b8)


After successfully building lets add the mlc-llm path to our pythonpath variable. Change the following to match your paths if you used something different:
```bash
export MLC_LLM_SOURCE_DIR=/home/ubuntu/mlc-llm
export PYTHONPATH=$MLC_LLM_SOURCE_DIR/python:$PYTHONPATH
```
And since mlc-llm is ran within python we will make an alias for calling it:
```bash
alias mlc_llm="python -m mlc_llm"
```

now provided you have all the pre-requisites installed you can run the following to test mlc-llm:
```bash
python -c "import mlc_llm; print(mlc_llm)"
```
if successful you should see something along the lines of:
```
<module 'mlc_llm' from '/home/ubuntu/mlc-llm/python/mlc_llm/__init__.py'>
```

## Running mlc-llm
To open the command help information run the following:
```bash
mlc_llm chat -h
```

### Selecting a device
When attempting to run mlc-llm using opencl on the orange pi 5 pro I found that it would default to a generic ARM opencl device rather than the Mali GPU which we setup drivers for earlier in this guide. Because of this I had to specify the device to run mlc-llm on otherwise I would encounter an error preventing mlc-llm from running. I believe the order of the devices is reflected by clinfo but this may just be a coincidence in my case. See examples of specifying the device in the command examples below
### Getting pre-compiled LLMs
To download and utilize some pre-comipled LLM models for mlc-llm we can visit the mlc-ai organization on huggingface https://huggingface.co/mlc-ai
Available quantization codes are: `q3f16_0`, `q4f16_1`, `q4f16_2`, `q4f32_0`, `q0f32`, and `q0f16`

for testing I will be using SmolLM-1.7B-Instruct-q4f16_1-MLC as its a pretty small download and I've found it runs decent. To run it as a chat run the following (note that in the future you may need to select a different model or an updated version of this model):
```bash
mlc_llm chat HF://mlc-ai/SmolLM-1.7B-Instruct-q4f16_1-MLC --device opencl:1
```

If you wanted to run the model as a server you would do the following:
```bash
mlc_llm serve HF://mlc-ai/SmolLM-1.7B-Instruct-q4f16_1-MLC --device opencl:1 --mode server --host 0.0.0.0
```
This will run mlc-llm as a REST server on opencl device 1 with server mode specified and the host changed from 127.0.0.1 to allow access outside of localhost

You can test if the server is accessible outside of localhost using curl or invoke-restmethod on windows![Pasted image 20240910185054](https://github.com/user-attachments/assets/7a1fda4c-aff6-4e72-bc64-d1c4e927a4b8)



## Extras
### Adding variables to your shell configuration
To make the variables and alias we added earlier permanent we can add them to our bash shell config. Note this is unique to the user and not systemwide:
Modify your shell config (use your text editor of choice):
```bash
sudo nano ~/.bashrc
```
At the bottom of the file add the following and change accordingly based on the paths you used and if you compiled tvm-unity:
```bash
export PYTHONPATH=/home/ubuntu/mlc-llm/python:/home/ubuntu/tvm-unity/python
alias mlc_llm="python -m mlc_llm"
```
Once saved you need to reload your shell in order for the variables to be loaded

### Compiling your own models
See the bottom of the following guide for basic instructions on building your own model:
https://github.com/Chrisz236/llm-rk3588

### Using mlc-llm with HomeAssistant
At the time of writing this HomeAssistant has been adding lots of support for LLMs to utilize as a "Conversation agent" within a custom voice assistant. This allows mlc-llm to act as the brains for whatever you are requesting via your HomeAssistant voice assistant. Currently there is no official integration which allows using mlc-llm with HomeAssistant but there is a custom integration which we can use thanks to mlc-llm Open-AI REST compatibility https://github.com/jekalmin/extended_openai_conversation

The Extended Open-AI conversation integration is a fork of the official Open-AI integration but allows for specifying a custom server address etc. Here is an example of the config options I used when using mlc-llm with HomeAssistant:
![Pasted image 20240910190953](https://github.com/user-attachments/assets/89a91b00-ff81-41bf-89d0-10a7a56127ff)

Also make sure you have the correct chat model selected. I entered the model as shown by the mlc-llm server when querying its models.
![Pasted image 20240910191005](https://github.com/user-attachments/assets/aaddc08d-0305-4539-8405-8bc0c1f3f439)


Do not expect this to perform well. The config options of this extension are vague and have no documentation so I am not sure if there is anything in HomeAssistant that can be tweaked for better performance. I expect we will need an official implementation of Open-AI compatible servers before we see better performance. This integration also attempts to use functions rather than tools for controlling entities in HomeAssistant, I haven't had much luck using either and I expect more work is needed to fine tune this.
