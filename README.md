PHASE TRACE: A PHYSICALLY BASED RAY TRACER FOR THE SIMULATION INDIRECT TIME-OF-FLIGHT CAMERAS
==============================================================================================

This physically based ray tracer based on pbrt-v4 offers simulation of indirect ToF cameras with single wavelength laser illumination including multi-scattering and basic noise simulation with a focus on physical accuracy. The simulation can be run on GPU or CPU. 

For question regarding pbrt-v4 and 2D image handling, check out pbrt-v4's Github page
(https://github.com/mmp/pbrt-v4) and the book "Physically based rendering: From Theory to Implementation", 
by M. Pharr, W. Jakob and G. Humphreys (Found online under: https://pbr-book.org). 

# License

Apache 2.0

# Running the code

## Installation and building

As pbrt-v4, PhaseTrace uses git submodules for a number of third-party libraries
that it depends on.  Therefore, be sure to use the `--recursive` flag when
cloning the repository:
```bash
$ git clone --recursive https://github.com/mmp/pbrt-v4.git
```

If you accidentally clone pbrt without using ``--recursive`` or to update
the pbrt source tree after a new submodule has been added, run the
following command to also fetch the dependencies:
```bash
$ git submodule update --init --recursive
```

Now, installation is a bit difficult. If you only want to run the code on a CPU you do not need to meet additional requirements. For running on a GPU you need a NVIDIA GPU, CUDA 11.0 or later and OptiX 7.1. I do not recommend trying on Windows, but it might just work as well. For me the following specification(s) worked: 

- RTX 4060 or GTX 1050 Ti with Ubuntu 22.04, Nvidia Driver 535, CUDA 12.2 and OptiX 7.6 or 7.7

Following a list with the necessary installations:

1. Find out your NVIDIA driver version. On Ubuntu: `nvidia-smi`
2. Find out potentially available NVIDIA driver versions. On Ubuntu: `sudo ubuntu-drivers list`
3. Check compability of CUDA versions with NVIDIA driver: https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html, Table 2.
4. Install CUDA toolkit. In some cases the version compatible with your driver will not have a download file for your OS. In this case check if any other one of the available driver versions would help you update (or downgrade) to a CUDA version which has a download file for your OS. The CUDA installation usually wants to install the newest drivers, if you do not want to update your NVIDIA driver install with a run file and disable driver installation.

    If you installed CUDA toolkit with run, make sure to add the library path to ~.bashrc with ```sudo nano /etc/bash.bashrc``` and add

    ```bash
    export PATH=/usr/local/cuda-1x.x/bin${PATH:+:${PATH}}
    export LD_LIBRARY_PATH=/usr/local/cuda-1x.x/lib ${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}
    ```

5. Install a compatible OptiX SDK (compatible with your NVIDIA driver, the CUDA version should not matter). The installation bash script just unpacks the OptiX driver. You can either keep it where it is unpacked, e.g., Downloads, or move it somewhere else, e.g., /opt. Wherever your OptiX SDK is, you have to change your CMakeLists.txt accordingly. 

6. There might be some additional libraries missing. Install them just in case or go through cmake and install what's missing. Here's what I had to install:

    ```bash
    sudo apt install mesa-utils libgl1-mesa-dev libglu1-mesa-dev freeglut3-dev libxkbcommon-dev libxrandr-dev libxinerama-dev libxcursor-dev libxi-dev
    libtiff-dev libwayland-dev
    ```

7. Then configure and build the code. Make sure to adapt to your GPU's compute capability. 

    **To build a release version**:

    ```bash
    cd <path_to_this_dir>
    mkdir build
    cd build
    cmake .. -DCMAKE_CUDA_ARCHITECTURES=<your compute capability> 
    cmake --build . --config Release --parallel
    ```

    **Building a Debug Version**:

    To Debug, you have to rebuild the entire code. To also enable LOG_DBG() info add the Logging flags for CUDA and CPU.

    ```bash
    cmake .. -DCMAKE_CUDA_ARCHITECTURES=89 -DCMAKE_BUILD_TYPE=Debug -DCMAKE_CXX_FLAGS="-DPBRT_DBG_LOGGING"   -DCMAKE_CUDA_FLAGS="-DPBRT_DBG_LOGGING"
    cmake --build . --config Debug --parallel
    ```

## Running the code

After building, run the code from the terminal. For this you need a pbrt file of some sort. 

Theoretically, you can render the same images in Debug as in Release, but due to all the additional information, you probably get an error of some sort for files with a large number of pixels or when setting the pixel samples to a high number. For starters, I would recommend sampling with a 5x5 images and `pixelSamples = 4`.

If you have to simulate a lot of files, it may be nicer to define the scene once and then just add 3D PLY files to it. For this purpose, you can use [pbrt_default](/src/pbrt/cmd/pbrt_default.cpp) which can be run similiar to pbrt, but automatically loads a scene. Please remember to pass the ply file directly instead of a pbrt file.

```bash
./pbrt_default --gpu  /files/someScene.ply
```

If you'd like some extra info about the run, you can log them in an external log file. For this add --log-file and --log-level like below:

```bash
./pbrt --gpu  /home/elan/Projects/tof_camerasimulation/test_scenes/wall-tof.pbrt --log-file ./logGPU.txt --log-level verbose
```

This makes the code a lot slower though and only includes logging information captured during CPU commands. If you need information about functions running on GPU, you have to build the Debug version. You should also decrease the pixel samples to 1 and the image size to something small (e.g. 10x10) for the code not to crash due to all the debugging info.

### Example scenes for pbrt-v4

* A number of scenes for pbrt-v4 are [available in a git repository](https://github.com/mmp/pbrt-v4-scenes).
* The [pbrt-v4 User's Guide](https://pbrt.org/users-guide-v4.html).
* Documentation on the [pbrt-v4 Scene Description Format](https://pbrt.org/fileformat-v4.html).

### Example scenes for PhaseTrace

You'll find an explanation of the pbrt scene description in the [docs](/docs/SceneDescription.md). There is also an example scene for ToF simulation.

# Bug Reports and PRs


# PhaseTrace's additions to pbrt-v4

In general the following aspects changed compared to pbrt-v4. In the chapters below I also indicate specific changes that were made to accomodate ToF handling.

- TofFilm: A film class that captures single wavelength light in "buckets" and handles typical ToF phase-to-distance conversion
- Laser: A light class similiar to SpotLight that emits single wavelength light with a batwing intensity profile
- Phase handling: To offer depth calculation, travelled ray distances are now kept and distances are converting to phase when "hitting" a light source. This is mainly in `tof_correlation_func` and wherever `getTofBuckets()` is called.
- Cameras: 
    - Added CameraSpaceToCoordinates to all camera classes to convert radial distances back to cartesian coordinates with the parameters known to each camera class.
    - Added aperture handling to the perspective camera. This enables difference in radiance depending on the aperture similiar to realistic camera without needing to know the lens system specifically.
- Math: some additional things in math.h
- Output: Additional output file types PCD and TIF and their respective handling (image.h and image.cpp)

## Accumulating radiance

Right now, depth calculation is only possible with the Volpath integrator on GPU and CPU (it's the standard for GPU simulation, but CPU simulation technically offers more integrators). There radiance, including depth, are accumulated in the following functions:

- Shadow Ray: RecordShadowRayResult (src/pbrt/wavefront/intersect.h 31)
- Shadow Ray within medium: TraceTransmittance
- Ray leaves scene: HandleEscapedRays
- Hit light: HandleEmissiveIntersection
- Scatter within medium: SampleMediumInteraction (this occurs in medium scattering, depth not really covered currently)

## Noise

In general pbrt (and this simulation as well) is a Monte-Carlo simulation of the radiance's expected value. In case of the ToF camera, we distribute this radiance with the correlation at specific distances into ToF buckets. At the end we have the expected value and with this value we can determine the noise we expect.

For the noise we use:
- shotnoise
- dark current and its noise (created thermally, but poisson distributed)

## TODO

- Sub-surface scattering
- BRDF of specific objects

# Acknowledgments

Matt Pharr, Wenzel Jakob, and Greg Humphreys for their great and well-documented physically based ray tracer pbrt-v4.

Martin Lambers for his ToF simulation WurblPT: The phase calculation in [TofFilm::GetPixelData](/src/pbrt/film.cpp) is heavily inspired by [WurblPT](https://marlam.de/wurblpt/), a CPU-based ray tracer for iToF cameras.

Additionally, a big thank you to all developers of third party libaries who made this possible (see [THIRD_PARTY.md](/THIRD_PARTY.md)).

