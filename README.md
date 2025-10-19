# 📋 TÀI LIỆU KỸ THUẬT CHI TIẾT - PHIÊN BẢN REVISED

## 🎯 ĐIỀU CHỈNH QUAN TRỌNG

### 1. **Async/Await thay vì Polling**
Sử dụng Python asyncio để xử lý concurrent, không cần poll status.

### 2. **Dịch vụ cụ thể thay vì API giả định**
- **YouTube Download**: `yt-dlp` (library Python)
- **Video Editing**: `FFmpeg` (local processing)
- **Alternative cloud**: Shotstack API, Creatomate API
- **AssemblyAI**: SDK chính thức với examples rõ ràng

---

## 📚 ARCHITECTURE REVISED

```
Input URLs
  ↓
[yt-dlp] Download Audio/Video (async)
  ↓
[AssemblyAI SDK] Transcription với word timestamps
  ↓
[Google Gemini] AI Highlights Extraction
  ↓
[FFmpeg] Video Processing Pipeline
  ├─ Extract Clips
  ├─ Generate Subtitles
  ├─ Overlay & Composite
  └─ Export Final Videos
  ↓
Output: TikTok-ready clips
```

---

## 🔧 MODULE 1: VIDEO/AUDIO DOWNLOAD

### Setup Dependencies
```bash
pip install yt-dlp
```

### Implementation với yt-dlp

```python
import yt_dlp
import os
from pathlib import Path

class YouTubeDownloader:
    """
    Download audio/video từ YouTube sử dụng yt-dlp
    """
    
    def __init__(self, output_dir: str = "downloads"):
        self.output_dir = Path(output_dir)
        self.output_dir.mkdir(parents=True, exist_ok=True)
    
    def download_audio(self, url: str, output_filename: str = None) -> str:
        """
        Download audio từ YouTube video
        
        Args:
            url: YouTube URL
            output_filename: Tên file output (optional)
        
        Returns:
            Path to downloaded audio file
        """
        if not output_filename:
            output_filename = "audio_%(id)s.mp3"
        
        output_path = self.output_dir / output_filename
        
        ydl_opts = {
            'format': 'bestaudio/best',
            'outtmpl': str(output_path),
            'postprocessors': [{
                'key': 'FFmpegExtractAudio',
                'preferredcodec': 'mp3',
                'preferredquality': '192',
            }],
            'quiet': False,
            'no_warnings': False,
            'extract_flat': False,
        }
        
        try:
            with yt_dlp.YoutubeDL(ydl_opts) as ydl:
                print(f"Downloading audio from: {url}")
                info = ydl.extract_info(url, download=True)
                
                # Get actual filename after download
                actual_filename = ydl.prepare_filename(info)
                # Replace extension with .mp3
                actual_filename = os.path.splitext(actual_filename)[0] + '.mp3'
                
                print(f"✅ Audio downloaded: {actual_filename}")
                return actual_filename
                
        except Exception as e:
            raise Exception(f"Failed to download audio: {e}")
    
    def download_video_segment(
        self, 
        url: str, 
        start_time: float, 
        end_time: float,
        output_filename: str = None
    ) -> str:
        """
        Download một segment cụ thể của video
        
        Args:
            url: YouTube URL
            start_time: Start time in seconds
            end_time: End time in seconds
            output_filename: Output filename
        
        Returns:
            Path to downloaded video segment
        """
        if not output_filename:
            output_filename = f"clip_{int(start_time)}_{int(end_time)}.mp4"
        
        output_path = self.output_dir / output_filename
        
        # Download full video first
        temp_file = self.output_dir / f"temp_{output_filename}"
        
        ydl_opts = {
            'format': 'bestvideo[height<=1920]+bestaudio/best',
            'outtmpl': str(temp_file),
            'merge_output_format': 'mp4',
            'quiet': False,
        }
        
        try:
            # Download full video
            with yt_dlp.YoutubeDL(ydl_opts) as ydl:
                print(f"Downloading video segment from {start_time}s to {end_time}s")
                info = ydl.extract_info(url, download=True)
                full_video = ydl.prepare_filename(info)
            
            # Extract segment using FFmpeg
            duration = end_time - start_time
            
            import subprocess
            cmd = [
                'ffmpeg',
                '-ss', str(start_time),
                '-i', full_video,
                '-t', str(duration),
                '-c', 'copy',  # Copy without re-encoding (fast)
                '-y',  # Overwrite output
                str(output_path)
            ]
            
            subprocess.run(cmd, check=True, capture_output=True)
            
            # Clean up temp file
            os.remove(full_video)
            
            print(f"✅ Video segment extracted: {output_path}")
            return str(output_path)
            
        except Exception as e:
            raise Exception(f"Failed to download video segment: {e}")
    
    def get_video_info(self, url: str) -> dict:
        """
        Lấy metadata của video
        """
        ydl_opts = {
            'quiet': True,
            'no_warnings': True,
        }
        
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            info = ydl.extract_info(url, download=False)
            
            return {
                'title': info.get('title'),
                'duration': info.get('duration'),  # seconds
                'uploader': info.get('uploader'),
                'upload_date': info.get('upload_date'),
                'view_count': info.get('view_count'),
            }

# Usage Example
downloader = YouTubeDownloader(output_dir="downloads")

# Download full podcast audio
audio_path = downloader.download_audio(
    "https://www.youtube.com/watch?v=example"
)

# Download specific clip
clip_path = downloader.download_video_segment(
    url="https://www.youtube.com/watch?v=example",
    start_time=120,  # 2 minutes
    end_time=180     # 3 minutes
)
```

