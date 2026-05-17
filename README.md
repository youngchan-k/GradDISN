# GradDISN

Gradient-based Deep Implicit Surface Network for detailed single-view 3D reconstruction.

GradDISN builds on [DISN](https://github.com/laughtervv/DISN). The main change in this repository is an optional gradient-weighted SDF sample format: generated GradDISN samples store a fifth `pc_sdf_sample` column containing the local SDF gradient weight, and training feeds that value into the reconstruction loss.

## Repository Layout

- `preprocessing/`: dataset metadata, image H5 conversion, and SDF sample generation.
- `data/`: threaded H5 data loaders used by training and evaluation.
- `models/`: TensorFlow model definitions and custom TensorFlow ops.
- `train/`: SDF reconstruction training entry point.
- `cam_est/`: camera-estimation model and training entry point.
- `test/`: reconstruction, SDF accuracy, IoU, Chamfer/EMD, and F-score utilities.
- `demo/`: a minimal demo image and output mesh example.
- `isosurface/`: external marching-cubes and distance-field binaries used during preprocessing and mesh extraction.
- `assets/`: README figures.

## Requirements

This codebase follows the original DISN TensorFlow 1.x stack and uses `tf.contrib`, so it is not compatible with TensorFlow 2.x without migration work. The preprocessing and evaluation utilities also expect the external `isosurface` binaries, ShapeNet-style data, and the paths configured in `preprocessing/info.json`.

Typical Python dependencies include TensorFlow 1.x, NumPy, h5py, OpenCV, trimesh, PyMesh, SciPy, and joblib. Custom TensorFlow ops live under `models/tf_ops/`; rebuild them only when your TensorFlow/CUDA environment differs from the checked-in `.so` files.

## Data Configuration

Edit `preprocessing/info.json` before running preprocessing or training. The file lists:

- `lst_dir`: category train/test file lists.
- `cats` and `all_cats`: ShapeNet category names and IDs.
- `raw_dirs_v1`: mesh, rendered image, SDF, and marching-cubes output directories.

The checked-in file lists under `data/filelists/` provide category splits, but the raw ShapeNet meshes, rendered images, generated SDFs, and checkpoints are not included.

## SDF Sample Generation

Generate SDF H5 files from the repository root:

```bash
mkdir -p log
source isosurface/LIB_PATH
python -u preprocessing/create_point_sdf_grid.py --model GradDISN --thread_num 9 --category all
```

Use `--category chair` or another category name from `preprocessing/info.json` to process a single class.

The `--model` argument controls the sample format:

- `GradDISN` writes `[x, y, z, sdf, gradient]` and is the default.
- `DISN` writes the original `[x, y, z, sdf]` format. The loader treats missing gradient values as `1.0`.

## Training

Train the SDF model from the repository root after data paths and checkpoints are configured:

```bash
python -u train/train_sdf.py --log_dir checkpoint/SDF_GradDISN --category all --img_feat_twostream
```

Useful options include `--gpu`, `--batch_size`, `--num_sample_points`, `--restore_model`, `--restore_modelcnn`, `--restore_modelpn`, `--cam_est`, and `--binary`.

## Evaluation And Demo

Create meshes for the test split:

```bash
python -u test/create_sdf.py --log_dir checkpoint/SDF_GradDISN --category chair --img_feat_twostream --create_obj
```

Run SDF accuracy evaluation:

```bash
python -u test/test_sdf_acc.py --log_dir checkpoint/SDF_GradDISN --test_lst_dir data/filelists --category chair --img_feat_twostream
```

Run the minimal demo:

```bash
python -u demo/demo.py --log_dir checkpoint/SDF_GradDISN --img_feat_twostream
```

## Method Summary

<img src="./assets/occupancy.PNG"/>

GradDISN emphasizes fine geometric detail by weighting the SDF reconstruction loss with a local gradient estimate. Points near occupancy changes receive larger weights, encouraging the model to preserve delicate structures.

## Results

<img src="./assets/results.PNG"/>

In the included examples, GradDISN reconstructs fine structures such as wings and antennae more clearly than the baseline DISN setup.
