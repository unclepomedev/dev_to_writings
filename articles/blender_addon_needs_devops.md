---
description: Blender Addon Development Needs More DevOps.
id: 3087857
published: false
series: Blender Addon Development
tags:
- python
- blender
- devops
- cicd
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

There is plenty of literature detailing DevOps, so I'll skip the deep dive. In short, it is considered a best practice in general software development. In this methodology, the development flow is represented as an infinite loop of the following phases:

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

Also, *Deploy* and *Operate* are out of the developer's hands. Regarding *Monitor*, especially for addons that have passed the Extensions review, you are generally limited to specific methods determined by the platform, such as GitHub Issues.

So, while I put "DevOps" in the title, this article focuses on the phases marked with "★": Code, Build, Test, and Release.

## Motivation (Feel free to skip)

Blender 5 was officially released last month. I happened to decide to start Blender just a few days before 5 came out, so I installed the beta version of 5 from the start. At the same time, I installed about 30 addons, both paid and free, but I found that a multitude of them were spewing errors.

Since it was a beta version at the time, I ignored it, thinking it was unavoidable due to breaking API changes. However, even now, a little while after the official release, there are still quite a few addons with no working releases. If the maximum supported version is written, the story ends with "It's my fault for forcing it into Blender 5." But for those with descriptions that read like "4.2 and later supported" that still don't work, I'm left wondering: "Should I wait, or do I have to fix it myself?"

