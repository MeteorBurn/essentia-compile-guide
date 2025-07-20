<div align="center">
  <h1>TensorFlow 2.16.2 GPU Build Guide</h1>
  <p>
    Complete step-by-step guide for compiling TensorFlow 2.16.2 with CUDA GPU support from source on WSL Ubuntu 22.04
  </p>
  <p>
    <i>Creates optimized .whl packages and C API .tar.gz archives with support for specific GPU architectures</i>
  </p>
</div>

> [!WARNING]
> ### Important Notes
> - Build process takes 5-8 hours and requires significant resources
> - Minimum 16GB RAM and 50GB free disk space required
> - Creates libraries optimized for your specific GPU architecture
> - CUDA, cuDNN, and TensorRT versions must exactly match TensorFlow 2.16.2 requirements

---

## 📋 System Requirements

- **OS:** WSL 2 Ubuntu 22.04
- **RAM:** Minimum 16GB (32GB recommended)
- **Storage:** 50GB+ free space
- **GPU:** NVIDIA with CUDA support (compute capability >= 3.5)
- **Access:** sudo privileges required

---

## 🚀 Step 1: Install CUDA Toolkit 12.3

*TensorFlow 2.16.2 officially supports CUDA 12.3 for optimal performance.*

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install basic dependencies
sudo apt install -y build-essential wget

# Create working directory
export TF_BUILD_DIR=~/tensorflow_build
mkdir -p $TF_BUILD_DIR
cd $TF_BUILD_DIR

# Download and setup CUDA 12.3 repository for WSL
wget https://developer.download.nvidia.com/compute/cuda/repos/wsl-ubuntu/x86_64/cuda-wsl-ubuntu.pin
sudo mv cuda-wsl-ubuntu.pin /etc/apt/preferences.d/cuda-repository-pin-600

# Download CUDA 12.3 local repository
wget https://developer.download.nvidia.com/compute/cuda/12.3.0/local_installers/cuda-repo-wsl-ubuntu-12-3-local_12.3.0-1_amd64.deb

# Install repository
sudo dpkg -i cuda-repo-wsl-ubuntu-12-3-local_12.3.0-1_amd64.deb
sudo cp /var/cuda-repo-wsl-ubuntu-12-3-local/cuda-*-keyring.gpg /usr/share/keyrings/

# Install CUDA Toolkit 12.3
sudo apt-get update
sudo apt-get -y install cuda-toolkit-12-3

# Setup environment variables
echo 'export CUDA_HOME=/usr/local/cuda-12.3' >> ~/.bashrc
echo 'export PATH=$CUDA_HOME/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

## 🧠 Step 2: Install cuDNN 8.9

*cuDNN 8.9 is compatible with CUDA 12.3 and required for TensorFlow 2.16.2.*

```bash
cd $TF_BUILD_DIR

# Download cuDNN 8.9 for CUDA 12.x
wget https://developer.download.nvidia.com/compute/cudnn/redist/cudnn/linux-x86_64/cudnn-linux-x86_64-8.9.7.29_cuda12-archive.tar.xz

# Extract archive
tar -xf cudnn-linux-x86_64-8.9.7.29_cuda12-archive.tar.xz

# Copy files to CUDA directory
sudo cp -r cudnn-linux-x86_64-8.9.7.29_cuda12-archive/include/* /usr/local/cuda-12.3/include/
sudo cp -r cudnn-linux-x86_64-8.9.7.29_cuda12-archive/lib/* /usr/local/cuda-12.3/lib64/

# Set permissions
sudo chmod a+r /usr/local/cuda-12.3/include/cudnn*.h /usr/local/cuda-12.3/lib64/libcudnn*
```

## 🔗 Step 3: Update Linker Cache

*Update system library cache for proper CUDA and cuDNN functionality.*

```bash
# Update linker cache
sudo ldconfig

# Reload environment variables
source ~/.bashrc
```

## ✅ Step 4: Verify CUDA and cuDNN Installation

*Verify that all components are installed correctly.*

```bash
# Check CUDA version
nvcc --version

# Check GPU and driver
nvidia-smi

# Verify cuDNN installation
ls /usr/local/cuda-12.3/include/cudnn*.h

# Get your GPU's compute capability (needed for configuration)
nvidia-smi --query-gpu=name,compute_cap --format=csv
```