### Async Version (Advanced)

```python
import asyncio
import aiohttp

class AsyncYouTubeDownloader:
    """
    Async version để download multiple videos đồng thời
    """
    
    async def download_multiple_clips(
        self,
        url: str,
        time_ranges: list  # [(start1, end1), (start2, end2), ...]
    ) -> list:
        """
        Download nhiều clips đồng thời
        """
        tasks = [
            self._download_clip_async(url, start, end)
            for start, end in time_ranges
        ]
        
        results = await asyncio.gather(*tasks)
        return results
    
    async def _download_clip_async(self, url: str, start: float, end: float):
        """
        Download single clip async
        """
        # Wrap sync download in executor
        loop = asyncio.get_event_loop()
        downloader = YouTubeDownloader()
        
        clip_path = await loop.run_in_executor(
            None,
            downloader.download_video_segment,
            url, start, end
        )
        
        return clip_path

# Usage
async def main():
    async_downloader = AsyncYouTubeDownloader()
    
    clips = await async_downloader.download_multiple_clips(
        url="https://youtube.com/watch?v=example",
        time_ranges=[
            (0, 30),
            (60, 90),
            (120, 150)
        ]
    )
    
    print(f"Downloaded {len(clips)} clips")

# Run
asyncio.run(main())
```

---

## 🔧 MODULE 2: ASSEMBLYAI TRANSCRIPTION (CHI TIẾT)

### Setup
```bash
pip install assemblyai
```

### AssemblyAI Flow Giải Thích

**AssemblyAI hoạt động như sau:**

1. **Upload**: Bạn upload file audio lên storage của AssemblyAI (họ host)
2. **Submit**: Tạo transcription job với audio URL vừa upload
3. **Wait**: AssemblyAI xử lý (tự động, không cần code gì)
4. **Retrieve**: Lấy kết quả với word-level timestamps (AssemblyAI đã xử lý sẵn)

**⚠️ QUAN TRỌNG**: AssemblyAI SDK có built-in async/await, không cần polling!

### Implementation với AssemblyAI SDK