If there was a simple CI setup to run even a single pass, basic compatibility errors could be detected. In that case, maintainers could temporarily declare Blender 5 out of support scope, or explicitly declare the end of support. (Though, if the maintainer has disappeared, there's nothing to be done.)

Also, when writing my own addon, I investigated repositories of various sizes. I felt that **the culture of testing in the Blender addon category is weaker compared to other software development areas.** While projects close to the official ones often have automated tests and delivery environments, individual developer repositories—even ultra-major ones—often lack anything resembling test automation.

Since there are often no manual test procedures either, even if I want to contribute, I can't submit a PR with confidence because I don't know where my changes might cause side effects. Especially with major addons, the code tends to become bloated in the process of responding to feature requests, making it even harder to trace. Maintainers don't memorize the entire codebase either, so the psychological burden of LGTM-ing a PR is likely high.

There may be issues specific to the characteristics of Blender addons, but I feel the main reason is simply that practices common in general software development haven't permeated the community. In this article, I propose one method. Some parts might be personal preference, but others should be directly adoptable. Let's aim for a state where we can run automated tests whenever there's a feature change or a Blender update, allowing for stress-free maintenance!

# Main Topic

Recently, I released my first addon, "SavePoints," on Blender Extensions. Since it was my first work, I implemented it with the intention of creating a template, specifically focusing on DevOps.

It's an addon that saves `.blend` files with thumbnails and notes, and performs periodic auto-saves. Here, I will introduce specific implementation points.

https://extensions.blender.org/approval-queue/savepoints/

## Addon Implementation

"Maintainability" has various contexts, but here are three points I kept in mind to improve "Testability" and "Operation in CI (Headless Environment)."

By the way, a "headless environment" refers to execution without a GUI, which can be done by adding `-b` to the command line arguments.

https://docs.blender.org/manual/en/latest/advanced/command_line/arguments.html

### Point 1: Separate Logic and Operators

I keep [`operators.py`](https://github.com/unclepomedev/blender_savepoints/blob/main/savepoints/operators.py#L204) (the interface with Blender) as thin as possible and extract the actual processing into functions in [`core.py`](https://github.com/unclepomedev/blender_savepoints/blob/main/savepoints/core.py#L293) that do not depend on `bpy`.

```python
class SAVEPOINTS_OT_delete(bpy.types.Operator):
    # ...
    def execute(self, context):
        # ...

        # The actual processing calls a function in core.py
        delete_version_by_id(item.version_id) 

        # ...
        return {'FINISHED'}
```

```python
def delete_version_by_id(version_id: str) -> None:
    # ... Only standard Python processing like shutil, json operations, etc., without using bpy
```

This makes it possible to verify logic even in unit tests that do not launch Blender.

### Point 2: Use `bpy.data`, not `bpy.ops`

This seems to be a [long-standing rule of thumb](https://blender.stackexchange.com/questions/2848/why-avoid-bpy-ops), but I implement with the policy: "First consider operations with `bpy.data`, and only consider `bpy.ops` if absolutely necessary."

`bpy.ops` is a "simulation of user operations" and requires an appropriate context (mouse position, active window, etc.) to execute. Therefore, in a headless environment like CI, [Poll errors (incorrect context)](https://docs.blender.org/api/current/info_gotchas_operators.html#why-does-an-operator-s-poll-fail) occur, making test implementation difficult. From a performance perspective as well, you should use the low-level data API `bpy.data` whenever possible.

### Point 3: Create an Escape Hatch for Viewport-Dependent Processing

Anticipating that CI environments basically do not have a GUI, I create escape hatches (guard clauses) to prevent crashing with errors.

For example, in [`core.py`](https://github.com/unclepomedev/blender_savepoints/blob/main/savepoints/core.py#L210), when capturing a thumbnail using OpenGL, an error occurs in headless mode because no window exists. So, I suppress the exception and allow the process to continue.

```python
def capture_thumbnail(context: bpy.types.Context, thumb_path: str) -> None:
    # ...
    try:
        # In headless mode, windows is empty
        if context.window_manager.windows:
            bpy.ops.render.opengl(write_still=True)
            # ...
        else:
            pass # Do nothing (skip thumbnail generation)
    except Exception as e:
        print(f"Thumbnail generation failed: {e}")
```

To be fair, this has the disadvantage of increasing environment-dependent branching. If you are not going to run in headless mode (i.e., not implementing CI), this processing might just be code bloat and unnecessary.

## CI

To maintain the implemented addon stably and easily, I implement tests and automation.

### Point 1: Two-Tier Testing

I think splitting tests into two layers is cost-effective.

1.  **Logic Tests**
    * Use `pytest` + `fake-bpy-module`.
    * Verify Python logic (path manipulation, data calculation, etc.) without launching Blender.
2.  **E2E Tests**
    * Launch actual Blender in headless mode and run the addon.

In Logic Tests, parts that do not depend on Blender are verified at lightning speed (unless there is computationally heavy processing). On the other hand, E2E Tests run in the actual Blender environment, so while they are somewhat heavier, they reliably verify that hitting the API doesn't cause errors. It's good practice to run Logic Tests frequently and E2E Tests occasionally in the development environment, and at key points like merging into the `main` branch in the CI environment.

Since the main culprit of the "Not working in Blender 5" problem is API changes and behavior differences, the key to preventing this type of issue is how easily and automatically you can run E2E tests.

Note that when using `fake-bpy-module` for type stubs in Logic Tests, runtime errors may occur; in such cases, mocking is recommended.

### Point 2: Introducing the Task Runner `Just`

The first step in test automation is "making the same command work both locally and in CI." To run Blender tests, you usually need to chant a long spell like this:

```bash
/Applications/Blender.app/Contents/MacOS/Blender --background --factory-startup -P tests/test_e2e.py
```

Paths differ by OS, and it's impossible to memorize. So, I introduce a task runner. Anything is fine, but I recommend [`Just`](https://github.com/casey/just).

```just
# Can be overwritten by env var. Default matches the developer's environment (e.g., Mac path)
blender_exe := env_var_or_default('BLENDER_EXE', '/Applications/Blender.app/Contents/MacOS/Blender')

# Run tests
test-blender:
    "{{blender_exe}}" --factory-startup -b -P tests/test_e2e.py

# Release build (Create a clean zip with git archive)
build version:
    git archive --format=zip --output=dist/my_addon_{{version}}.zip HEAD .
```

Now, in the development environment, tests run just by typing `just test-blender`. If you install `just` on CI, you can use the exact same command.

### Point 3: Test Multiple Blender Versions with GitHub Actions Matrix

I want to check operation on both "Blender 4.2 LTS" and "Blender 5.0". But doing it manually is a hassle. This is where the GitHub Actions matrix feature comes in. I believe most other hosting environments like GitLab can do something similar.

```yaml
# [https://github.com/unclepomedev/blender_savepoints/blob/main/.github/workflows/release.yml](https://github.com/unclepomedev/blender_savepoints/blob/main/.github/workflows/release.yml)
# (Excerpt)

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        # Just adding here triggers parallel tests automatically
        blender-version: ["4.2.0", "5.0.0"]

    steps:
      - uses: actions/checkout@v5
      - uses: extractions/setup-just@v3

      # 1. Check cache (Make 2nd run onwards quick)
      - name: Cache Blender
        id: cache-blender
        uses: actions/cache@v4
        with:
          path: blender-bin
          key: blender-${{ runner.os }}-${{ matrix.blender-version }}

      # 2. Download from official source if cache misses
      - name: Download Blender
        if: steps.cache-blender.outputs.cache-hit != 'true'
        run: |
          # Generate URL from version number (e.g., 5.0.0 -> Blender5.0/blender-5.0.0...)
          # Download via curl following official naming conventions
          curl -SL --fail "$URL" -o blender.tar.xz
          # ... (Extraction logic) ...

      # 3. Run Tests
      - name: Run Tests
        run: just test-blender blender_exe='blender'
```

With this, when you want to verify, environments for 4.2 and 5.0 spin up, and tests run in parallel. If either fails, it results in a Failure, preventing the inclusion of changes that "don't work in 5.0."

You need to keep in mind that GitHub Actions Ubuntu runners do not have GPUs, so processes using OpenGL will error out. If this is accounted for during implementation, there is no problem.

By the way, there seem to be third-party actions to setup the Blender environment, and if you check the implementation and judge it's OK, you can use them.

### Point 4: Automatic Release as a Gatekeeper

If tests pass, perform automatic release. At this time, it is good to ensure the build does not run unless tests pass.

```yaml
# (Same excerpt)

  deploy:
    needs: test  # <--- Will not execute unless tests pass on ALL versions
    runs-on: ubuntu-latest
    steps:      
      # ...
      
      # Create a clean zip excluding unnecessary files with git archive
      - run: |
          zip -r ./savepoints_${{ steps.version.outputs.TAG_NAME }}.zip savepoints -x "*.pyc" -x "*__pycache__*"
      
      # Upload to GitHub Releases
      - uses: softprops/action-gh-release@v2
        with:
          files: savepoints_${{ steps.version.outputs.TAG_NAME }}.zip
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

With this, when a version tag is pushed, you achieve the state where "a clean Zip verified on all Blender versions is distributed."

# Conclusion

Let's let machines handle non-creative tasks like "operation verification."

If you think doing all this is excessive, just introducing `Just` will improve the developer experience.

Also, once you build this kind of pipeline as a template, you can easily copy and paste it to other projects. Please try creating your favorite DevOps pipeline.
