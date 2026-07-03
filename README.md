[![Runpod](https://api.runpod.io/badge/kodxana/whisperx-worker_v2)](https://console.runpod.io/hub/kodxana/whisperx-worker_v2)

# WhisperX Worker for Runpod

A serverless worker that provides high-quality speech transcription with timestamp alignment and speaker diarization using WhisperX on the Runpod platform.

## Prerequisites

Diarization and speaker verification require access to gated models on Hugging Face. You must accept the terms for each model before using those features:

1. [pyannote/speaker-diarization-community-1](https://huggingface.co/pyannote/speaker-diarization-community-1) — required for diarization
2. [pyannote/embedding](https://huggingface.co/pyannote/embedding) — required for speaker verification and per-speaker embeddings

Set your Hugging Face token as the `HF_TOKEN` environment variable on your Runpod endpoint. The worker will use it automatically for diarization and speaker verification — no need to send it with every request.

You can also pass `huggingface_access_token` per-request to override the env var.

## Features

- Automatic speech transcription with WhisperX
- Automatic language detection
- Word-level timestamp alignment
- Speaker diarization (optional)
- Speaker verification against reference samples (optional)
- Per-speaker voiceprint embeddings for external storage/matching (optional)
- Base64 audio input (no need to host files)
- Highly parallelized batch processing
- Voice activity detection with configurable parameters
- Runpod serverless compatibility

## Input Parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `audio_file` | string | Yes | N/A | URL to the audio file, or base64-encoded audio data (optionally with data URI prefix) |
| `language` | string | No | `null` | ISO code of the language spoken in the audio (e.g., 'en', 'fr'). If not specified, automatic detection will be performed |
| `language_detection_min_prob` | float | No | `0` | Minimum probability threshold for language detection |
| `language_detection_max_tries` | int | No | `5` | Maximum number of attempts for language detection |
| `initial_prompt` | string | No | `null` | Optional text to provide as a prompt for the first transcription window |
| `batch_size` | int | No | `64` | Batch size for parallelized input audio transcription |
| `temperature` | float | No | `0` | Temperature to use for sampling (higher = more random) |
| `vad_onset` | float | No | `0.500` | Voice Activity Detection onset threshold |
| `vad_offset` | float | No | `0.363` | Voice Activity Detection offset threshold |
| `align_output` | bool | No | `false` | Whether to align Whisper output for accurate word-level timestamps |
| `diarization` | bool | No | `false` | Whether to assign speaker ID labels to segments |
| `huggingface_access_token` | string | No | `null` | HuggingFace token for diarization. Overrides the `HF_TOKEN` env var if provided |
| `min_speakers` | int | No | `null` | Minimum number of speakers (only applicable if diarization is enabled) |
| `max_speakers` | int | No | `null` | Maximum number of speakers (only applicable if diarization is enabled) |
| `debug` | bool | No | `false` | Whether to print compute/inference times and memory usage information |
| `speaker_samples` | list | No | `[]` | List of `{name, url}` reference-audio objects, used when `speaker_verification` is enabled |
| `speaker_verification` | bool | No | `false` | Match diarized speakers against the reference audio in `speaker_samples` and relabel segments with the closest match |
| `include_embeddings` | bool | No | `false` | Return one L2-normalized voiceprint centroid per diarized speaker (requires `diarization: true`). See [With Speaker Embeddings](#with-speaker-embeddings) |

## Usage Examples

### Basic Transcription

```json
{
  "input": {
    "audio_file": "https://github.com/runpod-workers/sample-inputs/raw/main/audio/gettysburg.wav"
  }
}
```

### Base64 Audio Input

You can send audio directly as base64-encoded data instead of a URL. This supports raw base64 or data URI format:

```json
{
  "input": {
    "audio_file": "data:audio/wav;base64,UklGRi..."
  }
}
```

Or without the data URI prefix:

```json
{
  "input": {
    "audio_file": "UklGRi..."
  }
}
```

Note: Runpod payload limits apply (20 MB for `/runsync`, 10 MB for `/run`). Compress audio to MP3/OGG before encoding for larger files.

### Transcription with Language Detection and Alignment

```json
{
  "input": {
    "audio_file": "https://github.com/runpod-workers/sample-inputs/raw/main/audio/gettysburg.wav",
    "align_output": true,
    "batch_size": 32,
    "debug": true
  }
}
```

### Full Configuration with Diarization

```json
{
  "input": {
    "audio_file": "https://github.com/runpod-workers/sample-inputs/raw/main/audio/gettysburg.wav",
    "language": "en",
    "batch_size": 32,
    "temperature": 0.2,
    "align_output": true,
    "diarization": true,
    "huggingface_access_token": "YOUR_HUGGINGFACE_TOKEN",
    "min_speakers": 2,
    "max_speakers": 5,
    "debug": true
  }
}
```
### Full Configuration with Speaker Verification

There is no limit to the number of voices you can upload, but precision may be reduced over a certain threshold.

```json
{
  "input": {
    "audio_file": "https://example.com/audio/sample.mp3",
    "language": "en",
    "batch_size": 32,
    "temperature": 0.2,
    "align_output": true,
    "diarization": true,
    "huggingface_access_token": "YOUR_HUGGINGFACE_TOKEN",
    "min_speakers": 2,
    "max_speakers": 5,
    "debug": true,
    "speaker_verification": true,
    "speaker_samples": [
      {
        "name": "Speaker1",
        "url": "https://example.com/speaker1.wav"
      },
      {
        "name": "Speaker2",
        "url": "https://example.com/speaker2.wav"
      },
      {
        "name": "Speaker3",
        "url": "https://example.com/speaker3.wav"
      }
    ]
  }
}
```

### Per-Speaker Voiceprint Embeddings

Set `include_embeddings: true` (together with `diarization: true`) to get one averaged, L2-normalized embedding vector per diarized speaker. Unlike `speaker_verification`, this does **not** match against reference audio — it returns the raw voiceprints so you can store and match them yourself over time (e.g. to recognise recurring speakers across jobs without re-supplying samples on every request).

```json
{
  "input": {
    "audio_file": "https://github.com/runpod-workers/sample-inputs/raw/main/audio/gettysburg.wav",
    "diarization": true,
    "include_embeddings": true
  }
}
```

If `include_embeddings` is requested without `diarization: true`, it is skipped and a `warning` is returned in the response.

## Output Format

The service returns a JSON object structured as follows:

### Without Diarization

```json
{
  "segments": [
    {
      "start": 0.0,
      "end": 2.5,
      "text": "Transcribed text segment 1",
      "words": [
        {"word": "Transcribed", "start": 0.1, "end": 0.7},
        {"word": "text", "start": 0.8, "end": 1.2},
        {"word": "segment", "start": 1.3, "end": 1.9},
        {"word": "1", "start": 2.0, "end": 2.4}
      ]
    },
    {
      "start": 2.5,
      "end": 5.0,
      "text": "Transcribed text segment 2",
      "words": [
        {"word": "Transcribed", "start": 2.6, "end": 3.2},
        {"word": "text", "start": 3.3, "end": 3.7},
        {"word": "segment", "start": 3.8, "end": 4.4},
        {"word": "2", "start": 4.5, "end": 4.9}
      ]
    }
  ],
  "detected_language": "en",
  "language_probability": 0.997
}
```

### With Diarization

```json
{
  "segments": [
    {
      "start": 0.0,
      "end": 2.5,
      "text": "Transcribed text segment 1",
      "words": [
        {"word": "Transcribed", "start": 0.1, "end": 0.7, "speaker": "SPEAKER_01"},
        {"word": "text", "start": 0.8, "end": 1.2, "speaker": "SPEAKER_01"},
        {"word": "segment", "start": 1.3, "end": 1.9, "speaker": "SPEAKER_01"},
        {"word": "1", "start": 2.0, "end": 2.4, "speaker": "SPEAKER_01"}
      ],
      "speaker": "SPEAKER_01"
    },
    {
      "start": 2.5,
      "end": 5.0,
      "text": "Transcribed text segment 2",
      "words": [
        {"word": "Transcribed", "start": 2.6, "end": 3.2, "speaker": "SPEAKER_02"},
        {"word": "text", "start": 3.3, "end": 3.7, "speaker": "SPEAKER_02"},
        {"word": "segment", "start": 3.8, "end": 4.4, "speaker": "SPEAKER_02"},
        {"word": "2", "start": 4.5, "end": 4.9, "speaker": "SPEAKER_02"}
      ],
      "speaker": "SPEAKER_02"
    }
  ],
  "detected_language": "en",
  "language_probability": 0.997,
  "speakers": {
    "SPEAKER_01": {"name": "Speaker 1", "time": 2.5},
    "SPEAKER_02": {"name": "Speaker 2", "time": 2.5}
  }
}
```

### With Speaker Embeddings

When `include_embeddings: true` (and `diarization: true`), the response adds a `speaker_embeddings` object keyed by diarization label. Each entry is that speaker's averaged, L2-normalized voiceprint — the mean of its per-segment `pyannote/embedding` vectors — plus the total speech duration used to compute it. A label with no successfully embedded segments is omitted.

```json
{
  "segments": [ ... ],
  "detected_language": "en",
  "speaker_embeddings": {
    "SPEAKER_00": {
      "embedding": [0.0123, -0.0456, "..."],
      "dimension": 512,
      "model": "pyannote/embedding",
      "speech_seconds": 42.13
    },
    "SPEAKER_01": {
      "embedding": [-0.0079, 0.0210, "..."],
      "dimension": 512,
      "model": "pyannote/embedding",
      "speech_seconds": 17.98
    }
  }
}
```

## Performance Considerations

- **GPU Memory**: Adjust `batch_size` based on available GPU memory for optimal performance
- **Processing Time**: Enabling diarization and alignment will increase processing time
- **File Size**: Large audio files may require more processing time and resources
- **Language Detection**: For shorter audio clips, language detection may be less accurate

## Troubleshooting

### Common Issues

1. **"Model was trained with pyannote.audio 0.0.1, yours is X.X.X"**
   - This is a warning only and shouldn't affect functionality in most cases
   - If issues persist, consider downgrading pyannote.audio

2. **Diarization failures**
   - Ensure you're providing a valid HuggingFace access token
   - Try specifying reasonable min/max speaker values

## Development and Deployment

### Building Your Own Image

```bash
docker build -t your-username/whisperx-worker:your-tag .
```

## License

This project is licensed under the Apache License, Version 2.0. See the [LICENSE](LICENSE) file for details.

## Acknowledgments

- This project utilizes code from [WhisperX](https://github.com/m-bain/whisperX), licensed under the BSD-2-Clause license
- Special thanks to the Runpod team for the serverless platform

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.