> [!NOTE]
> Remember your compute capability value - you'll need it during TensorFlow configuration.

## 🐍 Step 5: Create Build Environment

*Create isolated Python environments for building TensorFlow.*

### For Python 3.10:
```bash
cd $TF_BUILD_DIR

# Install Python 3.10 and dependencies
sudo apt install -y python3.10 python3.10-dev python3.10-venv python3-pip

# Create virtual environment
python3.10 -m venv tf_build_env_310
source tf_build_env_310/bin/activate

# Upgrade base packages
pip install --upgrade pip setuptools wheel
```

### For Python 3.11:
```bash
cd $TF_BUILD_DIR

# Install Python 3.11 and dependencies
sudo apt install -y python3.11 python3.11-dev python3.11-venv

# Create virtual environment
python3.11 -m venv tf_build_env_311
source tf_build_env_311/bin/activate

# Upgrade base packages
pip install --upgrade pip setuptools wheel
```

## 📦 Step 6: Download TensorFlow Source Code

*Clone the official TensorFlow repository version 2.16.2.*

```bash
cd $TF_BUILD_DIR

# Clone TensorFlow repository
git clone https://github.com/tensorflow/tensorflow.git
cd tensorflow

# Switch to version 2.16.2
git checkout v2.16.2

# Initialize submodules (this will take time)
git submodule update --init --recursive
```

## 🔧 Step 7: Install Build Dependencies

*Install all required Python packages and system dependencies.*

### System Dependencies:
```bash
# Install system packages for building
sudo apt install -y \
    build-essential \
    curl \
    git \
    pkg-config \
    zip \
    unzip \
    wget \
    libtool \
    autoconf \
    automake
```

### Python Dependencies:
```bash
# Ensure virtual environment is active
# source tf_build_env_310/bin/activate  # for Python 3.10
# or
# source tf_build_env_311/bin/activate  # for Python 3.11

# Install TensorFlow 2.16.2 build dependencies
pip install \
    numpy==1.26.4 \
    packaging \
    requests \
    opt_einsum \
    astunparse \
    gast==0.4.0 \
    google_pasta \
    h5py \
    libclang \
    ml_dtypes \
    protobuf \
    setuptools \
    six \
    termcolor \
    typing_extensions \
    wrapt \
    tensorflow_estimator \
    keras \
    tensorboard
```

### Install Bazel:
```bash
cd $TF_BUILD_DIR

# Download and install Bazel 6.1.0 (required for TensorFlow 2.16.2)
wget https://github.com/bazelbuild/bazel/releases/download/6.1.0/bazel-6.1.0-installer-linux-x86_64.sh
chmod +x bazel-6.1.0-installer-linux-x86_64.sh
./bazel-6.1.0-installer-linux-x86_64.sh --user


# Add Bazel to PATH
echo 'export PATH="$PATH:$HOME/bin"' >> ~/.bashrc
source ~/.bashrc

# Verify installation
bazel --version
```

## ⚡ Step 8: Install TensorRT (Optional)

*TensorRT accelerates neural network inference on NVIDIA GPUs.*

```bash
cd $TF_BUILD_DIR

# Download TensorRT 8.6 for CUDA 12.x
wget https://developer.nvidia.com/downloads/compute/machine-learning/tensorrt/secure/8.6.1/tars/tensorrt-8.6.1.6.linux.x86_64-gnu.cuda-12.0.tar.gz

# Extract
tar -xzf tensorrt-8.6.1.6.linux.x86_64-gnu.cuda-12.0.tar.gz

# Copy to CUDA directory
sudo cp -r TensorRT-8.6.1.6/include/* /usr/local/cuda-12.3/include/
sudo cp -r TensorRT-8.6.1.6/lib/* /usr/local/cuda-12.3/lib64/

# Set permissions
sudo chmod a+r /usr/local/cuda-12.3/include/NvInfer*.h /usr/local/cuda-12.3/lib64/libnvinfer*

# Update linker cache
sudo ldconfig

# Verify installation
ls /usr/local/cuda-12.3/include/NvInfer*.h
```

## ⚙️ Step 9: Configure TensorFlow

*Configure TensorFlow build parameters.*