```python
import assemblyai as aai
from typing import List, Dict

class PodcastTranscriber:
    """
    Transcribe audio với AssemblyAI SDK
    """
    
    def __init__(self, api_key: str):
        aai.settings.api_key = api_key
        self.transcriber = aai.Transcriber()
    
    def transcribe_audio(self, audio_file_path: str) -> Dict:
        """
        Transcribe audio file và trả về word-level timestamps
        
        AssemblyAI tự động:
        1. Upload file lên storage của họ
        2. Process transcription
        3. Trả về kết quả với timestamps
        
        Args:
            audio_file_path: Path to local audio file
        
        Returns:
            {
                "text": "full transcript text...",
                "words": [
                    {
                        "text": "Hello",
                        "start": 0.5,
                        "end": 0.8,
                        "confidence": 0.99
                    },
                    ...
                ]
            }
        """
        config = aai.TranscriptionConfig(
            # Enable word-level timestamps
            word_boost=["podcast", "AI", "technology"],  # Optional: boost specific words
            language_code="en"  # Hoặc "vi" cho tiếng Việt
        )
        
        print(f"🎙️  Starting transcription for: {audio_file_path}")
        print("📤 Uploading to AssemblyAI...")
        
        # SDK tự động upload và process
        transcript = self.transcriber.transcribe(
            audio_file_path,
            config=config
        )
        
        # Kiểm tra status
        if transcript.status == aai.TranscriptStatus.error:
            raise Exception(f"Transcription failed: {transcript.error}")
        
        print("✅ Transcription completed!")
        
        # Extract word-level data
        words_data = []
        
        if transcript.words:
            for word in transcript.words:
                words_data.append({
                    "text": word.text,
                    "start": word.start / 1000.0,  # Convert ms to seconds
                    "end": word.end / 1000.0,
                    "confidence": word.confidence
                })
        
        return {
            "text": transcript.text,
            "words": words_data,
            "duration": transcript.audio_duration / 1000.0  # Convert to seconds
        }
    
    def transcribe_from_url(self, audio_url: str) -> Dict:
        """
        Transcribe từ URL (không cần download về local)
        
        Useful nếu audio đã host ở đâu đó
        """
        config = aai.TranscriptionConfig()
        
        transcript = self.transcriber.transcribe(audio_url, config=config)
        
        if transcript.status == aai.TranscriptStatus.error:
            raise Exception(f"Transcription failed: {transcript.error}")
        
        words_data = [
            {
                "text": w.text,
                "start": w.start / 1000.0,
                "end": w.end / 1000.0,
                "confidence": w.confidence
            }
            for w in transcript.words
        ]
        
        return {
            "text": transcript.text,
            "words": words_data,
            "duration": transcript.audio_duration / 1000.0
        }

# Usage Example
transcriber = PodcastTranscriber(api_key="your_assemblyai_api_key")

# Transcribe local file
result = transcriber.transcribe_audio("downloads/podcast.mp3")

print(f"Full transcript: {result['text'][:100]}...")
print(f"Total words: {len(result['words'])}")
print(f"Duration: {result['duration']} seconds")

# Example: Print first 10 words with timestamps
for word in result['words'][:10]:
    print(f"{word['start']:.2f}s - {word['end']:.2f}s: {word['text']}")
```

### Async Version

```python
import asyncio

class AsyncPodcastTranscriber:
    """
    Async wrapper cho AssemblyAI
    """
    
    def __init__(self, api_key: str):
        self.api_key = api_key
    
    async def transcribe_audio_async(self, audio_file_path: str) -> Dict:
        """
        Transcribe async (chạy trong executor để không block)
        """
        loop = asyncio.get_event_loop()
        
        transcriber = PodcastTranscriber(self.api_key)
        
        # Run blocking transcribe in executor
        result = await loop.run_in_executor(
            None,
            transcriber.transcribe_audio,
            audio_file_path
        )
        
        return result
    
    async def transcribe_multiple(self, audio_files: List[str]) -> List[Dict]:
        """
        Transcribe nhiều files đồng thời
        """
        tasks = [
            self.transcribe_audio_async(audio_file)
            for audio_file in audio_files
        ]
        
        results = await asyncio.gather(*tasks)
        return results

# Usage
async def main():
    transcriber = AsyncPodcastTranscriber(api_key="your_key")
    
    results = await transcriber.transcribe_multiple([
        "podcast1.mp3",
        "podcast2.mp3"
    ])
    
    for i, result in enumerate(results):
        print(f"Podcast {i+1}: {len(result['words'])} words")

asyncio.run(main())
```

### Alternative: OpenAI Whisper (Local Processing)

**Nếu muốn xử lý local không dùng cloud:**

```python
import whisper
import torch

class LocalWhisperTranscriber:
    """
    Transcribe local bằng Whisper model
    """
    
    def __init__(self, model_size: str = "base"):
        """
        model_size: tiny, base, small, medium, large
        """
        print(f"Loading Whisper {model_size} model...")
        self.model = whisper.load_model(model_size)
    
    def transcribe_audio(self, audio_path: str) -> Dict:
        """
        Transcribe với word-level timestamps
        """
        result = self.model.transcribe(
            audio_path,
            word_timestamps=True,
            language="en"  # Hoặc None để auto-detect
        )
        
        # Extract word-level data
        words_data = []
        
        for segment in result['segments']:
            if 'words' in segment:
                for word in segment['words']:
                    words_data.append({
                        "text": word['word'].strip(),
                        "start": word['start'],
                        "end": word['end'],
                        "confidence": word.get('probability', 1.0)
                    })
        
        return {
            "text": result['text'],
            "words": words_data,
            "language": result.get('language', 'unknown')
        }

# Usage
transcriber = LocalWhisperTranscriber(model_size="base")
result = transcriber.transcribe_audio("podcast.mp3")
```

