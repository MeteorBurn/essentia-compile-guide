<div align="center">
  <img src="https://essentia.upf.edu/images/essentia_logo_gradient.png" alt="Essentia Logo" width="150"/>
  <h1>Essentia Compile Guide for Ubuntu</h1>
  <p>
    A comprehensive, step-by-step guide to compiling Essentia with TensorFlow GPU support from source on a clean Ubuntu 22.04 server.
  </p>
  <p>
    <i>This guide is based on the <a href="https://github.com/wo80/essentia">wo80/essentia</a> fork. This version is actively maintained by another developer and includes several useful improvements, most notably a modern CMake build system.</i>
  </p>
</div>

> [!WARNING]
> ### A Note on the Process (Disclaimer)
> This guide is the product of a significant amount of trial and error. The process of building Essentia with mandatory TensorFlow GPU support is notoriously difficult.
>
> The single greatest challenge was resolving dependency conflicts. **TensorFlow, in particular, requires a very specific set of package versions to function correctly**, and these often clash with the requirements of other libraries. The versions specified in this guide were carefully chosen to create a stable, compatible environment that worked at the time of writing.
>
> Think of this guide as a stable, verified snapshot in time. While it provides a proven path, it is not a guarantee of a flawless installation on every system.

---

## 📋 Prerequisites

Before you begin, ensure you have the following:

-   ✅ A clean server running **Ubuntu 22.04**.
-   ✅ `sudo` or `root` access.
-   ✅ Basic knowledge of the Linux command line.

---

## 📦 What You'll Need for TensorFlow

To integrate with TensorFlow, Essentia requires two components. It is **critical** that their versions match exactly. This guide uses version `2.14.1`.

1.  **The TensorFlow C API**: A C++ library for model execution.
2.  **The TensorFlow Python Package**: The `.whl` file for your Python environment.

You have several options to obtain these files:

<details>
<summary><strong>Option 1: Use Pre-built Binaries (Recommended)</strong></summary>

This is the simplest and fastest approach.

-   **A) Use the files from this guide:** The `wget` commands in Step 4 will download known-working, pre-built GPU-enabled versions of TensorFlow. This is the most reliable path.
-   **B) Find other pre-built binaries:** If the version in this guide is not compatible with your GPU/CUDA setup, you can find other pre-built versions on the official [TensorFlow website](https://www.tensorflow.com/install/lang_c). Make sure you download both the C API and the Python `.whl` file of the *same version*.

</details>

<details>
<summary><strong>Option 2: Compile TensorFlow from Source (Advanced)</strong></summary>

For maximum performance or compatibility with very specific hardware, you can compile TensorFlow from its source code. This allows you to tailor the build to your exact GPU architecture (e.g., setting `TF_CUDA_COMPUTE_CAPABILITIES`).

This process is significantly more complex and is **outside the scope of this guide**. If you choose this path, you will need to follow the official [TensorFlow build from source instructions](https://www.tensorflow.org/install/source). After building, you will have your own `libtensorflow.tar.gz` (C API) and `tensorflow-....whl` (Python) files to use in Step 4.

</details>

---

## 🚀 Step-by-Step Installation

### ⚙️ Step 1: Install System Dependencies

*First, we'll update the package list and install all the necessary tools and libraries for compilation.*

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

### 🎬 Step 2: Compile FFmpeg from Source

*Essentia uses FFmpeg for audio decoding. We will compile it from source to ensure all required codecs are enabled.*

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

### 🐍 Step 3: Set Up a Python Virtual Environment

*Using a virtual environment is crucial to isolate our Python dependencies and avoid conflicts with system packages.*

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
```
> [!NOTE]
> After activation, you should see `(essentia_env)` at the beginning of your command prompt.

### 🧠 Step 4: Install TensorFlow & Python Dependencies

*Now, with the virtual environment active, we'll install the TensorFlow C API, the TensorFlow Python package, and other libraries Essentia needs.*

```bash
# Navigate back to our main build directory
cd $BUILD_DIR

# 1. Download and install the TensorFlow C API (for C++ code)
wget https://storage.googleapis.com/tensorflow/libtensorflow/libtensorflow-gpu-linux-x86_64-2.14.1.tar.gz
sudo tar -C /usr/local -xzf libtensorflow-gpu-linux-x86_64-2.14.1.tar.gz
sudo ldconfig

# 2. Install Python dependencies
# First, download the matching TensorFlow wheel
wget https://storage.googleapis.com/tensorflow/libtensorflow/tensorflow-2.14.1-cp310-cp310-linux_x86_64.whl

# Install the local wheel file and other required packages
pip install ./tensorflow-2.14.1-cp310-cp310-linux_x86_64.whl
pip install "numpy>=1.23.5,<1.24" "pyyaml>=5.4,<7.0" "six>=1.15,<2.0" "av>=10.0,<11.0"
```

### ✨ Step 5: Build and Install Essentia

*This is the final step: compiling Essentia itself.*

```bash
# Navigate back to our main build directory
cd $BUILD_DIR

# Clone the Essentia repository (using the cmake-ready branch)
git clone --depth 1 --branch cmake https://github.com/wo80/essentia.git essentia_src
cd essentia_src

# Initialize and fetch the required submodules
git submodule update --init --recursive

# Create a build directory
mkdir build && cd build

# Configure the build with CMake
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
sudo ldconfig

# Finally, install the Essentia Python wheel that was just built
cd wheel
pip install --force-reinstall --no-deps essentia-*.whl
```

---

## ✅ Final Verification

*Let's make sure everything was installed correctly. Ensure your virtual environment is still active.*

1.  **Check Essentia**
    ```bash
    python -c "import essentia; import essentia.standard; print(f'✅ Essentia version: {essentia.__version__}')"
    ```
2.  **Check TensorFlow**
    ```bash
    python -c "import tensorflow as tf; print(f'✅ TensorFlow version: {tf.__version__}')"
    ```
3.  **Check Essentia's TensorFlow Integration**
    ```bash
    python -c "from essentia.standard import TensorflowPredictEffnetDiscogs; print('✅ Success: Essentia TensorFlow bindings are working!')"
    ```

<br>

<div align="center">
  <h3>🎉 Congratulations! Your installation is complete. 🎉</h3>
  <p>To leave the virtual environment, simply type: <code>deactivate</code></p>
</div>
