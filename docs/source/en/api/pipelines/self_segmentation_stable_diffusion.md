<!--Copyright 2023 The HuggingFace Team. All rights reserved.

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with
the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on
an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the
specific language governing permissions and limitations under the License.
-->


# Self-Segmentation for Stable Diffusion

This pipeline outputs a segmentation map that corresponds to the generated image. Each segment is labeled according to
the prompt that generates the image. The segmentation method is introduced in this [paper](https://arxiv.org/abs/2303.11306) by
Or Patashnik, Daniel Garibi, Idan Azuri, Hadar Averbuch-Elor, Daniel Cohen-Or. The segmentation does not rely on any dedicated segmentation network, but rather on the rich prior of the image
generation network itself. Performing self-segmentation is simple and fast, and adds only a nigilble overhead to the
image generation.

In a nutshell, we save the self-attention maps, and cluster them at the end of the image generation. Each cluster
corresponds to a semantic segment in the image. To label each segment, we utilize the cross-attention maps of the
diffusion model. For each segment, we find the token in the prompt whose associated cross-attention map has the highest
intersection with the segment. The token is then used as the label for the segment.
We overcome the fact that most clustering algorithms require the number of clusters as input by merging segments
according to their labels.

The abstract of the paper is the following:

*Text-to-image models give rise to workflows which often begin with an exploration step, where users sift through a large collection of generated images. The global nature of the text-to-image generation process prevents users from narrowing their exploration to a particular object in the image. In this paper, we present a technique to generate a collection of images that depicts variations in the shape of a specific object, enabling an object-level shape exploration process. Creating plausible variations is challenging as it requires control over the shape of the generated object while respecting its semantics. A particular challenge when generating object variations is accurately localizing the manipulation applied over the object's shape. We introduce a prompt-mixing technique that switches between prompts along the denoising process to attain a variety of shape choices. To localize the image-space operation, we present two techniques that use the self-attention layers in conjunction with the cross-attention layers. Moreover, we show that these localization techniques are general and effective beyond the scope of generating object variations. Extensive results and comparisons demonstrate the effectiveness of our method in generating object variations, and the competence of our localization techniques.*

## FAQ

### How well it performs compared to dedicated segmentation networks?

The self-segmentation technique leverages the generative prior of large diffusion models,
rather than an external segmentation model. It is true that large segmentation models (e.g., SAM, ODISE) trained with a
lot of supervision, are strong enough, but also come with heavy computational costs. Our self-segmentation utilizes the
rich features that are used to synthesize the given image. This allows for dealing with challenging scenarios, like the
fish with a transparent fin or a non-realistic image such as the sketch of the cup.
Note that while the resolution of the segmentation map is rather low, it is easy to induce them to a higher resolution.

![img](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/diffusers/self_seg_comparison.png)

### What would be a good usecase to use this segmentation technique?

As shown in our [paper](https://huggingface.co/papers/2303.11306), this technique is useful for image editing. It allows to
localize edis by taking segments that should not be edited from the original image. Since we get this segmentation
"for free" when generating an image, we believe that localizing an edit is the best use-case for this technique, rather
than using it as a heavy segmentation model.

## Available Pipelines:

| Pipeline | Tasks 
|---|---
| [SelfSegmentationStableDiffusionPipeline](https://github.com/huggingface/diffusers/blob/main/src/diffusers/pipelines/self_segmentation_stable_diffusion_pipeline.py) | *Text-to-Image Generation, Segmentation* | - |


## Usage

The usage of this pipeline is very similar to the vanilla Stable Diffusion pipeline. We describe the differences below.

### Inputs

- `segmentation_resolution` - This is the resolution of the self-attention maps used for the clustering. The default is
32, since we found it to be a good trade-off between the segmentation resolution and the semantics encoded in the
self-attention maps.
- `segmentation_clustering_algorithm` - The clustering algorithm used to cluster the self-attention maps. You can choose
between `kmeans` and `spectral`. The default is `spectral`.
- `num_segments` - The number of clusters to use for the clustering algorithm. The default is 6. Since the method merges
clusters, it is recommended to use a higher number of clusters than the actual number of desired segments.
- `seg_labels_indices_in_prompt` (optional) - The indices of the tokens in the prompt that correspond to the labels of the segments.
To label the segments, we consider the cross-attention maps in these indices and for each segment, we find the token
whose cross-attention map has the highest intersection with the segment. This argument accepts a list of `int` as input.
If not provided, we automatically extract indices of nouns from the prompt and use them.

### Outputs

In addition to the outputted generated image, this pipeline outputs the following:
- `seg_map` - The segmentation map of the generated image before merging clusters.
- `seg_labels` - A dictionary that maps each segment to its label.
- `seg_map_merged` - The segmentation map of the generated image after merging clusters.
- `seg_labels_merged` - A dictionary that maps each segment to its label after merging clusters.

All the outputs are given in a list, each element corresponds to the corresponding image in the output images list.

### Example

A simple execution of the pipeline that used the default inputs:

```python
from diffusers import SelfSegmentationStableDiffusionPipeline
import torch

pipe = SelfSegmentationStableDiffusionPipeline.from_pretrained(
    "runwayml/stable-diffusion-v1-5", torch_dtype=torch.float16
)
pipe = pipe.to("cuda")
prompt = "A cat with sunglasses"
out = pipe(prompt)  # uses the defaults for the segmentation parameters

image = out.images[0]
seg_map = out.seg_map[0]
labels = out.seg_labels[0]
seg_map_merged = out.merged_seg_map[0]
merged_labels = out.merged_seg_labels[0]
```

The segmentation maps can be visualized with the following code snippet:

```python
import matplotlib.pyplot as plt
import numpy as np
import matplotlib.patches as mpatches
from skimage import color

fig, ax = plt.subplots()
rgb_seg_map_merged = color.label2rgb(seg_map_merged)
rgb_seg_map_merged = (rgb_seg_map_merged * 255).astype(np.uint8)
ax.imshow(rgb_seg_map_merged)

label2rgb = {}
for i in range(seg_map_merged.shape[0]):
    for j in range(seg_map_merged.shape[1]):
        label = seg_map_merged[i, j]
        label2rgb[label] = rgb_seg_map_merged[i, j] / 255
splitted_prompt = prompt.split(" ")

merged_labels_names = {}
for l in merged_labels:
    if merged_labels[l] != "background":
        merged_labels_names[l] = splitted_prompt[merged_labels[l]]
    else:
        merged_labels_names[l] = "background"

legend_elements = []
for i in merged_labels:
    patch = mpatches.Patch(color=label2rgb[i], label=merged_labels_names[i])
    legend_elements.append(patch)
ax.legend(handles=legend_elements, loc="lower center", bbox_to_anchor=(0.5, -0.16), ncol=3)
plt.axis("off")
plt.show()
```

### Tips

*Want to play with the segmentation parameters after the image has already generated?* No problem! The attention maps
are saved in the pipeline object even after the generation. You can use the following code snippet to re-segment the
last generated image.

```python
# Define parameters
segmentation_resolution = 32
segmentation_clustering_algorithm = "spectral"
num_segments = 6
seg_labels_indices_in_prompt = None

# Re-segment the last generated image
# (here, we're reusing the same `pipe` initialized above.)
agg_self_attn = pipe.attn_processor.get_aggregated_self_attn(segmentation_resolution)
agg_cross_attn = pipe.attn_processor.get_aggregated_cross_attn(16)
segmentation_map = pipe.self_attn_to_seg(
    agg_self_attn, segmentation_clustering_algorithm, segmentation_resolution, num_segments
)

if seg_labels_indices_in_prompt is None:
    seg_labels_indices_in_prompt = pipe.get_nouns_indices([prompt])
seg_labels = pipe.label_seg(segmentation_map, agg_cross_attn, seg_labels_indices_in_prompt)
merged_seg_map, new_labels = pipe.merge_seg_map(segmentation_map, seg_labels)
```

## SelfSegmentationStableDiffusionPipeline
[[autodoc]] SelfSegmentationStableDiffusionPipeline
    - all
    - __call__