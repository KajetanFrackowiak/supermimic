# SuperMimic

[Project page](https://supermimic.github.io) | [Paper](https://arxiv.org/abs/2610.28480)

SuperMimic trains a humanoid whole-body motion tracker with one differentiable world model, built on [mjlab](https://github.com/mujocolab/mjlab).

## Setup

Install [uv](https://docs.astral.sh/uv/):

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Clone both repos as siblings:

```bash
mkdir supermimic_ws && cd supermimic_ws
git clone git@github.com:HybridRobotics/supermimic.git
git clone git@github.com:HybridRobotics/mjlab_research.git
cd supermimic
```

## Usage

Available task IDs:

- `SuperMimic-GenCoord-G1` — generalized coordinates representation
- `SuperMimic-MaxCoord-G1` — maximal coordinates representation

Train a world model and policy:

```bash
CUDA_VISIBLE_DEVICES=<GPU_ID> uv run src/supermimic/scripts/train.py \
  <TASK_ID> --runner.run_name <description>
```

Play back a trained model:

```bash
uv run src/supermimic/scripts/play.py \
  <TASK_ID> --wandb-path <RUN_PATH>
```

## Prepare motion data

Reference motions are not versioned. `prepare_motions.py` downloads the
retargeted G1 LAFAN1 CSVs from the Hugging Face Hub and converts them, with
mjlab's own `mjlab.scripts.csv_to_npz` converter, to the `motion.npz` archives
the tracking task loads (30 Hz input, 50 Hz output):

```bash
uv sync --extra data
uv run src/supermimic/scripts/prepare_motions.py
```

This writes one archive per clip plus `manifest.json` under
`data/processed/motions/`, next to the raw CSVs in `data/raw/lafan1/`. Useful
flags: `--only walk1_subject1 run1_subject2`, `--skip-download`,
`--force-download`, `--overwrite`, `--input-fps`, `--output-fps`.

Optionally publish each clip as a W&B registry artifact, so training can refer
to it by name instead of by path:

```bash
uv run src/supermimic/scripts/upload_motions.py --all
```

The uploader prints the ready-to-paste `--registry_name` value. Without a
registry, point training or playback at a local archive:

```bash
uv run src/supermimic/scripts/train.py <TASK_ID> \
  --motion_file data/processed/motions/walk1_subject1.npz
```

`play.py` accepts the same `--motion_file` override, and `--motion_file` also
takes a glob or comma-separated list to train on several clips at once.

`train.py` and `train_ppo.py` default to **all 40 clips at once**, which is the
paper's setup: one policy over the whole library, with the reset distribution
spanning the concatenated timeline (441k frames, ~2.45 h). Pass a shorter list
for a quick smoke test:

```bash
uv run src/supermimic/scripts/train.py <TASK_ID> \
  --runner.run_name smoke --registry_name kajetanfrackowiak/wandb-registry-lafan1-mjlab/walk1_subject1:latest
```

The W&B destination defaults to `kajetanfrackowiak/wandb-registry-lafan1-mjlab`
(both in the uploader and in the training scripts) and can be changed with
`--entity`, `--project`, `--registry` and `--registry-org`.

## Cloud training

The `Dockerfile` in the parent directory (`supermimic_ws/`, next to this clone)
builds a GPU image with the training stack. The build context is the parent, not
this repository, because the project installs mjlab from the sibling checkout:

```bash
cd ..                                   # supermimic_ws
docker build -t thekajtek/supermimic:v1 .
docker run --gpus all -it thekajtek/supermimic:v1
```

Inside the container, training reads motions straight from the W&B registry:

```bash
uv run python src/supermimic/scripts/train.py SuperMimic-GenCoord-G1 \
  --runner.run_name <description>
```

The image needs an NVIDIA driver new enough for the CUDA 13 torch build
(driver 580+); check with `nvidia-smi` on the instance. Pass credentials at run
time rather than baking them in, either as `-e WANDB_API_KEY=...` or by mounting
`~/.netrc`.

## Acknowledgements

SuperMimic contains code inspired by these projects:

- [ControlVAE](https://github.com/heyuanYao-pku/Control-VAE) (BSD-3-Clause)
- [robotic_world_model](https://github.com/leggedrobotics/robotic_world_model) (Apache-2.0)
- [DreamerV3](https://github.com/danijar/dreamerv3) (MIT)

## License

Copyright 2026 The SuperMimic Authors.

Released under the [Apache License 2.0](LICENSE).
