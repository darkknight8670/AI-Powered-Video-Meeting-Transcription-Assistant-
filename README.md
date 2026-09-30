# AI Video Meeting Transcription Assistant

Turn a YouTube recording or local audio/video file into a searchable meeting brief. The assistant transcribes the recording, generates a title and summary, extracts action items, decisions, and open questions, and lets you ask questions about the transcript through a retrieval-augmented chat interface.

## Features

- Accepts YouTube URLs and local audio/video files.
- Converts input media to mono, 16 kHz WAV audio and processes it in 10-minute chunks.
- Transcribes English with a local OpenAI Whisper model.
- Transcribes Hinglish and translates it to English with the Sarvam AI speech-to-text translation API.
- Generates a short meeting title and an LLM-based bullet summary.
- Extracts action items, owners, deadlines, key decisions, and unresolved questions.
- Builds a local Chroma vector store using `all-MiniLM-L6-v2` embeddings.
- Answers questions using only the processed meeting transcript.
- Provides both a Streamlit web interface and an interactive command-line workflow.

## How It Works

```text
YouTube URL or local media
	|
	v
Download/convert to WAV -> split into 10-minute chunks
	|
	v
Whisper (English) or Sarvam AI (Hinglish -> English)
	|
	v
Transcript -> title, summary, action items, decisions, questions
	|
	v
Chroma vector store -> transcript-grounded Q&A
```

## Requirements

- Python 3.10 or newer.
- FFmpeg installed and available on your `PATH`. `ffmpeg-python` is only a Python wrapper; it does not install the FFmpeg binary.
- A Mistral API key for title generation, summarization, extraction, and Q&A.
- A Sarvam API key only when using `hinglish` transcription.
- Enough disk space and memory for the selected Whisper model and Hugging Face embedding model. The default Whisper `small` model is downloaded on first use.

## Installation

Clone the repository and create a virtual environment:

```bash
git clone <repository-url>
cd AI-Powered-Video-Meeting-Transcription-Assistant-

python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r Requirements.txt
```

The current source imports two packages that are not yet listed in `Requirements.txt`. Install them explicitly after the requirements file:

```bash
python -m pip install langchain-text-splitters langchain-chroma
```

Install FFmpeg with your operating system's package manager. For Ubuntu/Debian:

```bash
sudo apt update
sudo apt install ffmpeg
```

For systems where `sudo` is unavailable, install FFmpeg through the platform's normal package manager and verify that `ffmpeg -version` works.

## Configuration

Create a `.env` file in the repository root:

```dotenv
MISTRAL_API_KEY=your_mistral_api_key

# Required only for the hinglish option
SARVAM_API_KEY=your_sarvam_api_key

# Optional Whisper model name; defaults to small
WHISPER_MODEL=small

# Optional Sarvam model name; defaults to saaras:v2.5
SARVAM_STT_MODEL=saaras:v2.5
```

Do not commit `.env` or API keys. The application loads environment variables with `python-dotenv`.

## Run the Streamlit App

The web interface is the recommended way to use the assistant:

```bash
streamlit run app.py
```

Open the local URL printed by Streamlit, then:

1. Enter a YouTube URL or a path to a local audio/video file in the sidebar.
2. Select `english` or `hinglish`.
3. Select **Analyse** and wait for the pipeline to finish.
4. Review the generated title, summary, transcript, action items, decisions, and open questions.
5. Ask follow-up questions in **Chat with your Meeting**.

The chat answers are generated from the current transcript's local vector store. A new analysis replaces the current session state.

## Run the Command-Line Workflow

Run the interactive CLI:

```bash
python main.py
```

You will be prompted for the source and language:

```text
Enter YouTube URL or local file path: /path/to/meeting.mp4
Language (english/hinglish): english
```

The CLI prints the title, summary, extracted information, and then starts an interactive Q&A loop. Enter `exit`, `quit`, or `q` to leave the chat.

The reusable pipeline function can also be imported:

```python
from main import run_pipeline

result = run_pipeline("/path/to/meeting.mp4", language="english")
print(result["summary"])
```

The returned dictionary contains `title`, `transcript`, `summary`, `action_items`, `key_decisions`, `open_questions`, and `rag_chain`.

## Supported Inputs

### YouTube

Pass a standard `http://` or `https://` URL. `yt-dlp` downloads the best available audio and FFmpeg converts it to WAV. The downloaded file is placed in the `downloades/` directory created by the application.

### Local files

Pass a local path to an audio or video file readable by FFmpeg, for example:

```text
/home/user/meetings/product-review.mp4
/home/user/meetings/standup.m4a
/home/user/meetings/interview.wav
```

Converted audio and 10-minute chunks are written next to the source file. These generated files can be large and are not automatically cleaned up.

## Project Structure

```text
.
├── app.py                    # Streamlit web application
├── main.py                   # Reusable pipeline and interactive CLI
├── Requirements.txt          # Python dependencies
├── core/
│   ├── transcriber.py        # Whisper and Sarvam transcription
│   ├── summarizer.py         # Title and summary generation
│   ├── extractor.py          # Action items, decisions, questions
│   ├── rag_engine.py         # Retrieval-augmented question answering
│   └── vector_store.py       # Chroma persistence and embeddings
├── utils/
│   └── audio_processor.py    # Download, conversion, and chunking
├── test.py                   # Manual pipeline experiment
└── vector_db/                # Generated Chroma persistence directory
```

## Processing Details

- English transcription uses Whisper locally and defaults to the `small` model. Set `WHISPER_MODEL` to another model supported by Whisper if needed, such as `base`, `medium`, or `large`.
- Hinglish transcription uses Sarvam's synchronous endpoint. Each chunk is split into 25-second WAV pieces because the endpoint accepts audio of no more than 30 seconds per request.
- Summarization splits transcripts into 3,000-character sections with 200-character overlap before combining partial summaries.
- Q&A splits transcripts into 500-character sections with 50-character overlap and retrieves the four most similar sections for each question.
- Mistral uses the `mistral-small-latest` model for all LLM tasks.

## Troubleshooting

### `ffmpeg` not found

Install the FFmpeg system binary and confirm it is available:

```bash
ffmpeg -version
```

### `MISTRAL_API_KEY` errors

Check that `.env` is in the project root, the variable name is exactly `MISTRAL_API_KEY`, and the virtual environment is running the same project directory.

### Hinglish transcription fails

Set `SARVAM_API_KEY` in `.env`. The key is required for every `hinglish` run; English runs use Whisper and do not call Sarvam.

### First run is slow

Whisper and the Hugging Face embedding model are downloaded and initialized lazily. Later runs can still take time because transcription and LLM calls are performed for each input.

### YouTube download fails

Confirm the URL is accessible, update `yt-dlp`, and check that the video is not private, age-restricted, or blocked in your region:

```bash
python -m pip install --upgrade yt-dlp
```

## Limitations

- There is no speaker diarization; transcript text is not attributed to individual speakers.
- Summaries, extracted items, and answers depend on transcription quality and LLM output and should be reviewed before being treated as authoritative.
- Processing is synchronous and can be slow for long recordings.
- The current vector store uses a fixed collection name and local `vector_db/` directory. Multiple meetings in the same directory can share or accumulate data unless the directory is cleared between sessions.
- Generated downloads, converted files, chunks, and the vector database are not cleaned up automatically.
- `test.py` contains a hard-coded example YouTube URL and is intended as a manual experiment, not an automated test suite.

## Privacy and API Usage

English audio is processed locally by Whisper, but the transcript is sent to Mistral for title generation, summarization, extraction, and Q&A. Hinglish audio is sent in short WAV pieces to Sarvam for transcription and translation. Review the applicable provider terms and avoid processing sensitive recordings unless your organization's policies permit it.

## License

No license file is currently included in this repository. Add a license before distributing the project publicly.