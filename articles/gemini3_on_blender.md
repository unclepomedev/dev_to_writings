---
title: Controlling Blender with Gemini 3 
published: false
description: Bringing the power of Gemini 3 to Blender.
tags: [ python, blender, ai ]
series: My Blender Tools
---

# Controlling Blender with Gemini 3

Gemini 3 was recently announced. According to the [benchmarks](https://blog.google/products/gemini/gemini-3/), it belongs to the top tier in coding capabilities. I tried using it with the Blender API, and it felt not just competitive, but potentially better than other models. Here is a quick introduction.

## Showcase

It generates objects like these from simple prompts:

![Gemini 3 on Blender 1](https://raw.githubusercontent.com/unclepomedev/dev_to_writings/main/images/gemini3_on_blender/1.png)
![Gemini 3 on Blender 2](https://raw.githubusercontent.com/unclepomedev/dev_to_writings/main/images/gemini3_on_blender/2.png)
![Gemini 3 on Blender 3](https://raw.githubusercontent.com/unclepomedev/dev_to_writings/main/images/gemini3_on_blender/3.png)
![Gemini 3 on Blender 4](https://raw.githubusercontent.com/unclepomedev/dev_to_writings/main/images/gemini3_on_blender/4.png)
![Gemini 3 on Blender 5](https://raw.githubusercontent.com/unclepomedev/dev_to_writings/main/images/gemini3_on_blender/5.png)

It can even generate rigs and animations.

![Gemini 3 on Blender 6](https://raw.githubusercontent.com/unclepomedev/dev_to_writings/main/images/gemini3_on_blender/6.gif)

Since it seemed quite powerful, I built a Streamlit app to interactively input prompts.

https://github.com/unclepomedev/BlenderGeminiAgent

## Conclusion

There are no specific benchmarks for Blender coding, so this is just my personal impression, but Gemini 3 feels superior to other models for this task. If this intuition is correct, I wonder if it is due to the influence of YouTube training data or its multimodal capabilities.
