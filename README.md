<div align="center">
  <h1>Essentia Compile Guide for Ubuntu</h1>
  <p>
    A comprehensive, step-by-step guide to compiling Essentia with TensorFlow GPU support from source on a clean Ubuntu 22.04 server.
  </p>
  <p>
    <i>This guide is based on the <a href="https://github.com/wo80/essentia">wo80/essentia</a> fork. This version is actively maintained by one of its original developers and includes several useful improvements, most notably a modern CMake build system. At the time of writing, the <code>cmake</code> branch was the most current for this purpose, which is why it is used in the instructions below.</i>
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

## ⚠️ GPU Support Setup (Pre-installation)

> [!IMPORTANT]
> This guide assumes your server is already configured with the necessary NVIDIA drivers and libraries for GPU support. The TensorFlow version we are installing requires a specific environment.
>
> You must install these components **before** starting Step 1.

You are responsible for installing:
1.  **NVIDIA Driver:** A recent version compatible with your GPU.
2.  **CUDA Toolkit:** Version **11.8**.
3.  **cuDNN Library:** Version **8.6**.

You can find the required cuDNN archive here: [cuDNN 8.6 for CUDA 11.x](https://developer.download.nvidia.com/compute/cudnn/redist/cudnn/linux-x86_64/cudnn-linux-x86_64-8.6.0.163_cuda11-archive.tar.xz).

This guide does not provide instructions for installing these NVIDIA components. Please refer to the official NVIDIA documentation for your specific hardware and OS.

---

## 📦 What You'll Need for TensorFlow

To integrate with TensorFlow, Essentia requires two components. It is **critical** that their versions match exactly. This guide uses version `2.14.1` for Python `3.10`.

1.  **The TensorFlow C API**: A C++ library for model execution.
2.  **The TensorFlow Python Package**: The `.whl` file for your Python environment.

This guide's default method is to download these files directly using `wget` in Step 4. However, advanced users have other options:

<details>
<summary><strong>Alternative Options (Manual Download or Build from Source)</strong></summary>

-   **Manual Download:** You can download the necessary files beforehand from the official TensorFlow sources. If you do this, place the downloaded files into the `~/essentia_build` directory and skip the `wget` commands in Step 4, proceeding directly to the `tar` and `pip install` commands.

-   **Build from Source (Advanced):** For maximum performance or compatibility with specific hardware, you can compile TensorFlow yourself. This process is complex and is outside the scope of this guide. You will need to follow the official [TensorFlow build from source instructions](https://www.tensorflow.org/install/source). After building, use your custom `libtensorflow.tar.gz` and `tensorflow-....whl` files in Step 4.

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

*Now, with the virtual environment active, we'll install the C API via `wget` and then manually download and install the Python package.*

**Part A: Install the C API**

```bash
# Navigate back to our main build directory
cd $BUILD_DIR

# Download and install the TensorFlow C API
wget https://storage.googleapis.com/tensorflow/libtensorflow/libtensorflow-gpu-linux-x86_64-2.14.1.tar.gz
sudo tar -C /usr/local -xzf libtensorflow-gpu-linux-x86_64-2.14.1.tar.gz
sudo ldconfig
```

**Part B: Manually Download and Install the Python Package**

Direct `wget` links for Python packages from official sources can be unreliable. The safest method is to download the `.whl` file manually from your browser.

1.  **Open the PyPI page:** In your local web browser, go to the official download page:
    [**TensorFlow 2.14.1 Files on PyPI**](https://pypi.org/project/tensorflow/2.14.1/#files).

2.  **Find the correct file:** Carefully look for the file that matches our environment:
    -   It must contain `cp310` (for Python 3.10).
    -   It must contain `manylinux` and `x86_64`.
    -   The full name is `tensorflow-2.14.1-cp310-cp310-manylinux_2_17_x86_64.manylinux2014_x86_64.whl`.

3.  **Download the file** to your local computer.

4.  **Upload the file** from your computer to your server's build directory (`~/essentia_build`). You can use `scp`, `sftp`, or any other file transfer tool.

Once the `.whl` file is in your `~/essentia_build` directory on the server, run the following commands:

```bash
# Navigate to the build directory if you aren't there already
cd $BUILD_DIR

# Install the .whl file you just uploaded
# Note: The wildcard * makes it easier so you don't have to type the full long name.
pip install ./tensorflow-2.14.1-cp310-*.whl

# Finally, install the other Python packages for Essentia
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

---

## 🎓 Citing Essentia

If you use this software in your research, please don't forget to cite the original authors of Essentia:

> Bogdanov, D., Wack N., Gómez E., Gulati S., Herrera P., Mayor O., et al. (2013). **ESSENTIA: an Audio Analysis Library for Music Information Retrieval.** In *Proceedings of the 14th International Society for Music Information Retrieval Conference (ISMIR'13)* (pp. 493-498).
