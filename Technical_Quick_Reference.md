# OpenVINS Implementation Quick Reference

## Repository Links

- **Main OpenVINS**: https://github.com/rpng/open_vins
- **SuperPoint OpenVINS**: https://github.com/robintzeng/open_vins_with_superpoint

## Feature Extraction Comparison

### Traditional Feature Extractors

#### FAST (Features from Accelerated Segment Test)
```cpp
// Configuration example
fast_threshold: 20
fast_nonmaxSuppression: true
grid_x: 5
grid_y: 3
num_features: 200
```

#### ORB (Oriented FAST and Rotated BRIEF)
```cpp
// Configuration example
orb_nfeatures: 500
orb_scaleFactor: 1.2f
orb_nlevels: 8
orb_edgeThreshold: 31
```

### SuperPoint Configuration
```yaml
# SuperPoint settings
feature_tracker:
  type: "SUPERPOINT"
  model_path: "/path/to/superpoint_model.pb"
  detection_threshold: 0.015
  descriptor_dim: 256
  max_features: 200
  nms_radius: 4
  gpu_device: 0  # -1 for CPU
```

## Performance Benchmarks

### Timing Comparison (per frame)
| Method | CPU (ms) | GPU (ms) | Memory (MB) |
|--------|----------|----------|-------------|
| FAST | 1-2 | N/A | 50-100 |
| ORB | 3-5 | N/A | 80-120 |
| SuperPoint | 25-40 | 8-15 | 300-500 |

### Accuracy Metrics (EuRoC Dataset)
| Sequence | Traditional RMSE (m) | SuperPoint RMSE (m) | Improvement |
|----------|---------------------|---------------------|-------------|
| MH_01 | 0.162 | 0.123 | 24% |
| MH_02 | 0.195 | 0.147 | 25% |
| V1_01 | 0.087 | 0.064 | 26% |
| V1_02 | 0.098 | 0.071 | 28% |

## Code Integration Examples

### Traditional Feature Tracking
```cpp
#include "track/TrackKLT.h"
#include "track/Grider_FAST.h"

// Initialize tracker
TrackKLT tracker;
tracker.set_num_features(200);
tracker.set_fast_threshold(20);

// Process frame
std::vector<cv::KeyPoint> keypoints;
cv::Mat descriptors;
tracker.feed_new_camera(timestamp, image, 0);
```

### SuperPoint Integration
```cpp
#include "track/TrackDescriptor.h"
#include "superpoint/SuperPointExtractor.h"

// Initialize SuperPoint
SuperPointExtractor extractor;
extractor.load_model("superpoint_model.pb");
extractor.set_detection_threshold(0.015);

// Process frame
std::vector<cv::KeyPoint> keypoints;
cv::Mat descriptors;
extractor.extract_features(image, keypoints, descriptors);
```

## Build Instructions

### Traditional OpenVINS
```bash
# Dependencies
sudo apt-get install libeigen3-dev libopencv-dev libboost-all-dev

# Build
git clone https://github.com/rpng/open_vins.git
cd open_vins
mkdir build && cd build
cmake ..
make -j4
```

### SuperPoint OpenVINS
```bash
# Additional dependencies
pip install tensorflow-gpu  # or tensorflow-cpu
# OR
pip install torch torchvision

# Build with SuperPoint support
git clone https://github.com/robintzeng/open_vins_with_superpoint.git
cd open_vins_with_superpoint
mkdir build && cd build
cmake .. -DWITH_SUPERPOINT=ON -DWITH_TENSORFLOW=ON
make -j4
```

## Configuration Templates

### Traditional Config (config/euroc_mav.yaml)
```yaml
# Visual tracking
num_pts: 200
fast_threshold: 20
grid_x: 5
grid_y: 3
min_px_dist: 10

# Feature representation
feat_rep_msckf: "ANCHORED_MSCKF_INVERSE_DEPTH"
feat_rep_slam: "ANCHORED_MSCKF_INVERSE_DEPTH"

# Tracker type
use_klt: true
use_descriptor: false
```

### SuperPoint Config (config/euroc_superpoint.yaml)
```yaml
# Visual tracking
num_pts: 200
superpoint_threshold: 0.015
grid_x: 5
grid_y: 3
min_px_dist: 15

# SuperPoint specific
model_path: "/path/to/models/superpoint_v1.pb"
descriptor_dim: 256
nms_radius: 4
use_gpu: true

# Tracker type
use_klt: false
use_descriptor: true
use_superpoint: true
```

## Debugging Tips

### Traditional OpenVINS
```cpp
// Enable debug visualization
tracker.set_histmethod(ov_core::HISTOGRAM::HISTOGRAM);
tracker.enable_debug_output(true);

// Check feature count
if (features.size() < min_features) {
    PRINT_WARNING("Low feature count: %d\n", features.size());
}
```

### SuperPoint OpenVINS
```cpp
// Check GPU availability
if (!extractor.is_gpu_available()) {
    PRINT_WARNING("GPU not available, falling back to CPU\n");
}

// Monitor inference time
auto start = std::chrono::high_resolution_clock::now();
extractor.extract_features(image, keypoints, descriptors);
auto end = std::chrono::high_resolution_clock::now();
auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
PRINT_INFO("SuperPoint inference time: %d ms\n", duration.count());
```

## Common Issues and Solutions

### Traditional OpenVINS
| Issue | Solution |
|-------|----------|
| Low feature count | Reduce FAST threshold, increase grid size |
| Poor tracking | Enable KLT, adjust min_px_dist |
| Drift in low-texture | Switch to ORB descriptors |

### SuperPoint OpenVINS
| Issue | Solution |
|-------|----------|
| Slow performance | Enable GPU, reduce image resolution |
| Model not found | Check model path, download pre-trained weights |
| Memory errors | Reduce batch size, check GPU memory |
| Poor features | Adjust detection threshold, check lighting |

## Hardware Recommendations

### Traditional OpenVINS
- **CPU**: Intel i5 or ARM Cortex-A72 equivalent
- **RAM**: 2GB minimum
- **Storage**: 1GB for build
- **Power**: <5W typical

### SuperPoint OpenVINS
- **CPU**: Intel i7 or equivalent
- **GPU**: NVIDIA GTX 1050 or better (4GB VRAM minimum)
- **RAM**: 8GB minimum
- **Storage**: 5GB (including models)
- **Power**: 15-30W typical

## Model Files

### SuperPoint Pre-trained Models
```bash
# Download TensorFlow model
wget https://github.com/rpautrat/SuperPoint/raw/master/pretrained_models/sp_v6.pb

# Download PyTorch model
wget https://github.com/magicleap/SuperPointPretrainedNetwork/raw/master/superpoint_v1.pth
```

### Model Conversion (if needed)
```python
# Convert PyTorch to TensorFlow
import torch
import tensorflow as tf

# Load PyTorch model
model = torch.load('superpoint_v1.pth')

# Convert to TensorFlow (requires additional tools)
# See SuperPoint documentation for details
```

## Performance Tuning

### Optimization Strategies
1. **Image Resolution**: Reduce input size for speed vs. accuracy trade-off
2. **Feature Count**: Adjust num_features based on application needs
3. **Detection Threshold**: Higher values = fewer but stronger features
4. **Grid Size**: Larger grids = better spatial distribution

### Real-time Optimization
```cpp
// Adaptive feature count based on processing time
if (processing_time > target_time) {
    reduce_feature_count();
} else if (processing_time < target_time * 0.8) {
    increase_feature_count();
}
```

---

*This quick reference is designed for developers implementing or modifying OpenVINS systems. For comprehensive analysis and comparison, see the full documentation.*