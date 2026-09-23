## OpScribe-AI's Official Submission for SurgVU 2026 Category 2 Surgical VQA Challenge

Team Members:
- Noah John Kalthoff
- Ruffin Hager Bryant
- Aadya Ganjigunta
- Tuo Peter Li
- Dhananjay Bhaskar

Our model achieved BERTScore-F1 score of **0.9128** in Category 2 during the [preliminary phase](https://surgvu26.grand-challenge.org/evaluation/category-2-final-phase/leaderboard/) of the MICCAI EndoVis SurgVU Surgical VQA Challenge.
The preliminary set is the organisers' public 11-case sample, which was also used to
calibrate the lookup-table wording and the fixed cutoffs (for example the motion cutoff);
the CNNs, detector, needle-driver head and VLM were trained with those 11 cases held out.

In this challenge, a 30-second surgical clip and a text question are provided; the model must produce a free-text answer. Submissions are scored
using BERTScore-F1, taking the best answer graded across five independent human reference answers.


## Reproducing and running the container

The submission is two things rather than one.

| artifact | contains |
|---|---|
| the container image | the code, both ResNet-50 recognisers, the detector, the variant head, and the VLM's LoRA adapter |
| a model tarball | the NF4-quantised Qwen2.5-VL-7B base, ~5 GB |

We split them because Grand Challenge caps the image at 10 GB and the base model on its
own is about 5 GB. Grand Challenge extracts the model tarball to `/opt/ml/model/` when
the container runs, and `resolve_vlm_model_dir()` checks there before it checks anywhere
else. The LoRA inside the image was trained on that exact base, so the two have to be
used together.

### How to get the weights

The weights are not in this repository. All of them are on Hugging Face at
[opscribe-ai/surgvu26-cat2-v6.2](https://huggingface.co/opscribe-ai/surgvu26-cat2-v6.2).
`containers/build_submission.sh` pulls them into the build context, and if yours are
somewhere else you can point `VLM_MODEL_SRC` at a directory that holds
`qwen25vl-7b-nf4/`.

### Building

```bash
# Apptainer (what was used)
containers/build_submission.sh
apptainer build surgvu26-submission.sif containers/surgvu26-submission.def

# Docker (Grand Challenge upload format)
docker build -f containers/Dockerfile -t surgvu26-cat2 .
docker save surgvu26-cat2 | gzip > surgvu26-cat2.tar.gz

# the model tarball. The trailing dot is load-bearing: Grand Challenge uses the
# archive's paths as-is, so packing the parent directory makes every lookup miss.
tar -czf surgvu26-models.tar.gz -C /path/to/models .
```

### How to run a case

The container reads and writes fixed paths, and both JSON files hold a JSON-encoded
string rather than raw text.

| direction | path |
|---|---|
| read | `/input/endoscopic-robotic-surgery-video.mp4` |
| read | `/input/visual-context-question.json` |
| write | `/output/visual-context-response.json` |

```bash
mkdir -p input output model
tar -xzf surgvu26-models.tar.gz -C model/          # gives model/qwen25vl-7b-nf4/

cp your_clip.mp4 input/endoscopic-robotic-surgery-video.mp4
echo '"What instrument is being used?"' > input/visual-context-question.json

docker run --rm --gpus all \
  -v "$(pwd)/input:/input:ro" \
  -v "$(pwd)/output:/output" \
  -v "$(pwd)/model:/opt/ml/model:ro" \
  surgvu26-cat2

cat output/visual-context-response.json            # e.g. "Bipolar Forceps"
```

If you forget the `/opt/ml/model` mount the container will still run, but the VLM won't
work properly and every question ends up being answered by the router instead.

The `--judge` flag in the entrypoint is inert in the submitted configuration:
`config/arbiter.json` ships arbiter mode `per_intent`, which never consults the judge,
and no judge weights are in the model tarball.

### Environment

The base image is `pytorch/pytorch:2.5.1-cuda12.1-cudnn9-runtime`. The VLM layer pins
`transformers==4.57.6`, `accelerate==1.14.0` and `bitsandbytes==0.50.1`. Those pins are
load-bearing, and `containers/surgvu26-submission.def` records why.
`docs/container_build.md` covers the build in full, and `docs/submission_interface.md`
documents the Grand Challenge input/output contract.

The final build ran on a single 14.6 GiB T4 within the 600 second per-case wall clock,
with no network access at inference.

---

## Repository layout

```
src/surgvu/     pipeline: perception, router, arbiter, VLM, frame planning
scripts/        inference entrypoint, training, evaluation, tuning
config/         model config, serving thresholds, arbiter policy, splits
containers/     Dockerfile, Apptainer definition, build scripts
condor/         HTCondor job files, how every run was actually executed
tests/          test suite
docs/           design notes, build guide, compliance audit
```

- **`docs/compliance_audit.md`**: ensures we were working within the rules.
- **`docs/design/`**: the plans and measurements we followed.

---

## Data and licensing

The SurgVU 2026 dataset is not redistributed here. The organizers will likely release
it after the challenge is over.

Code is Apache-2.0 (`LICENSE`). Third-party components and their licences are listed
in `NOTICE`.

## Hugging Face links

- **Base Qwen model** — [Qwen/Qwen2.5-VL-7B-Instruct](https://huggingface.co/Qwen/Qwen2.5-VL-7B-Instruct)
- **NVIDIA then trained that model further on CholecT50** — [nvidia/Qwen2.5-VL-7B-Surg-CholecT50](https://huggingface.co/nvidia/Qwen2.5-VL-7B-Surg-CholecT50)
- **The dataset we then used to train a LoRA on minimally invasive surgery** — [opscribe-ai/mis-abdominal-gi-min-invasive](https://huggingface.co/datasets/opscribe-ai/mis-abdominal-gi-min-invasive)
- **The dataset we fine-tuned that LoRA on to reach the model used in the pipeline** — [opscribe-ai/surgvu-cat2-vqa](https://huggingface.co/datasets/opscribe-ai/surgvu-cat2-vqa)
- **All the artifacts (CNNs, YOLO, the LoRA)** — [opscribe-ai/surgvu26-cat2-v6.2](https://huggingface.co/opscribe-ai/surgvu26-cat2-v6.2)

---

## AI assistance

We used [Claude Code](https://claude.com/claude-code) while building this. It helped
with the pipeline code, the evaluation tooling, the documentation and commits. 
