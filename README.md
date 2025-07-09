# Manual Installation Guide for Essentia on Ubuntu

This guide provides step-by-step instructions for compiling and installing Essentia and its dependencies (FFmpeg, TensorFlow) from source on a clean Ubuntu 22.04 server. This process ensures you have a custom build tailored to your system.

**Important Note:** This guide uses a fork of the official Essentia repository, available at **https://github.com/wo80/essentia.git**. This fork is actively maintained by another developer and includes several useful improvements, most notably a modern and easier-to-use build system based on `CMake`. At the time of writing this guide, the `cmake` branch was the most current and stable for this purpose, which is why it is used here.

## What You'll Need

1.  **TensorFlow C API:** This is the C++ library that Essentia uses to run TensorFlow models.
2.  **TensorFlow Python Package:** This is the `.whl` file for installing TensorFlow in your Python environment.

The C API and Python package versions must match exactly. This guide uses version `2.14.1`. You have two options:
*   **Use Pre-built Files:** Download the exact files used in this guide for a known-working setup (recommended for simplicity).
*   **Use Your Own Files:** Download a different version from the official [TensorFlow website](https://www.tensorflow.com/install/lang_c) that is compatible with your specific GPU and CUDA version. If you do this, you must update the filenames in the commands below.

## Prerequisites

- A clean server running Ubuntu 22.04.
- `sudo` or root access.
- Basic knowledge of the Linux command line.

---

### Step 1: Install System Dependencies

First, update your package list and install all the necessary tools and libraries for compilation. These include build tools, audio codec libraries, and Python development files.

```bash
# Update package lists
sudo apt-get update -q

# Install essential packages for building software
sudo apt-get install -y -q \
    build-essential yasm nasm pkg-config autoconf automake cmake patchelf git libtool wget \
    libyaml-dev libmp3lame-dev libfdk-aac-dev libfaad-dev libopus-dev libvorbis-dev \
    libflac-dev libwavpack-dev libtwolame-dev libgsm1-dev libsndfile1-dev \
    libsamplerate0-dev libchromaprint-dev libeigen3-dev libfftw3-dev \
    libavcodec-dev libavformat-dev libavutil-dev libswresample-dev libtag1-dev \
    python3.10 python3.10-dev python3.10-venv \
    python3-numpy python3-yaml python3-six
```

---

### Step 2: Compile FFmpeg from Source

Essentia uses FFmpeg for audio decoding. We will compile it from source to ensure all required codecs are enabled.

```bash
# Create a directory for our source code
export BUILD_DIR=~/essentia_build
mkdir -p $BUILD_DIR
cd $BUILD_DIR

# Clone the specific version of FFmpeg
git clone --depth 1 --branch n7.1.1 https://git.ffmpeg.org/ffmpeg.git ffmpeg_src

# Navigate into the source directory
cd ffmpeg_src

# Configure the build with required flags
./configure \
    --prefix="/usr/local" \
    --enable-gpl --enable-nonfree --enable-shared --disable-static --disable-debug \
    --enable-libmp3lame --enable-libfdk-aac --enable-libopus \
    --enable-libvorbis --enable-libtwolame --enable-libgsm \
    --enable-chromaprint

# Compile using all available processor cores
make -j"$(nproc)"

# Install the compiled libraries and binaries
sudo make install

# Update the system's linker cache
sudo ldconfig

# Verify the installation
ffmpeg -version
```

---

### Step 3: Set Up a Python Virtual Environment

Using a virtual environment is crucial to isolate our Python dependencies and avoid conflicts with system packages.

```bash
# Navigate back to our main build directory
cd $BUILD_DIR

# Create a virtual environment using Python 3.10
python3.10 -m venv essentia_env

# Activate the virtual environment
# IMPORTANT: You must do this in every new terminal session to use the installed packages.
source $BUILD_DIR/essentia_env/bin/activate

# Upgrade pip and install the wheel package
pip install --upgrade pip wheel

# You should now see (essentia_env) at the beginning of your command prompt.
```

---

### Step 4: Install TensorFlow C API and Python Dependencies

Essentia's TensorFlow integration requires both the C++ library and the Python package.

**Important:** Ensure your virtual environment is still active before running these commands.

```bash
# Navigate back to our main build directory
cd $BUILD_DIR

# 1. Download and install the TensorFlow C API (for C++ code)
# This version matches the Python package we will install later.
wget https://storage.googleapis.com/tensorflow/libtensorflow/libtensorflow-gpu-linux-x86_64-2.14.1.tar.gz

# Extract the library directly into /usr/local
sudo tar -C /usr/local -xzf libtensorflow-gpu-linux-x86_64-2.14.1.tar.gz

# Update the linker cache again
sudo ldconfig

# 2. Install Python dependencies
# First, download the matching TensorFlow wheel
wget https://storage.googleapis.com/tensorflow/libtensorflow/tensorflow-2.14.1-cp310-cp310-linux_x86_64.whl

# Install the local wheel file
pip install ./tensorflow-2.14.1-cp310-cp310-linux_x86_64.whl

# Install other required Python packages for Essentia
pip install "numpy>=1.23.5,<1.24" "pyyaml>=5.4,<7.0" "six>=1.15,<2.0" "av>=10.0,<11.0"
```

---

### Step 5: Build and Install Essentia

Now we are ready to compile Essentia itself.

**Important:** Ensure your virtual environment is still active.

```bash
# Navigate back to our main build directory
cd $BUILD_DIR

# Clone the Essentia repository (using the cmake-ready branch)
git clone --depth 1 --branch cmake https://github.com/wo80/essentia.git essentia_src

# Navigate into the source directory
cd essentia_src

# Initialize and fetch the required submodules
git submodule update --init --recursive

# Create a build directory
mkdir build && cd build

# Configure the build with CMake. This tells Essentia where to find everything
# and enables Python bindings and TensorFlow support.
cmake .. \
      -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_INSTALL_PREFIX="/usr/local" \
      -DBUILD_SHARED_LIBS=ON \
      -DBUILD_PYTHON_BINDINGS=ON \
      -DUSE_TENSORFLOW=ON \
      -DBUILD_EXAMPLES=ON

# Compile Essentia
make -j"$(nproc)"

# Install the C++ library and Python module components
sudo make install

# Update the linker cache one last time
sudo ldconfig

# Finally, install the Essentia Python wheel that was just built
# This makes it available in your virtual environment
cd wheel
pip install --force-reinstall --no-deps essentia-*.whl
```

---

### Step 6: Verification

Let's verify that everything was installed correctly.

**Important:** Ensure your virtual environment is still active.

```bash
# 1. Check Essentia
# Run a python command to import essentia and print its version
python -c "import essentia; import essentia.standard; print(f'Essentia version: {essentia.__version__}')"

# Expected output should be something like: Essentia version: 2.1-beta...

# 2. Check TensorFlow
# Run a python command to import tensorflow and print its version
python -c "import tensorflow as tf; print(f'TensorFlow version: {tf.__version__}')"

# Expected output: TensorFlow version: 2.14.1

# 3. Check Essentia's TensorFlow integration
python -c "from essentia.standard import TensorflowPredictEffnetDiscogs; print('✅ Success: Essentia TensorFlow bindings are working!')"

# Expected output: ✅ Success: Essentia TensorFlow bindings are working!
```

If all commands run without errors, your installation is complete!

To leave the virtual environment, simply type:
```bash
deactivate
```