---

## 🔧 MODULE 3: AI HIGHLIGHTS EXTRACTION

### Implementation (giữ nguyên từ version trước)

```python
import google.generativeai as genai
import json

class HighlightsExtractor:
    """
    Extract viral moments từ transcript
    """
    
    PROMPT = """You are given a transcript array where each entry is an object with:
- **text**: transcript text
- **index**: word position
- **start**: start time in seconds
- **end**: end time in seconds

**Task**: Extract 3-5 viral highlights (30-120s each, max 600s total).

**Criteria for highlights:**
1. High emotional impact or surprising information
2. Standalone, self-contained moments
3. Clear beginning and end
4. No filler words

**Output ONLY a JSON array (no markdown, no explanations):**
[
  {
    "transcript": "exact words from highlight",
    "start_index": 10,
    "end_index": 45
  }
]
"""
    
    def __init__(self, api_key: str):
        genai.configure(api_key=api_key)
        self.model = genai.GenerativeModel("gemini-2.0-flash-exp")
    
    def extract_highlights(
        self,
        words: List[Dict],
        min_duration: float = 30,
        max_duration: float = 120,
        max_total_duration: float = 600
    ) -> List[Dict]:
        """
        Extract highlights từ word array
        """
        # Prepare structured data
        structured_words = [
            {
                "text": w["text"],
                "index": idx,
                "start": round(w["start"], 2),
                "end": round(w["end"], 2)
            }
            for idx, w in enumerate(words)
        ]
        
        transcript_json = json.dumps(structured_words)
        
        # Generate
        full_prompt = f"{self.PROMPT}\n\nTranscript:\n{transcript_json}"
        
        response = self.model.generate_content(
            full_prompt,
            generation_config={
                "temperature": 0.3,
                "top_p": 0.8,
                "max_output_tokens": 4096
            }
        )
        
        # Parse response
        raw_text = response.text.strip()
        
        # Remove markdown if present
        if "```" in raw_text:
            raw_text = raw_text.split("```")[1]
            if raw_text.startswith("json"):
                raw_text = raw_text[4:]
        
        highlights = json.loads(raw_text)
        
        # Validate
        validated = self._validate_highlights(
            highlights,
            words,
            min_duration,
            max_duration,
            max_total_duration
        )
        
        return validated
    
    def _validate_highlights(
        self,
        highlights: List[Dict],
        words: List[Dict],
        min_dur: float,
        max_dur: float,
        max_total: float
    ) -> List[Dict]:
        """
        Validate và filter highlights
        """
        valid = []
        total_duration = 0
        
        for h in highlights:
            if not all(k in h for k in ["start_index", "end_index", "transcript"]):
                continue
            
            start_idx = h["start_index"]
            end_idx = h["end_index"]
            
            # Get actual times
            start_time = words[start_idx]["start"]
            end_time = words[end_idx]["end"]
            duration = end_time - start_time
            
            # Validate duration
            if duration < min_dur or duration > max_dur:
                continue
            
            # Check total
            if total_duration + duration > max_total:
                break
            
            valid.append({
                **h,
                "start_time": start_time,
                "end_time": end_time,
                "duration": duration
            })
            
            total_duration += duration
        
        return valid

# Usage
extractor = HighlightsExtractor(api_key="your_gemini_key")

highlights = extractor.extract_highlights(
    words=transcription_result["words"]
)

for i, h in enumerate(highlights):
    print(f"\nHighlight {i+1}:")
    print(f"  Time: {h['start_time']:.1f}s - {h['end_time']:.1f}s")
    print(f"  Duration: {h['duration']:.1f}s")
    print(f"  Text: {h['transcript'][:100]}...")
```

---

## 🔧 MODULE 4: VIDEO EDITING VỚI FFMPEG

### Setup FFmpeg
```bash
# Ubuntu/Debian
sudo apt-get install ffmpeg