```bash
cd $TF_BUILD_DIR/tensorflow

# Activate the desired virtual environment
source ../tf_build_env_310/bin/activate  # for Python 3.10
# or
# source ../tf_build_env_311/bin/activate  # for Python 3.11

# Set environment variable for correct Python version
export TF_PYTHON_VERSION=3.10  # or 3.11 respectively

# Run configuration
python configure.py
```

### Configuration Answers:

| Question | Recommended Answer | Explanation |
|----------|-------------------|-------------|
| Python library path | `[Enter]` | Use default path |
| ROCm support | `N` | For NVIDIA GPUs |
| CUDA support | `y` | Required for GPU |
| TensorRT support | `y` | If TensorRT is installed |
| CUDA SDK version | `12.3` | Installed CUDA version |
| cuDNN version | `8.9` | Installed cuDNN version |
| TensorRT version | `8.6` | If TensorRT is installed |
| NCCL version | `[Enter]` | Use GitHub version |
| CUDA library paths | `[Enter]` | Use default paths |
| Compute capabilities | `8.6` | Your value from step 4 |
| Use clang as CUDA compiler | `N` | Use nvcc |
| GCC host compiler | `[Enter]` | Use system GCC |
| MPI support | `N` | Usually not needed |
| Optimization flags | `-march=native -Wno-sign-compare` | Optimize for your CPU |
| Android builds | `N` | Not required |

## 🏗️ Step 10: Build TensorFlow

*Compile TensorFlow with the specified parameters.*

### Build Python Package:
```bash
cd $TF_BUILD_DIR/tensorflow

# Activate corresponding environment
source ../tf_build_env_310/bin/activate  # for Python 3.10
export TF_PYTHON_VERSION=3.10

# Build for Python 3.10 (takes 5-8 hours)
bazel build \
    --config=opt \
    --config=cuda \
    --jobs=4 \
    --local_ram_resources=8000 \
    //tensorflow/tools/pip_package:build_pip_package
```

For Python 3.11:
```bash
# Deactivate current environment and activate 3.11
deactivate
source ../tf_build_env_311/bin/activate
export TF_PYTHON_VERSION=3.11

# Build for Python 3.11
bazel build \
    --config=opt \
    --config=cuda \
    --jobs=4 \
    --local_ram_resources=8000 \
    //tensorflow/tools/pip_package:build_pip_package
```

> [!TIP]
> **Build Parameters:**
> - `--jobs=4` - use 4 parallel processes
> - `--local_ram_resources=8000` - limit RAM usage to 8GB
> - For systems with more RAM, you can increase these values

## 📦 Step 11: Package into .whl

*Create .whl packages for installation.*

### For Python 3.10:
```bash
cd $TF_BUILD_DIR/tensorflow

# Activate Python 3.10 environment
source ../tf_build_env_310/bin/activate
export TF_PYTHON_VERSION=3.10

# Create .whl file
./bazel-bin/tensorflow/tools/pip_package/build_pip_package ../tf_wheel_310

# Check result
ls -la ../tf_wheel_310/tensorflow-*.whl
```

### For Python 3.11:
```bash
# Activate Python 3.11 environment
deactivate
source ../tf_build_env_311/bin/activate
export TF_PYTHON_VERSION=3.11

# Create .whl file
./bazel-bin/tensorflow/tools/pip_package/build_pip_package ../tf_wheel_311

# Check result
ls -la ../tf_wheel_311/tensorflow-*.whl
```

## 🔗 Step 12: Build C API and Package into .tar.gz

*Create C API library for use in other programming languages.*

```bash
cd $TF_BUILD_DIR/tensorflow

# Сборка архива с libtensorflow.so, libtensorflow_cc.so, заголовками и .zip
bazel build \
  --config=opt \
  --config=cuda \
  //tensorflow/tools/lib_package:libtensorflow_pkg

# Copy archive
cp bazel-bin/tensorflow/tools/lib_package/libtensorflow_pkg.tar.gz ../libtensorflow-2.16.2-gpu-linux-x86_64.tar.gz

# Verify
ls -la ../libtensorflow-2.16.2-gpu-linux-x86_64.tar.gz

## ✅ Step 13: Installation and Testing

*Test the built components.*

### Install Python Package:
```bash
cd $TF_BUILD_DIR

