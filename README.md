# VapourSynth-SRMD-ncnn-Vulkan

[![Build Status](https://github.com/Kiyamou/VapourSynth-SRMD-ncnn-Vulkan/workflows/CI/badge.svg)](https://github.com/Kiyamou/VapourSynth-SRMD-ncnn-Vulkan/actions)

SRMD super resolution for VapourSynth, based on [srmd-ncnn-vulkan](https://github.com/nihui/srmd-ncnn-vulkan). Some code is from [vapoursynth-waifu2x-ncnn-vulkan](https://github.com/Nlzy/vapoursynth-waifu2x-ncnn-vulkan).

## Usage

```python
core.srmdnv.SRMD(clip clip, [int scale, int noise, int tilesize_x, int tilesize_y,
                 int gpu_id, int gpu_thread, bool tta])
```

Models should be located in folder `models`, and folder `models` should be located in the same folder as dynamic link library.

* ***clip***
  * Required parameter.
  * Clip to process.
  * Only 32bit RGB is supported.
* ***scale***
  * Optional parameter. Range: 2–4. *Default: 2*.
* ***noise***
  * Optional parameter. Range: -1–10. *Default: 3*.
  * The larger value, the stronger the effect of denoise.
* ***tilesize_x***
  * Optional parameter. *Default: depands on video memory size*
  * The tilesize for horizontal.
  * Recommend to set a value that can divide the width.
* ***tilesize_y***
  * Optional parameter. *Default: same as tilesize_x*.
  * The tilesize for vertical.
  * Recommend to set a value that can divide the height.
* ***gpu_id***
  * Optional parameter. *Default: 0*.
  * If you have more than one gpu devices, you can select gpu device by the parameter.
* ***tta***
  * Optional parameter. *Default: False*.
  * If true, quality will be improved, but speed will be significantly slower.

## VapourSynth API 4 and wheels

This fork targets the VapourSynth API 4 (R55 and newer, required since R80). The namespace `srmdnv`, the function and
its arguments are unchanged. It builds against ncnn 20260526, which loads the Vulkan driver itself, so no Vulkan SDK is
needed.

Each instance compiles its own shaders: with a shader cache shared by all instances, a `tta=True` instance after a
non-TTA one in the same process got the non-TTA shader and failed with `binding_count not match`.

The Windows build of the API 3 version that Hybrid shipped gave wrong colours (red channel nearly black); this build
stays within 36 to 41 dB of a bicubic upscale.

Wheels for Windows x64, Linux x86_64 and macOS arm64 are attached to the GitHub releases. They contain the models of
[srmd-ncnn-vulkan 20220728](https://github.com/nihui/srmd-ncnn-vulkan/releases/tag/20220728):

```
pip install vapoursynth_srmd_ncnn_vulkan-<version>-py3-none-<platform>.whl
```

The plugin and its `models` folder go to `site-packages/vapoursynth/plugins/srmd`.

## Compilation

```
git clone --recursive https://github.com/Selur/VapourSynth-SRMD-ncnn-Vulkan
cd VapourSynth-SRMD-ncnn-Vulkan
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build
```

The VapourSynth API 4 headers are vendored in `include/vapoursynth`. For the tests, put the `models-*` folders of the
srmd-ncnn-vulkan release into `build/models` and run `python3 test/test_srmd.py build/libsrmdnv.so`.

## Reference Code

* srmd-ncnn-vulkan: https://github.com/nihui/srmd-ncnn-vulkan
* vapoursynth-waifu2x-ncnn-vulkan: https://github.com/Nlzy/vapoursynth-waifu2x-ncnn-vulkan
