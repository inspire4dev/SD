# Requirements

1. Install Homebrew and remember to follow the instructions under "Next steps"
   
   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```
   
1. Install Wget
   
   ```bash
   brew install wget
   ```
   
1. Download and install [Xcode](https://developer.apple.com/download/all/?q=Xcode)

1. Select Xcode as the active Command Line Tools provider.

    There are two ways to achieve this.

    1. In Terminal, run the following command:

        ```bash
        sudo xcode-select -s /Applications/Xcode.app
        ```

    1. Or open Xcode, go to the Xcode menu / Settings... / Locations and select your Xcode version in the "Command Line Tools" picker.

1. Download and Install [Miniconda](https://docs.conda.io/en/latest/miniconda.html)
   
1. Once done, run the commands below according to their display order
   
   ```bash
   git clone https://github.com/apple/ml-stable-diffusion.git
   ```
   
   ```bash
   conda create -n coreml_stable_diffusion python=3.8 -y
   ```
   
   ```bash
   conda activate coreml_stable_diffusion
   ```
   
   ```bash
   cd ml-stable-diffusion
   ```
   
   ```bash
   pip install -e .
   ```
   
   ```bash
   pip install omegaconf
   ```
   
   ```bash
   pip install safetensors
   ```
   
1. Download [this Python script](https://github.com/huggingface/diffusers/raw/main/scripts/convert_original_stable_diffusion_to_diffusers.py) and place it in the same folder as the model

[↑ Back to top](#top)

# SD model → Diffusers

This process takes ~1min to complete.

1. Activate the Conda environment
   
   ```bash
   conda activate coreml_stable_diffusion
   ```
   
1. Navigate to the folder where the script is located via `cd /<YOUR-PATH>` (you can also type `cd ` and then drag the folder into the Terminal app)
   
1. Now you have two options:
   
   1. If your model is in CKPT format, run
   
      ```bash
      python convert_original_stable_diffusion_to_diffusers.py --checkpoint_path <MODEL-NAME>.ckpt --device cpu --extract_ema --dump_path <MODEL-NAME>_diffusers
      ```
   
   1. If your model is in SafeTensors format, run
      
      ```bash
      python convert_original_stable_diffusion_to_diffusers.py --checkpoint_path <MODEL-NAME>.safetensors --from_safetensors --device cpu --extract_ema --dump_path <MODEL-NAME>_diffusers
      ```

### Important Notes

- When exclusively converting SDXL 1.0 models, be sure to include the following flag: `--pipeline_class_name StableDiffusionXLPipeline`

[↑ Back to top](#top)

# Diffusers → MLMODELC

This process takes ~25 minutes to complete.

Each conversion script actually runs twice to make 2 different types of one particular component.  This enables the converted models to work with and without the ControlNet feature.

If you're doing this right after the previous step, ignore points 1 and 2.

1. Activate the Conda environment
   
   ```bash
   conda activate coreml_stable_diffusion
   ```
   
1. Navigate to the folder where the script is located via `cd /<YOUR-PATH>` (you can also type `cd ` and then drag the folder into the Terminal app)
   
1. Now you have two options:
   
   1. `SPLIT_EINSUM`, which is compatible with all compute units
      
      ```bash
      python -m python_coreml_stable_diffusion.torch2coreml --convert-vae-decoder --convert-vae-encoder --convert-unet --unet-support-controlnet --convert-text-encoder --model-version <MODEL-NAME>_diffusers --bundle-resources-for-swift-cli --attention-implementation SPLIT_EINSUM -o <MODEL-NAME>_split-einsum && python -m python_coreml_stable_diffusion.torch2coreml --convert-unet --model-version <MODEL-NAME>_diffusers --bundle-resources-for-swift-cli --attention-implementation SPLIT_EINSUM -o <MODEL-NAME>_split-einsum
      ```
   
   1. `ORIGINAL`, which is only compatible with `CPU & GPU`
      
      ```bash
      python -m python_coreml_stable_diffusion.torch2coreml --compute-unit CPU_AND_GPU --convert-vae-decoder --convert-vae-encoder --convert-unet --unet-support-controlnet --convert-text-encoder --model-version <MODEL-NAME>_diffusers --bundle-resources-for-swift-cli --attention-implementation ORIGINAL -o <MODEL-NAME>_original && python -m python_coreml_stable_diffusion.torch2coreml --compute-unit CPU_AND_GPU --convert-unet --model-version <MODEL-NAME>_diffusers --bundle-resources-for-swift-cli --attention-implementation ORIGINAL -o <MODEL-NAME>_original
      ```
      
      1. Only when using the `ORIGINAL` implementation, it's possible to modify the output image size by adding the `--latent-w <SIZE>` and `--latent-h <SIZE>` flags. For example:
         ```bash
         python -m python_coreml_stable_diffusion.torch2coreml --latent-w 64 --latent-h 96 --compute-unit CPU_AND_GPU --convert-vae-decoder --convert-vae-encoder --convert-unet --unet-support-controlnet --convert-text-encoder --model-version <MODEL-NAME>_diffusers --bundle-resources-for-swift-cli --attention-implementation ORIGINAL -o <MODEL-NAME>_original_512x768 && python -m python_coreml_stable_diffusion.torch2coreml --latent-w 64 --latent-h 96 --compute-unit CPU_AND_GPU --convert-unet --model-version <MODEL-NAME>_diffusers --bundle-resources-for-swift-cli --attention-implementation ORIGINAL -o <MODEL-NAME>_original_512x768
         ```
         
         The chosen image size must be divisible by `64`. Also, you have to specify it divided by `8` (e.g. `768/8=96`).\
         In the example above, the model will always output images at a resolution of 512x768
   
1. The needed files will be created under the `<MODEL-NAME>/Resources` folder. Everything else can be discarded

### Important Notes

- When exclusively converting SDXL 1.0 models, be sure to include the following flag: `--xl-version`
- As of today, `ORIGINAL` implementations with output sizes greater than 512x768 or 768x512, work slowly on lower-performance machines or do not work at all. 768x768 models had been tested with a time of ~1min/step with M1 (and some kernel panics), ~1s/step with M1 Max 32 GPU, and 1024x1024 models just can't be run ([MPSNDArray error: product of dimension sizes > 2**31](https://github.com/pytorch/pytorch/issues/84039)).

[↑ Back to top](#top)

# Troubleshooting

#### Miniconda

- `This package is incompatible with this version of macOS`: after the "Software Licence Agreement" step, click on "Change Install Location..." and select "Install for me only" 

#### Terminal errors

- `xcrun: error: unable to find utility "coremlcompiler", not a developer tool or in PATH`: open Xcode and go to "Settings..." → "Locations" then click on the "Command Line Tools" drop-down menu and reselect the Command Line Tools version
- `ModuleNotFoundError: No module named 'pytorch_lightning'`: while the conda `coreml_stable_diffusion` environment is active, run
  
  ```bash
  pip install pytorch_lightning
  ```
  
  Every time you see a similar message, you can solve it by installing what is requested via `pip install <NAME>`
  
- `zsh: killed python`: your Mac has run out of memory. Close some memory-hungry applications you may have open and do the process again. Still not working? Reboot. Still not working? Use `nice -n 10` before the command. Still not working? Well, `SPLIT_EINSUM` conversions tend to be the more demanding, so while converting, close all the other apps and leave your Mac melting alone

#### Terminal warnings

- If you get any of these
  
  ```
  TracerWarning: Converting a tensor to a Python boolean might cause the trace to be incorrect. We can't record the data flow of Python values, so this value will be treated as a constant in the future. This means that the trace might not generalize to other inputs!
  ```
  
  ```bash
  WARNING:__main__:Casted the `beta`(value=0.0) argument of `baddbmm` op from int32 to float32 dtype for conversion!
  ```
  
  ```bash
  WARNING:coremltools:Tuple detected at graph output. This will be flattened in the converted model.
  ```
  
  ```bash
  WARNING:coremltools:Saving value type of int64 into a builtin type of int32, might lose precision!
  ```
  
  You're fine
