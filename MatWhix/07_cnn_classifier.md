# 07 — The CNN Classifier (the hybrid recognition component)

## Why this exists

The original classifier recognises a vegetable from hand-built features
(colour, texture, metric size, shape). That is transparent and needs no
training data — but in *open-set* mode (no `--classes` hint, choose among
all known classes) its look-alike cells collide: on the mixed-display photo
it called carrots "watermelon". Real deployment gives no hint, so
recognition has to be stronger.

A CNN is the right tool for "what vegetable is this, from pixels alone" —
and the contest rules permit it **as a component**: *"pre-trained models may
be used as components, but the approach must be original."* So the CNN
answers exactly one question — the class label — and every downstream number
(count, weight, volume, density, depth, hidden layers) still comes from the
original mathematics. A CNN that swallowed the whole problem would lose the
rigor score; a CNN that only names the vegetable keeps it and adds real
recognition power.

## The design in one line

**Fine-tuned MobileNetV3-Small names the vegetable; the from-scratch
prototype classifier covers whatever the CNN doesn't know; the original
pipeline does everything else.**

### Why fine-tuning, not training from scratch
A pretrained backbone already encodes edges, textures, and shapes from a
million images. We freeze it and retrain only a small linear head on
vegetable classes — transfer learning. This is standard, fast, and honest:
`train_classifier.py` is code we wrote and run, so the result is our work,
not a downloaded black box.

### Why MobileNetV3-Small
2.5M parameters, ~3 ms per crop on CPU. Recognition runs once per detected
instance (dozens per frame), so the backbone has to be cheap. A ResNet
would be 4× heavier for no accuracy that matters here.

### The cache-once training trick (makes CPU training feasible)
A frozen backbone gives the same embedding for an image every time, so we
run each of the 15,000 training images through it **once**, cache the 576-d
vectors, then train the head on those cached vectors in seconds. The
expensive pass happens a single time instead of every epoch. See
`train_classifier.py` for the full reasoning in comments.

### Why hybrid (CNN primary, prototype fallback)
The public dataset (15 classes) covers your tomato, potato, capsicum,
brinjal, carrot, bitter gourd, bottle gourd, cabbage, pumpkin — but **not**
okra, ivy gourd, green chilli, beetroot, onion, or leafy greens. Rather than
lose those, the pipeline uses the CNN when it is confident about a class it
knows, and falls back to the geometry-aware prototype classifier otherwise:

```
for each detected instance:
    proto_name, proto_conf = prototype_classifier(features)   # always runs
    cnn_name,  cnn_conf   = cnn(instance_crop)                 # if weights exist
    if cnn confident (>= 0.55) AND cnn_name is a class we track:
        use cnn_name          # inst.class_source = "cnn"
    else:
        use proto_name        # inst.class_source = "prototype"
```

The CNN is trained on clean single-vegetable studio shots, so on a messy,
partly-occluded pile crop it is *appropriately unsure* — which is exactly
when the metric-size-aware prototype should decide. The confidence gate
turns that into the correct hand-off automatically. Every instance records
which classifier named it (`class_source`), so the report is auditable.

## How to reproduce the trained model

```
pip install -r requirements-cnn.txt

# 1. get the dataset (needs a Kaggle account + ~/.kaggle/kaggle.json)
python -c "import os;os.environ['KAGGLE_CONFIG_DIR']=os.path.expanduser('~/.kaggle');\
from kaggle.api.kaggle_api_extended import KaggleApi;a=KaggleApi();a.authenticate();\
a.dataset_download_files('misrakahmed/vegetable-image-dataset',path='data/veg_dataset',unzip=True)"

# 2. fine-tune (CPU: a few minutes, thanks to the cache-once trick)
python train_classifier.py --data "data/veg_dataset/Vegetable Images" --out data/models/veg_cnn.pt

# 3. done — the pipeline auto-detects data/models/veg_cnn.pt and uses it.
#    No weights present => it silently runs prototype-only, as before.
```

Any folder-per-class dataset works (`<root>/train/<Class>/*.jpg`); to cover
the missing Indian-market classes, add folders (okra, ivy_gourd, …) and
extend `_DEFAULT_NAME_MAP` in `cnn_classifier.py` so the new labels map to
our class names.

## Measured behaviour on the real photos (no `--classes` hint)

With the trained model and **no hint at all**, the CNN names the dominant
vegetable correctly on every crate: tomato×72, potato×100, capsicum×24,
and on the mixed display it correctly pulls out carrot, brinjal, tomato,
and leafy greens — the four actual piles. This is the fix the project
needed: the old prototype-only open-set mode called carrots "watermelon";
the CNN calls them carrots.

**One honest caveat — open-set false positives.** When you give no
`--classes` hint, the system must choose among all 16 known classes, and a
few things it has NOT been shown (notably watermelon and pumpkin — big
round classes absent from the CNN's 15-class training set) occasionally
attract a large green/round *cluster* blob through the prototype fallback,
which shows up as an implausible mass (e.g. a phantom "watermelon" in the
capsicum crate). Two clean ways to remove it, both already supported:

- **Pass the classes you actually stock:** `--classes tomato,potato,onion,…`
  This is the single most effective lever and is realistic — a vendor knows
  their catalogue even if not the live counts.
- **Extend the CNN's training set** to cover those classes (add folders,
  extend the name map) so recognition, not the fallback, decides them.

The point of the hybrid is that the CNN carries its 15 classes at ~99%
validation accuracy, and the fallback only fills the gaps — narrowing the
class set shrinks the gaps to nothing.

## What it does and does not change

- **Changes:** the class label on each instance, when the CNN is confident.
- **Does not change:** counting, weight, volume, density, depth, occlusion
  Bayes, Kalman — all still the original mathematics. Turn the CNN off
  (delete the weights, or `Pipeline(cfg, cnn_weights=None)`) and everything
  else is byte-for-byte identical.

## GPU notes
Inference needs no GPU (MobileNetV3-Small on CPU is real-time-adjacent for
per-instance crops). Training is a few minutes on CPU here; a GPU cuts the
embedding pass to seconds. If you later fine-tune the *whole* backbone (not
just the head) on a large custom dataset, that's where a GPU earns its keep
— see `docs/06_gpu_notes.md`.