# Activate desired environment
source tf_build_env_310/bin/activate  # for Python 3.10
# or
# source tf_build_env_311/bin/activate  # for Python 3.11

# Install built TensorFlow
pip install tf_wheel_310/tensorflow-*.whl --force-reinstall  # for Python 3.10
# or
# pip install tf_wheel_311/tensorflow-*.whl --force-reinstall  # for Python 3.11
```

### Test GPU Functionality:
```bash
# IMPORTANT: Exit TensorFlow source directory
cd ~

# Test TensorFlow and GPU
python -c "
import tensorflow as tf
print(f'TensorFlow version: {tf.__version__}')
print(f'CUDA available: {tf.test.is_built_with_cuda()}')
gpu_devices = tf.config.list_physical_devices('GPU')
print(f'GPU devices: {gpu_devices}')

# Check TensorRT (if installed)
try:
    from tensorflow.python.compiler.tensorrt import trt_convert
    print('TensorRT available: True')
except ImportError:
    print('TensorRT available: False')
"

# Test GPU computation
python -c "
import tensorflow as tf
print('Testing GPU computation...')
with tf.device('/GPU:0'):
    a = tf.random.normal([1000, 1000])
    b = tf.random.normal([1000, 1000])
    c = tf.matmul(a, b)
print('✅ GPU computation test passed!')
"
```

### Test C API:
```bash
cd $TF_BUILD_DIR

# Extract C API for testing
tar -xzf libtensorflow-2.16.2-gpu-linux-x86_64.tar.gz

# Check contents
ls -la libtensorflow-2.16.2-gpu/lib/
ls -la libtensorflow-2.16.2-gpu/include/tensorflow/c/

# Verify library contains symbols
nm -D libtensorflow-2.16.2-gpu/lib/libtensorflow.so | grep TF_NewSession
```

## 🎯 Build Results

After successful completion, you will have:

### Python Packages:
- `tensorflow-2.16.2-cp310-cp310-linux_x86_64.whl` (~450MB)
- `tensorflow-2.16.2-cp311-cp311-linux_x86_64.whl` (~450MB)

### C API Library:
- `libtensorflow-2.16.2-gpu-linux-x86_64.tar.gz` (~250MB)

### Build Features:
- ✅ Optimized for specific GPU architecture
- ✅ CUDA 12.3 + cuDNN 8.9 support
- ✅ Optional TensorRT 8.6 support
- ✅ `-march=native` CPU optimization
- ✅ Compatible with Essentia and other ML frameworks

## 🚨 Troubleshooting

### Memory Errors:
```bash
# Reduce parallel jobs
--jobs=4 --local_ram_resources=8000
```

### CUDA Errors:
```bash
# Check environment variables
echo $CUDA_HOME
echo $LD_LIBRARY_PATH
nvcc --version
```

### .whl Packaging Errors:
```bash
# Ensure TF_PYTHON_VERSION is set correctly
export TF_PYTHON_VERSION=3.10  # or 3.11
```

### Clean for Rebuild:
```bash
# Partial clean (preserves downloaded dependencies)
bazel clean

# Full clean (removes everything including downloaded dependencies)
bazel clean --expunge
```

## 📚 Additional Resources

- [TensorFlow Build from Source](https://www.tensorflow.org/install/source)
- [CUDA Installation Guide](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/)
- [cuDNN Installation Guide](https://docs.nvidia.com/deeplearning/cudnn/install-guide/)
- [TensorRT Installation Guide](https://docs.nvidia.com/deeplearning/tensorrt/install-guide/)

## 🎓 Performance Tips

### Build Optimization:
- Use `-march=native` for CPU optimization
- Set compute capability to match your GPU exactly
- Use maximum available RAM and CPU cores

### Runtime Optimization:
- Set `TF_FORCE_GPU_ALLOW_GROWTH=true` for dynamic memory allocation
- Use mixed precision training with `tf.keras.mixed_precision`
- Enable XLA compilation with `TF_XLA_FLAGS=--tf_xla_enable_xla_devices`

---

<div align="center">
  <h3>🎉 Congratulations! TensorFlow 2.16.2 with GPU support built successfully! 🎉</h3>
  <p>Your optimized components are ready for machine learning projects</p>
</div>