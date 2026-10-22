# <img src="./assets/ww-logo.png" alt="WhisperWriter icon" width="25" height="25"> WhisperWriter

![version](https://img.shields.io/badge/version-1.0.1-blue)

<p align="center">
    <img src="./assets/ww-demo-image-02.gif" alt="WhisperWriter demo gif" width="340" height="136">
</p>

> **About this fork.** This is my fork of [savbell/whisper-writer](https://github.com/savbell/whisper-writer), the original app by [@savbell](https://github.com/savbell). I set it up as a fully offline push-to-talk dictation tool on a Windows 11 laptop that has no NVIDIA GPU. To get it running there, I fixed the broken install, a crash on launch, and three Windows bugs that are still open upstream. My changes are summarized below. The rest of this README is the upstream documentation.

## What this fork changes

### How I run it locally

Tap F9, speak, and tap F9 again. The text is pasted into whichever window has focus. Audio stays in memory and is never sent off the machine.

1. **Hotkey.** A global `pynput` keyboard hook listens for the activation key in `press_to_toggle` mode.
2. **Recording.** `sounddevice` records the microphone at 16 kHz mono into a NumPy array. Nothing is written to disk.
3. **Transcription.** `faster-whisper` runs OpenAI's open-source Whisper `base.en` model (74M parameters, English only) on the CPU.
4. **Output.** The text is placed on the clipboard and pasted with Ctrl+V. The previous clipboard text is restored afterwards.

**Working within the hardware.** The laptop has Intel Iris Xe integrated graphics and 16 GB of RAM. With no CUDA available, the model has to be small and fast on the CPU:

- **Model.** The weights are OpenAI's Whisper `base.en`, in [Systran's CTranslate2 conversion](https://huggingface.co/Systran/faster-whisper-base.en) (142 MB, float16 on disk).
- **Runtime.** It runs on [CTranslate2](https://github.com/OpenNMT/CTranslate2) through `faster-whisper` instead of OpenAI's PyTorch reference code. The weights are the same, but it is roughly 2 to 4x faster on CPU and uses a fraction of the memory.
- **Quantization.** I set `compute_type: int8`, which makes CTranslate2 quantize the weights to 8-bit integers when it loads them. The file on disk isn't changed and there is no separate conversion step. `int8` also pins inference to the CPU.

My local `src/config.yaml` (gitignored, so it isn't in this repo):

```yaml
model_options:
  use_api: false
  local:
    model: base.en
    device: cpu
    compute_type: int8
recording_options:
  activation_key: f9
  recording_mode: press_to_toggle
```

**Measured on this laptop:**

| Recording length | Transcription time |
|---|---|
| 5.6 s | 1.02 s |
| 24.9 s | 2.22 s |
| 32.7 s | 2.65 s |

The model loads in about 1.4 s once the OS has the file cached, and in 4 to 10 s from a cold start. While idle, the app uses about 250 MB of RAM and roughly 0.2% of one CPU core. On Windows the hotkey is an OS-level hook, so nothing polls. For better accuracy on proper nouns, `small.en` can be swapped in. It is slower and about a 250 MB download.

### Fixes and changes to upstream

**Install on Python 3.13** ([`5345d3a`](https://github.com/JigneshAmmineni/whisper-writer/commit/5345d3a)). The upstream `requirements.txt` is a UTF-16 `pip freeze` with 77 pins from 2024. It no longer installs on Windows because `av==11.0.0` has no wheels on PyPI anymore (upstream #139, #142). I replaced it with 12 loosely pinned dependencies that install from prebuilt wheels on Python 3.13. `av` is capped below 19 because `faster-whisper` 1.2.1 still passes the `metadata_errors` argument that PyAV 19 removed.

**Crash on launch** ([`5345d3a`](https://github.com/JigneshAmmineni/whisper-writer/commit/5345d3a)). The app died with a native access violation inside the `WhisperModel` constructor, without a Python traceback. I narrowed it down: just importing PyQt5 before loading the model was enough to trigger it. PyQt5 ships an older `msvcp140.dll`, and ctranslate2 faults when it binds to that runtime. The fix is to import `ctranslate2` before PyQt5 in `src/main.py`. This is likely the crash in upstream #47.

**Start listening at launch** ([`5345d3a`](https://github.com/JigneshAmmineni/whisper-writer/commit/5345d3a)). The hotkey listener now starts as soon as the model loads, so there's no window or Start button to click first. Settings and Exit are still in the tray menu.

**Hotkey firing on the wrong keys** ([`4cee72e`](https://github.com/JigneshAmmineni/whisper-writer/commit/4cee72e), upstream #65). On Windows, the default `ctrl+shift+space` hotkey also fired on any ctrl+shift+letter. Two things combined to cause this. The pynput backend translated every key it didn't recognize to Space, and with Ctrl held, Windows reports letters as control characters that are missing from the key map. Unknown keys and mouse buttons are now dropped instead.

**Listener leak** ([`4cee72e`](https://github.com/JigneshAmmineni/whisper-writer/commit/4cee72e)). `start()` ran after every transcription and created a new keyboard hook and mouse hook each time without stopping the old ones. As a result, every keystroke passed through more and more hooks. `start()` now keeps live listeners and only recreates them if one has died, so the app can't go deaf after a callback error.

**Dropped or doubled characters** ([`4cee72e`](https://github.com/JigneshAmmineni/whisper-writer/commit/4cee72e), upstream #140, #85, #114). Upstream types the text one keystroke at a time, and some Windows apps can't keep up. I added a `clipboard` input method and made it the default. The clipboard is restored with a Qt timer instead of `time.sleep`, because Windows serves the paste request through the app's own event loop, and blocking that loop would stall the paste. Choose `pynput` in settings for terminals like Git Bash, where Ctrl+V doesn't paste.

**Verification.** Each fix was checked with synthetic key and mouse events against both the old and new code. Both bugs reproduced on the old code and were gone on the new code. The clipboard path was also checked with a stubbed keyboard, and the app was launched live. Before I ran it, I also read through the upstream code (about 2,300 lines) for security. I found no `eval`/`exec`/`pickle` and no network calls other than the optional OpenAI API client, and nothing records keystrokes beyond the set of currently held keys.

### Known limitations

- The clipboard restore only brings back plain text. Rich text comes back as plain text, and images or files are lost.
- If Windows clipboard history is on, it records each transcription.
- Pasting doesn't work into elevated (admin) windows, because of Windows UIPI.
- Text appears one to three seconds after you stop talking, not live.

## Original README

**Update (2024-05-28):** I've just merged in a major rewrite of WhisperWriter! We've migrated from using `tkinter` to using `PyQt5` for the UI, added a new settings window for configuration, a new continuous recording mode, support for a local API, and more! Please be patient as I work out any bugs that may have been introduced in the process. If you encounter any problems, please [open a new issue](https://github.com/savbell/whisper-writer/issues)!

WhisperWriter is a small speech-to-text app that uses [OpenAI's Whisper model](https://openai.com/research/whisper) to auto-transcribe recordings from a user's microphone to the active window.

Once started, the script runs in the background and waits for a keyboard shortcut to be pressed (`ctrl+shift+space` by default). When the shortcut is pressed, the app starts recording from your microphone. There are four recording modes to choose from:
- `continuous` (default): Recording will stop after a long enough pause in your speech. The app will transcribe the text and then start recording again. To stop listening, press the keyboard shortcut again.
- `voice_activity_detection`: Recording will stop after a long enough pause in your speech. Recording will not start until the keyboard shortcut is pressed again.
- `press_to_toggle` Recording will stop when the keyboard shortcut is pressed again. Recording will not start until the keyboard shortcut is pressed again.
- `hold_to_record` Recording will continue until the keyboard shortcut is released. Recording will not start until the keyboard shortcut is held down again.

You can change the keyboard shortcut (`activation_key`) and recording mode in the [Configuration Options](#configuration-options). While recording and transcribing, a small status window is displayed that shows the current stage of the process (but this can be turned off). Once the transcription is complete, the transcribed text will be automatically written to the active window.

The transcription can either be done locally through the [faster-whisper Python package](https://github.com/SYSTRAN/faster-whisper/) or through a request to [OpenAI's API](https://platform.openai.com/docs/guides/speech-to-text). By default, the app will use a local model, but you can change this in the [Configuration Options](#configuration-options). If you choose to use the API, you will need to either provide your OpenAI API key or change the base URL endpoint.

**Fun fact:** Almost the entirety of the initial release of the project was pair-programmed with [ChatGPT-4](https://openai.com/product/gpt-4) and [GitHub Copilot](https://github.com/features/copilot) using VS Code. Practically every line, including most of this README, was written by AI. After the initial prototype was finished, WhisperWriter was used to write a lot of the prompts as well!

## Getting Started

### Prerequisites
Before you can run this app, you'll need to have the following software installed:

- Git: [https://git-scm.com/downloads](https://git-scm.com/downloads)
- Python `3.11`: [https://www.python.org/downloads/](https://www.python.org/downloads/)

If you want to run `faster-whisper` on your GPU, you'll also need to install the following NVIDIA libraries:

- [cuBLAS for CUDA 12](https://developer.nvidia.com/cublas)
- [cuDNN 8 for CUDA 12](https://developer.nvidia.com/cudnn)

<details>
<summary>More information on GPU execution</summary>

The below was taken directly from the [`faster-whisper` README](https://github.com/SYSTRAN/faster-whisper?tab=readme-ov-file#gpu):

**Note:** The latest versions of `ctranslate2` support CUDA 12 only. For CUDA 11, the current workaround is downgrading to the `3.24.0` version of `ctranslate2` (This can be done with `pip install --force-reinsall ctranslate2==3.24.0`).

There are multiple ways to install the NVIDIA libraries mentioned above. The recommended way is described in the official NVIDIA documentation, but we also suggest other installation methods below.

#### Use Docker

The libraries (cuBLAS, cuDNN) are installed in these official NVIDIA CUDA Docker images: `nvidia/cuda:12.0.0-runtime-ubuntu20.04` or `nvidia/cuda:12.0.0-runtime-ubuntu22.04`.

#### Install with `pip` (Linux only)

On Linux these libraries can be installed with `pip`. Note that `LD_LIBRARY_PATH` must be set before launching Python.

```bash
pip install nvidia-cublas-cu12 nvidia-cudnn-cu12

export LD_LIBRARY_PATH=`python3 -c 'import os; import nvidia.cublas.lib; import nvidia.cudnn.lib; print(os.path.dirname(nvidia.cublas.lib.__file__) + ":" + os.path.dirname(nvidia.cudnn.lib.__file__))'`
```

**Note**: Version 9+ of `nvidia-cudnn-cu12` appears to cause issues due its reliance on cuDNN 9 (Faster-Whisper does not currently support cuDNN 9). Ensure your version of the Python package is for cuDNN 8.

#### Download the libraries from Purfview's repository (Windows & Linux)

Purfview's [whisper-standalone-win](https://github.com/Purfview/whisper-standalone-win) provides the required NVIDIA libraries for Windows & Linux in a [single archive](https://github.com/Purfview/whisper-standalone-win/releases/tag/libs). Decompress the archive and place the libraries in a directory included in the `PATH`.

</details>

### Installation
To set up and run the project, follow these steps:

#### 1. Clone the repository:

```
git clone https://github.com/savbell/whisper-writer
cd whisper-writer
```

#### 2. Create a virtual environment and activate it:

```
python -m venv venv

# For Linux and macOS:
source venv/bin/activate

# For Windows:
venv\Scripts\activate
```

#### 3. Install the required packages:

```
pip install -r requirements.txt
```

#### 4. Run the Python code:

```
python run.py
```

#### 5. Configure and start WhisperWriter:
On first run, a Settings window should appear. Once configured and saved, another window will open. Press "Start" to activate the keyboard listener. Press the activation key (`ctrl+shift+space` by default) to start recording and transcribing to the active window.

### Configuration Options

WhisperWriter uses a configuration file to customize its behaviour. To set up the configuration, open the Settings window:

<p align="center">
    <img src="./assets/ww-settings-demo.gif" alt="WhisperWriter Settings window demo gif" width="350" height="350">
</p>

#### Model Options
- `use_api`: Toggle to choose whether to use the OpenAI API or a local Whisper model for transcription. (Default: `false`)
- `common`: Options common to both API and local models.
  - `language`: The language code for the transcription in [ISO-639-1 format](https://en.wikipedia.org/wiki/List_of_ISO_639_language_codes). (Default: `null`)
  - `temperature`: Controls the randomness of the transcription output. Lower values make the output more focused and deterministic. (Default: `0.0`)
  - `initial_prompt`: A string used as an initial prompt to condition the transcription. More info: [OpenAI Prompting Guide](https://platform.openai.com/docs/guides/speech-to-text/prompting). (Default: `null`)

- `api`: Configuration options for the OpenAI API. See the [OpenAI API documentation](https://platform.openai.com/docs/api-reference/audio/create?lang=python) for more information.
  - `model`: The model to use for transcription. Currently, only `whisper-1` is available. (Default: `whisper-1`)
  - `base_url`: The base URL for the API. Can be changed to use a local API endpoint, such as [LocalAI](https://localai.io/). (Default: `https://api.openai.com/v1`)
  - `api_key`: Your API key for the OpenAI API. Required for non-local API usage. (Default: `null`)

- `local`: Configuration options for the local Whisper model.
  - `model`: The model to use for transcription. The larger models provide better accuracy but are slower. See [available models and languages](https://github.com/openai/whisper?tab=readme-ov-file#available-models-and-languages). (Default: `base`)
  - `device`: The device to run the local Whisper model on. Use `cuda` for NVIDIA GPUs, `cpu` for CPU-only processing, or `auto` to let the system automatically choose the best available device. (Default: `auto`)
  - `compute_type`: The compute type to use for the local Whisper model. [More information on quantization here](https://opennmt.net/CTranslate2/quantization.html). (Default: `default`)
  - `condition_on_previous_text`: Set to `true` to use the previously transcribed text as a prompt for the next transcription request. (Default: `true`)
  - `vad_filter`: Set to `true` to use [a voice activity detection (VAD) filter](https://github.com/snakers4/silero-vad) to remove silence from the recording. (Default: `false`)
  - `model_path`: The path to the local Whisper model. If not specified, the default model will be downloaded. (Default: `null`)

#### Recording Options
- `activation_key`: The keyboard shortcut to activate the recording and transcribing process. Separate keys with a `+`. (Default: `ctrl+shift+space`)
- `input_backend`: The input backend to use for detecting key presses. `auto` will try to use the best available backend. (Default: `auto`)
- `recording_mode`: The recording mode to use. Options include `continuous` (auto-restart recording after pause in speech until activation key is pressed again), `voice_activity_detection` (stop recording after pause in speech), `press_to_toggle` (stop recording when activation key is pressed again), `hold_to_record` (stop recording when activation key is released). (Default: `continuous`)
- `sound_device`: The numeric index of the sound device to use for recording. To find device numbers, run `python -m sounddevice`. (Default: `null`)
- `sample_rate`: The sample rate in Hz to use for recording. (Default: `16000`)
- `silence_duration`: The duration in milliseconds to wait for silence before stopping the recording. (Default: `900`)
- `min_duration`: The minimum duration in milliseconds for a recording to be processed. Recordings shorter than this will be discarded. (Default: `100`)

#### Post-processing Options
- `writing_key_press_delay`: The delay in seconds between each key press when writing the transcribed text. (Default: `0.005`)
- `remove_trailing_period`: Set to `true` to remove the trailing period from the transcribed text. (Default: `false`)
- `add_trailing_space`: Set to `true` to add a space to the end of the transcribed text. (Default: `true`)
- `remove_capitalization`: Set to `true` to convert the transcribed text to lowercase. (Default: `false`)
- `input_method`: The method to use for simulating keyboard input. (Default: `clipboard`)

#### Miscellaneous Options
- `print_to_terminal`: Set to `true` to print the script status and transcribed text to the terminal. (Default: `true`)
- `hide_status_window`: Set to `true` to hide the status window during operation. (Default: `false`)
- `noise_on_completion`: Set to `true` to play a noise after the transcription has been typed out. (Default: `false`)

If any of the configuration options are invalid or not provided, the program will use the default values.

## Known Issues

You can see all reported issues and their current status in our [Issue Tracker](https://github.com/savbell/whisper-writer/issues). If you encounter a problem, please [open a new issue](https://github.com/savbell/whisper-writer/issues/new) with a detailed description and reproduction steps, if possible.

## Roadmap
Below are features I am planning to add in the near future:
- [x] Restructuring configuration options to reduce redundancy
- [x] Update to use the latest version of the OpenAI API
- [ ] Additional post-processing options:
  - [ ] Simple word replacement (e.g. "gonna" -> "going to" or "smiley face" -> "😊")
  - [ ] Using GPT for instructional post-processing
- [x] Updating GUI
- [ ] Creating standalone executable file

Below are features not currently planned:
- [ ] Pipelining audio files

Implemented features can be found in the [CHANGELOG](CHANGELOG.md).

## Contributing

Contributions are welcome! I created this project for my own personal use and didn't expect it to get much attention, so I haven't put much effort into testing or making it easy for others to contribute. If you have ideas or suggestions, feel free to [open a pull request](https://github.com/savbell/whisper-writer/pulls) or [create a new issue](https://github.com/savbell/whisper-writer/issues/new). I'll do my best to review and respond as time allows.

## Credits

- [OpenAI](https://openai.com/) for creating the Whisper model and providing the API. Plus [ChatGPT](https://chat.openai.com/), which was used to write a lot of the initial code for this project.
- [Guillaume Klein](https://github.com/guillaumekln) for creating the [faster-whisper Python package](https://github.com/SYSTRAN/faster-whisper).
- All of our [contributors](https://github.com/savbell/whisper-writer/graphs/contributors)!

## License

This project is licensed under the GNU General Public License. See the [LICENSE](LICENSE) file for details.