# macOS
brew install ffmpeg

# Windows: Download từ https://ffmpeg.org/download.html
```

### Video Editor Class

```python
import subprocess
import os
from pathlib import Path
from typing import List, Dict

class VideoEditor:
    """
    Video editing pipeline với FFmpeg
    """
    
    def __init__(self, output_dir: str = "output"):
        self.output_dir = Path(output_dir)
        self.output_dir.mkdir(parents=True, exist_ok=True)
    
    def extract_clip(
        self,
        video_path: str,
        start_time: float,
        end_time: float,
        output_filename: str = None
    ) -> str:
        """
        Extract clip từ video
        """
        if not output_filename:
            output_filename = f"clip_{int(start_time)}_{int(end_time)}.mp4"
        
        output_path = self.output_dir / output_filename
        
        duration = end_time - start_time
        
        cmd = [
            'ffmpeg',
            '-ss', str(start_time),
            '-i', video_path,
            '-t', str(duration),
            '-c:v', 'libx264',
            '-c:a', 'aac',
            '-y',
            str(output_path)
        ]
        
        subprocess.run(cmd, check=True, capture_output=True)
        
        return str(output_path)
    
    def create_subtitle_file(
        self,
        words_chunks: List[Dict],
        output_path: str
    ):
        """
        Tạo file .srt cho subtitles
        
        words_chunks format:
        [
            {
                "words": "Hello world",
                "start": 0.5,
                "end": 1.2
            },
            ...
        ]
        """
        srt_content = []
        
        for idx, chunk in enumerate(words_chunks, 1):
            start_ts = self._format_timestamp(chunk["start"])
            end_ts = self._format_timestamp(chunk["end"])
            
            srt_content.extend([
                str(idx),
                f"{start_ts} --> {end_ts}",
                chunk["words"],
                ""
            ])
        
        with open(output_path, 'w', encoding='utf-8') as f:
            f.write('\n'.join(srt_content))
    
    def _format_timestamp(self, seconds: float) -> str:
        """
        Convert seconds to SRT timestamp format: HH:MM:SS,mmm
        """
        hours = int(seconds // 3600)
        minutes = int((seconds % 3600) // 60)
        secs = int(seconds % 60)
        millis = int((seconds % 1) * 1000)
        
        return f"{hours:02d}:{minutes:02d}:{secs:02d},{millis:03d}"
    
    def overlay_video_with_subtitles(
        self,
        main_clip_path: str,
        background_path: str,
        subtitle_chunks: List[Dict],
        output_filename: str = None
    ) -> str:
        """
        Composite video:
        - Background ở dưới (9:16 aspect ratio)
        - Main clip ở giữa (scaled)
        - Subtitles ở dưới với styling
        
        Returns: Path to final video
        """
        if not output_filename:
            output_filename = f"final_{os.path.basename(main_clip_path)}"
        
        output_path = self.output_dir / output_filename
        
        # Create subtitle file
        subtitle_file = self.output_dir / f"temp_subs_{os.path.basename(main_clip_path)}.srt"
        self.create_subtitle_file(subtitle_chunks, str(subtitle_file))
        
        # FFmpeg filter complex
        filter_complex = (
            # Scale background to 9:16 (1080x1920)
            "[1:v]scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920[bg];"
            
            # Scale main clip to fit width, maintain aspect ratio
            "[0:v]scale=1080:-1[clip];"
            
            # Overlay main clip on background (centered, slightly above center)
            "[bg][clip]overlay=(W-w)/2:(H-h)/2-150[v];"
            
            # Add subtitles with styling
            f"[v]subtitles={subtitle_file}:force_style='"
            "Fontname=Arial,"
            "Fontsize=28,"
            "Bold=1,"
            "PrimaryColour=&HFFFFFF,"
            "OutlineColour=&H000000,"
            "Outline=3,"
            "Shadow=2,"
            "Alignment=2,"  # Bottom center
            "MarginV=80"
            "'[out]"
        )
        
        cmd = [
            'ffmpeg',
            '-i', main_clip_path,      # Input 0: main clip
            '-i', background_path,      # Input 1: background
            '-filter_complex', filter_complex,
            '-map', '[out]',
            '-map', '0:a',  # Audio from main clip
            '-c:v', 'libx264',
            '-preset', 'medium',
            '-crf', '23',
            '-c:a', 'aac',
            '-b:a', '192k',
            '-shortest',  # End when shortest input ends
            '-y',
            str(output_path)
        ]
        
        print(f"🎬 Rendering final video...")
        subprocess.run(cmd, check=True, capture_output=False)
        
        # Clean up subtitle file
        os.remove(subtitle_file)
        
        print(f"✅ Video ready: {output_path}")
        return str(output_path)
    
    def extract_audio(self, video_path: str, output_path: str = None) -> str:
        """
        Extract audio từ video
        """
        if not output_path:
            output_path = str(self.output_dir / f"{Path(video_path).stem}.mp3")
        
        cmd = [
            'ffmpeg',
            '-i', video_path,
            '-vn',  # No video
            '-acodec', 'libmp3lame',
            '-q:a', '2',  # High quality
            '-y',
            output_path
        ]
        
        subprocess.run(cmd, check=True, capture_output=True)
        
        return output_path
    
    def get_video_duration(self, video_path: str) -> float:
        """
        Get video duration in seconds
        """
        cmd = [
            'ffprobe',
            '-v', 'error',
            '-show_entries', 'format=duration',
            '-of', 'default=noprint_wrappers=1:nokey=1',
            video_path
        ]
        
        result = subprocess.run(cmd, capture_output=True, text=True)
        return float(result.stdout.strip())

# Usage Example
editor = VideoEditor(output_dir="output")

# Extract clip
clip = editor.extract_clip(
    video_path="full_podcast_video.mp4",
    start_time=120,
    end_time=180
)

# Prepare subtitle chunks
subtitle_chunks = [
    {"words": "Hello everyone", "start": 0, "end": 1.5},
    {"words": "Welcome to", "start": 1.5, "end": 2.5},
    {"words": "my podcast", "start": 2.5, "end": 3.5},
]

# Create final video with subtitles
final_video = editor.overlay_video_with_subtitles(
    main_clip_path=clip,
    background_path="minecraft_gameplay.mp4",
    subtitle_chunks=subtitle_chunks
)
```

### Alternative: Cloud Video Editing APIs

**Option 1: Shotstack API**

```python
import requests
import time

class ShotstackVideoEditor:
    """
    Cloud video editing với Shotstack API
    """
    
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.shotstack.io/v1"
    
    def create_tiktok_video(
        self,
        main_clip_url: str,
        background_url: str,
        subtitle_chunks: List[Dict]
    ) -> str:
        """
        Tạo TikTok video với Shotstack
        """
        # Build timeline
        timeline = {
            "tracks": [
                # Track 0: Background
                {
                    "clips": [
                        {
                            "asset": {
                                "type": "video",
                                "src": background_url
                            },
                            "start": 0,
                            "length": "auto",
                            "fit": "cover",
                            "scale": 1
                        }
                    ]
                },
                # Track 1: Main clip
                {
                    "clips": [
                        {
                            "asset": {
                                "type": "video",
                                "src": main_clip_url
                            },
                            "start": 0,
                            "length": "auto",
                            "position": "center",
                            "offset": {
                                "y": -0.1  # Slightly above center
                            },
                            "scale": 0.8
                        }
                    ]
                },
                # Track 2: Subtitles
                {
                    "clips": [
                        {
                            "asset": {
                                "type": "title",
                                "text": chunk["words"],
                                "style": "future",
                                "color": "#ffffff",
                                "size": "large"
                            },
                            "start": chunk["start"],
                            "length": chunk["end"] - chunk["start"],
                            "position": "bottom"
                        }
                        for chunk in subtitle_chunks
                    ]
                }
            ]
        }
        
        # Submit render
        payload = {
            "timeline": timeline,
            "output": {
                "format": "mp4",
                "resolution": "hd",
                "aspectRatio": "9:16"
            }
        }
        
        headers = {
            "x-api-key": self.api_key,
            "Content-Type": "application/json"
        }
        
        response = requests.post(
            f"{self.base_url}/render",
            json=payload,
            headers=headers
        )
        
        render_id = response.json()["response"]["id"]
        
        # Wait for completion
        return self._wait_for_render(render_id)
    
    def _wait_for_render(self, render_id: str) -> str:
        """
        Wait for render to complete
        """
        headers = {"x-api-key": self.api_key}
        
        while True:
            response = requests.get(
                f"{self.base_url}/render/{render_id}",
                headers=headers
            )
            
            data = response.json()["response"]
            status = data["status"]
            
            if status == "done":
                return data["url"]
            elif status == "failed":
                raise Exception("Render failed")
            
            time.sleep(5)

# Usage
editor = ShotstackVideoEditor(api_key="your_shotstack_key")
video_url = editor.create_tiktok_video(
    main_clip_url="https://example.com/clip.mp4",
    background_url="https://example.com/background.mp4",
    subtitle_chunks=subtitle_chunks
)
```

---

## 🔧 MODULE 5: COMPLETE PIPELINE

### Orchestrator với Async

```python
import asyncio
from typing import List, Dict

class PodcastToTikTokPipeline:
    """
    Complete async pipeline
    """
    
    def __init__(self, config: Dict):
        self.config = config
        
        # Initialize components
        self.downloader = YouTubeDownloader()
        self.transcriber = AsyncPodcastTranscriber(
            config["assemblyai_api_key"]
        )
        self.extractor = HighlightsExtractor(
            config["gemini_api_key"]
        )
        self.editor = VideoEditor()
    
    async def process_podcast(
        self,
        podcast_url: str,
        background_url: str
    ) -> List[Dict]:
        """
        Complete pipeline
        """
        print("=" * 60)
        print("🚀 PODCAST TO TIKTOK PIPELINE")
        print("=" * 60)
        
        # Step 1: Download audio
        print("\n📥 Step 1: Downloading podcast audio...")
        loop = asyncio.get_event_loop()
        audio_path = await loop.run_in_executor(
            None,
            self.downloader.download_audio,
            podcast_url
        )
        
        # Step 2: Transcribe
        print("\n🎙️  Step 2: Transcribing audio...")
        transcription = await self.transcriber.transcribe_audio_async(audio_path)
        print(f"   ✅ Transcribed {len(transcription['words'])} words")
        
        # Step 3: Extract highlights
        print("\n🎯 Step 3: Extracting viral highlights...")
        highlights = await loop.run_in_executor(
            None,
            self.extractor.extract_highlights,
            transcription['words']
        )
        print(f"   ✅ Found {len(highlights)} highlights")
        
        # Step 4: Process each highlight
        print("\n🎬 Step 4: Generating clips...")
        
        clips = []
        for i, highlight in enumerate(highlights):
            print(f"\n   📹 Processing clip {i+1}/{len(highlights)}...")
            
            clip_data = await self._process_highlight(
                podcast_url,
                background_url,
                highlight,
                transcription['words']
            )
            
            clips.append(clip_data)
            print(f"   ✅ Clip {i+1} ready: {clip_data['output_path']}")
        
        print("\n" + "=" * 60)
        print(f"✅ PIPELINE COMPLETE: {len(clips)} clips generated")
        print("=" * 60)
        
        return clips
    
    async def _process_highlight(
        self,
        podcast_url: str,
        background_url: str,
        highlight: Dict,
        all_words: List[Dict]
    ) -> Dict:
        """
        Process single highlight into TikTok clip
        """
        loop = asyncio.get_event_loop()
        
        # Download main clip
        main_clip = await loop.run_in_executor(
            None,
            self.downloader.download_video_segment,
            podcast_url,
            highlight['start_time'],
            highlight['end_time']
        )
        
        # Download background
        duration = highlight['duration']
        background_clip = await loop.run_in_executor(
            None,
            self.downloader.download_video_segment,
            background_url,
            3,  # Start from 3s to skip intro
            3 + duration
        )
        
        # Extract audio from main clip
        audio_path = self.editor.extract_audio(main_clip)
        
        # Transcribe clip audio for precise subtitles
        clip_transcription = await self.transcriber.transcribe_audio_async(audio_path)
        
        # Group words into subtitle chunks (3 words per chunk)
        subtitle_chunks = self._create_subtitle_chunks(
            clip_transcription['words']
        )
        
        # Composite final video
        final_video = await loop.run_in_executor(
            None,
            self.editor.overlay_video_with_subtitles,
            main_clip,
            background_clip,
            subtitle_chunks
        )
        
        # Generate title
        title = await self._generate_title(highlight['transcript'])
        
        return {
            "output_path": final_video,
            "title": title,
            "transcript": highlight['transcript'],
            "duration": duration,
            "start_time": highlight['start_time'],
            "end_time": highlight['end_time']
        }
    
    def _create_subtitle_chunks(
        self,
        words: List[Dict],
        words_per_chunk: int = 3
    ) -> List[Dict]:
        """
        Group words into subtitle chunks
        """
        chunks = []
        
        for i in range(0, len(words), words_per_chunk):
            chunk_words = words[i:i + words_per_chunk]
            
            text = ' '.join([
                w['text'].capitalize()
                for w in chunk_words
            ])
            
            chunks.append({
                "words": text,
                "start": chunk_words[0]['start'],
                "end": chunk_words[-1]['end']
            })
        
        return chunks
    
    async def _generate_title(self, transcript: str) -> str:
        """
        Generate viral title
        """
        loop = asyncio.get_event_loop()
        
        prompt = f"""Create a viral TikTok title (max 60 chars) for this clip:

"{transcript[:200]}..."

Requirements:
- Attention-grabbing
- Include 1-2 hashtags
- Create curiosity

Output only the title, nothing else."""
        
        genai.configure(api_key=self.config['gemini_api_key'])
        model = genai.GenerativeModel("gemini-2.0-flash-exp")
        
        response = await loop.run_in_executor(
            None,
            model.generate_content,
            prompt
        )
        
        title = response.text.strip()
        
        # Ensure < 60 chars
        if len(title) > 60:
            title = title[:57] + "..."
        
        return title

# Usage
async def main():
    config = {
        "assemblyai_api_key": "your_key",
        "gemini_api_key": "your_key",
    }
    
    pipeline = PodcastToTikTokPipeline(config)
    
    clips = await pipeline.process_podcast(
        podcast_url="https://youtube.com/watch?v=example",
        background_url="https://youtube.com/watch?v=minecraft"
    )
    
    # Save metadata
    import json
    with open("clips_metadata.json", "w") as f:
        json.dump(clips, f, indent=2)
    
    print("\n📊 Generated clips:")
    for clip in clips:
        print(f"\n  {clip['title']}")
        print(f"  Duration: {clip['duration']:.1f}s")
        print(f"  File: {clip['output_path']}")

# Run
if __name__ == "__main__":
    asyncio.run(main())
```

---

## 📦 COMPLETE PROJECT STRUCTURE

```
podcast-to-tiktok/
│
├── requirements.txt
├── .env
├── config.py
├── main.py
│
├── modules/
│   ├── __init__.py
│   ├── downloader.py      # YouTube download
│   ├── transcriber.py     # AssemblyAI integration
│   ├── extractor.py       # Highlights extraction
│   ├── editor.py          # Video editing
│   └── pipeline.py        # Main orchestrator
│
├── downloads/             # Temporary downloads
├── output/                # Final clips
└── cache/                 # Transcription cache
```

**requirements.txt:**
```
yt-dlp>=2023.10.13
assemblyai>=0.17.0
google-generativeai>=0.3.0
python-dotenv>=1.0.0
aiohttp>=3.9.0
tqdm>=4.66.0
```

**.env:**
```
ASSEMBLYAI_API_KEY=your_assemblyai_key
GEMINI_API_KEY=your_gemini_key
```

**main.py:**
```python
import asyncio
import os
from dotenv import load_dotenv
from modules.pipeline import PodcastToTikTokPipeline

load_dotenv()

async def main():
    config = {
        "assemblyai_api_key": os.getenv("ASSEMBLYAI_API_KEY"),
        "gemini_api_key": os.getenv("GEMINI_API_KEY"),
    }
    
    pipeline = PodcastToTikTokPipeline(config)
    
    clips = await pipeline.process_podcast(
        podcast_url=input("Podcast URL: "),
        background_url=input("Background video URL: ")
    )
    
    print(f"\n✅ Generated {len(clips)} clips!")

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 🎯 KEY DIFFERENCES VỚI N8N

| Aspect | n8n | Custom Code |
|--------|-----|-------------|
| **Download** | 3-stage polling | Direct download, no polling |
| **Transcription** | Manual polling | SDK handles automatically |
| **Processing** | Sequential nodes | Async parallel processing |
| **Video Editing** | External API | FFmpeg local (hoặc cloud API) |
| **Control** | Limited | Full control |

---

Bây giờ đã rõ ràng hơn chưa? Có phần nào cần giải thích thêm không? 🚀