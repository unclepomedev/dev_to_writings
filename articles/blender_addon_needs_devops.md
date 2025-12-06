---
description: Blender Addon Development Needs More DevOps.
published: false
series: Blender Addon Development
tags:
  - python
  - blender
  - devops
title: Blender Addon Development Needs More DevOps
---

# TL;DR

- Write test code and automate it.
- Run tests on various versions of Blender to release with peace of mind.

# Intended Audience

- Individual developers who are (or want to be) developing Blender addons.
    - People who want to ensure quality but don't know how to introduce CI/CD.
    - People tired of manual fixes and checks.
- People interested in CI/CD using GitHub Actions (since this article simply applies a general CI/CD pipeline to Blender
  addon development).

# Introduction

## What is DevOps anyway?

There is plenty of literature detailing DevOps, so I'll skip the deep dive. In short, it is considered a best practice
in general software development. In this methodology, the development flow is represented as an infinite loop of the
following phases:

| Phase     | Process in Blender Addon Development                     |
|-----------|----------------------------------------------------------|
| Plan      | Planning feature implementation, bug fixes, etc.         |
| Code ★    | Writing Python code                                      |
| Build ★   | Creating a clean ZIP file                                |
| Test ★    | Manual or automated testing                              |
| Release ★ | Publishing the tested ZIP (GitHub/Extensions)            |
| Deploy    | Users downloading the ZIP and installing it into Blender |
| Operate   | Users actually using the addon for production            |
| Monitor   | Receiving Issues and feedback from users                 |

![DevOps](https://raw.githubusercontent.com/unclepomedev/dev_to_writings/main/images/blender_addon_needs_devops/1.jpeg)

Of these eight phases, while there are best practices for *Plan*, I will omit them here.

Also, *Deploy* and *Operate* are out of the developer's hands. Regarding *Monitor*, especially for addons that have
passed the Extensions review, you are generally limited to specific methods determined by the platform, such as GitHub
Issues.

So, while I put "DevOps" in the title, this article focuses on the phases marked with "★": Code, Build, Test, and
Release.

[^2]: Depending on monitoring requirements, distributing addons via external platforms might seem like the only option.
For addons distributed on Extensions, online access must be opt-in according to the terms, and online access tends to be
disliked (and might not pass review). Also, due to third-party package usage restrictions, you cannot embed tools
like [Sentry](https://sentry.io/welcome/). It would be great to easily grasp which Blender versions are crashing to
provide stable support, though.

## Motivation (Feel free to skip)

Blender 5 was officially released last month. I happened to decide to start Blender just a few days before 5 came out,
so I installed the beta version of 5 from the start. At the same time, I installed about 30 addons, both paid and free,
but I found that a multitude of them were spewing errors.

Since it was a beta version at the time, I ignored it, thinking it was unavoidable due to breaking API changes. However,
even now, a little while after the official release, there are still quite a few addons with no working releases. If the
maximum supported version is written, the story ends with "It's my fault for forcing it into Blender 5." But for those
with descriptions that read like "4.2 and later supported" that still don't work, I'm left wondering: "Should I wait, or
do I have to fix it myself?"

If there was a simple CI setup to run even a single pass, basic compatibility errors could be detected. In that case,
maintainers could temporarily declare Blender 5 out of support scope, or explicitly declare the end of support. (Though,
if the maintainer has disappeared, there's nothing to be done.)

Also, when writing my own addon, I investigated repositories of various sizes. I felt that **the culture of testing in
the Blender addon category is weaker compared to other software development areas.** While projects close to the
official ones often have automated tests and delivery environments, individual developer repositories—even ultra-major
ones—often lack anything resembling test automation.

Since there are often no manual test procedures either, even if I want to contribute, I can't submit a PR with
confidence because I don't know where my changes might cause side effects. Especially with major addons, the code tends
to become bloated in the process of responding to feature requests, making it even harder to trace. Maintainers don't
memorize the entire codebase either, so the psychological burden of LGTM-ing a PR is likely high.

There may be issues specific to the characteristics of Blender addons[^1], but I feel the main reason is simply that
practices common in general software development haven't permeated the community. In this article, I propose one method.
Some parts might be personal preference, but others should be directly adoptable. Let's aim for a state where we can run
automated tests whenever there's a feature change or a Blender update, allowing for stress-free maintenance!

[^1]: For example, logic tends to be tightly coupled with appearance and state, making test design difficult.

# Main Topic

Recently, I released my first addon, "SavePoints," on Blender Extensions. Since it was my first work, I implemented it
with the intention of creating a template, specifically focusing on DevOps.

It's an addon that saves `.blend` files with thumbnails and notes, and performs periodic auto-saves. Here, I will
introduce specific implementation points.

https://extensions.blender.org/approval-queue/savepoints/

## Addon Implementation

"Maintainability" has various contexts, but here are three points I kept in mind to improve "Testability" and "Operation
in CI (Headless Environment)."

By the way, a "headless environment" refers to execution without a GUI, which can be done by adding `-b` to the command
line arguments.

https://docs.blender.org/manual/en/latest/advanced/command_line/arguments.html

### Point 1: Separate Logic and Operators

I keep [`operators.py`](https://github.com/unclepomedev/blender_savepoints/blob/main/savepoints/operators.py#L204) (the
interface with Blender) as thin as possible and extract the actual processing into functions in [
`core.py`](https://github.com/unclepomedev/blender_savepoints/blob/main/savepoints/core.py#L293) that do not depend on
`bpy`.

```python:operators.py
class SAVEPOINTS_OT_delete(bpy.types.Operator):
    # ...
    def execute(self, context):
        # ...

        # The actual processing calls a function in core.py
        delete_version_by_id(item.version_id) 

        # ...
        return {'FINISHED'}