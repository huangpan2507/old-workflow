# iFlytek Audio Transcription Test Script Guide

## Overview

The `test_xfyun_asr.py` script tests the iFlytek Long Form ASR (Audio Speech Recognition) API for transcribing audio files to text. It supports uploading audio files and retrieving transcription results.

Based on iFlytek API documentation:
https://www.xfyun.cn/doc/asr/ifasr_new/API.html#%E6%8E%A5%E5%8F%A4%E8%AF%B4%E6%98%8E

## Prerequisites

1. iFlytek Open Platform account with API access
2. App ID and Secret Key from iFlytek Open Platform
3. Audio file to transcribe (supported formats: mp3, wav, pcm, aac, opus, flac, og...)
4. Audio properties:
   - Sample rate: 16k or 8k
   - Bit depth: 8bit or 16bit
   - Channels: mono or multi-channel
   - Maximum duration: 5 hours

## Installation

Install required Python packages:

```bash
pip install requests
```

## Usage

### Method 1: Using Command-Line Arguments

#### Basic Usage (Check Result Once)

```bash
python backend/test_xfyun_asr.py \
  --app-id YOUR_APP_ID \
  --secret-key YOUR_SECRET_KEY \
  --audio-file /path/to/audio.wav
```

#### With Duration Specified

```bash
python backend/test_xfyun_asr.py \
  --app-id YOUR_APP_ID \
  --secret-key YOUR_SECRET_KEY \
  --audio-file /path/to/audio.wav \
  --duration 300
```

#### Poll for Result Until Completion

```bash
python backend/test_xfyun_asr.py \
  --app-id YOUR_APP_ID \
  --secret-key YOUR_SECRET_KEY \
  --audio-file /path/to/audio.wav \
  --duration 300 \
  --poll
```

#### Custom Polling Settings

```bash
python backend/test_xfyun_asr.py \
  --app-id YOUR_APP_ID \
  --secret-key YOUR_SECRET_KEY \
  --audio-file /path/to/audio.wav \
  --duration 300 \
  --poll \
  --max-attempts 120 \
  --interval 15
```

### Method 2: Using Environment Variables

```bash
export XFYUN_APP_ID="YOUR_APP_ID"
export XFYUN_SECRET_KEY="YOUR_SECRET_KEY"
export XFYUN_AUDIO_FILE="/path/to/audio.wav"
export XFYUN_AUDIO_DURATION="300"  # Optional, in seconds

python backend/test_xfyun_asr.py
```

### Method 3: Using Environment Variables with Polling

```bash
export XFYUN_APP_ID="YOUR_APP_ID"
export XFYUN_SECRET_KEY="YOUR_SECRET_KEY"
export XFYUN_AUDIO_FILE="/path/to/audio.wav"
export XFYUN_AUDIO_DURATION="300"

python backend/test_xfyun_asr.py --poll
```

## Command-Line Arguments

| Argument | Description | Required | Default |
|----------|-------------|----------|---------|
| `--app-id` | iFlytek App ID | Yes* | - |
| `--secret-key` | iFlytek Secret Key | Yes* | - |
| `--audio-file` | Path to audio file | Yes* | - |
| `--duration` | Audio duration in seconds | No | Estimated |
| `--poll` | Poll for result until completion | No | False |
| `--max-attempts` | Maximum polling attempts | No | 60 |
| `--interval` | Interval between polling attempts (seconds) | No | 10 |

*Required if not set via environment variables

## Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `XFYUN_APP_ID` | iFlytek App ID | Yes* |
| `XFYUN_SECRET_KEY` | iFlytek Secret Key | Yes* |
| `XFYUN_AUDIO_FILE` | Path to audio file | Yes* |
| `XFYUN_AUDIO_DURATION` | Audio duration in seconds | No |

*Required if not provided via command-line arguments

## API Response Status Codes

The transcription status is indicated by the `status` field in the response:

- `2`: Transcription queued
- `3`: Transcription in progress
- `4`: Transcription completed

## Expected Processing Time

According to iFlytek documentation, the expected processing time varies by audio duration:

| Audio Duration (minutes) | Expected Return Time (minutes) |
|-------------------------|-------------------------------|
| X < 10                  | Y < 3                         |
| 10 ≤ X < 30             | 3 ≤ Y < 6                     |
| 30 ≤ X < 60             | 6 ≤ Y < 10                    |
| 60 ≤ X                  | 10 ≤ Y < 20                   |

Note: Actual return time depends on audio duration and current task queue. During peak hours, tasks may be queued.

## Signature Generation

The script automatically generates the required signature (signa) for API authentication using the following algorithm:

1. Get current timestamp (ts)
2. Concatenate app_id and ts to form base_string
3. Calculate MD5 hash of base_string
4. Use HMAC-SHA1 with secret_key to sign the MD5 hash
5. Base64 encode the result to get signa

## Error Handling

The script includes comprehensive error handling:

- File not found errors
- Network request errors
- API response errors
- Invalid configuration errors

All errors are displayed with clear error messages and stack traces for debugging.

## Example Output

### Successful Upload

```
============================================================
Audio File Upload
============================================================
File: /path/to/audio.wav
File name: audio.wav
File size: 5242880 bytes (5.00 MB)
Duration: 300 seconds
Upload URL: https://raasr.xfyun.cn/v2/api/upload?appId=...

✓ Upload successful
  Order ID: 1234567890123456789
```

### Polling for Result

```
============================================================
Polling for Transcription Result
============================================================
Max attempts: 60
Interval: 10 seconds

Attempt 1/60...
============================================================
Get Transcription Result
============================================================
Order ID: 1234567890123456789
Result URL: https://raasr.xfyun.cn/v2/api/getResult?appId=...

✓ Request successful
  Status: 3
  Transcription in progress, waiting 10 seconds...
```

### Completed Transcription

```
============================================================
Transcription Result
============================================================
{
  "lattice": [
    {
      "onebest": "Hello, this is a test transcription.",
      "speaker": "0",
      "begin": 0,
      "end": 5000
    }
  ]
}
```

## Troubleshooting

### Common Issues

1. **Upload fails with authentication error**
   - Verify App ID and Secret Key are correct
   - Check that the signature generation is working correctly

2. **File not found error**
   - Verify the audio file path is correct
   - Check file permissions

3. **Transcription takes too long**
   - This is normal for long audio files
   - Use `--poll` option to wait for completion
   - Increase `--max-attempts` for very long files

4. **Duration estimation is inaccurate**
   - Provide `--duration` parameter with actual audio duration
   - This helps with better queue management

## Notes

- The script supports both one-time result checking and polling modes
- Duration can be estimated automatically, but providing actual duration is recommended
- The script follows the project's coding standards and error handling patterns
- All output messages are in English as per project requirements
- The script uses absolute paths when possible to avoid path-related issues

## References

- iFlytek API Documentation: https://www.xfyun.cn/doc/asr/ifasr_new/API.html
- iFlytek Open Platform: https://www.xfyun.cn/

