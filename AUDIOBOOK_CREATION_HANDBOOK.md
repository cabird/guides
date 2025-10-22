# Audiobook Creation from Video Files: A Comprehensive Handbook

## Overview

This handbook provides detailed instructions for converting video files into a professional M4B audiobook format with chapters and metadata. The process involves downloading video segments, combining them, extracting audio, and packaging everything into an audiobook file.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Phase 1: Video Acquisition](#phase-1-video-acquisition)
3. [Phase 2: Video Processing](#phase-2-video-processing)
4. [Phase 3: Audio Extraction](#phase-3-audio-extraction)
5. [Phase 4: Audiobook Creation](#phase-4-audiobook-creation)
6. [Phase 5: Chapter Enhancement](#phase-5-chapter-enhancement)
7. [Metadata Configuration](#metadata-configuration)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Tools

Install these command-line tools before beginning:

```bash
# FFmpeg - for video/audio processing
sudo apt install ffmpeg          # Ubuntu/Debian
brew install ffmpeg              # macOS

# FFprobe - usually comes with FFmpeg
# Used for inspecting media file metadata

# MP4Box (from GPAC) - for advanced MP4 manipulation
sudo apt install gpac            # Ubuntu/Debian
brew install gpac                # macOS

# mp4chaps (from mp4v2) - for chapter management
sudo apt install mp4v2-utils     # Ubuntu/Debian
brew install mp4v2               # macOS

# AtomicParsley - for audiobook-specific metadata
sudo apt install atomicparsley   # Ubuntu/Debian
brew install atomicparsley       # macOS
```

---

## Phase 1: Video Acquisition

### Understanding HLS Streaming

Most online video courses use HLS (HTTP Live Streaming), which splits videos into:
- **M3U8 playlist files**: Text files listing video segments
- **TS (Transport Stream) segments**: Small video chunks (typically 2-10 seconds each)

### Step 1.1: Capture Network Traffic

When accessing your video course, capture network traffic using browser developer tools:

1. Open browser DevTools (F12)
2. Go to Network tab
3. Navigate to your video course
4. Export as HAR (HTTP Archive) file
5. Save HAR files to `./hars/` directory

**What to capture**: Look for M3U8 playlist requests in the network log. These usually have patterns like:
- `https://domain.com/path/index.m3u8`
- Files with `.m3u8` extension
- Content-type: `application/x-mpegURL` or `application/vnd.apple.mpegurl`

### Step 1.2: Extract M3U8 URLs from HAR Files

M3U8 files contain playlist information. You need to extract these from HAR files:

```bash
# Example HAR structure - what you're looking for:
# Request URL: https://example.com/videos/1_1_introduction/index.m3u8
# Response contains: #EXTM3U and stream information
```

The M3U8 file typically contains:
```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
https://example.com/videos/1_1_intro/360p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2000000,RESOLUTION=1280x720
https://example.com/videos/1_1_intro/720p/index.m3u8
```

**Key Points**:
- The first URL after `#EXT-X-STREAM-INF:` is usually the specific quality stream
- Choose the quality you want (360p is fine for audio extraction)
- The nested M3U8 file lists actual video segments

### Step 1.3: Download M3U8 Playlist Files

For each video, create a directory and download its M3U8 file:

```bash
# Create directory structure
mkdir -p videos/video_name

# Download the M3U8 playlist
curl -o videos/video_name/index.m3u8 "https://example.com/path/to/video.m3u8"

# Example with custom headers if needed
curl -H "User-Agent: Mozilla/5.0" \
     -H "Referer: https://course-site.com" \
     -o videos/video_name/index.m3u8 \
     "https://example.com/path/to/video.m3u8"
```

**Options explained**:
- `-o <file>`: Output filename
- `-H "Header: Value"`: Add custom HTTP headers (may be needed for authentication)
- Use quotes around URLs with special characters

### Step 1.4: Parse and Download TS Segments

The downloaded M3U8 file contains references to TS segments:

```bash
# View the M3U8 content
cat videos/video_name/index.m3u8

# Example content:
# #EXTM3U
# #EXT-X-TARGETDURATION:10
# #EXT-X-VERSION:3
# #EXTINF:10.0,
# seg-1-v1-a1.ts
# #EXTINF:10.0,
# seg-2-v1-a1.ts
```

Download each segment:

```bash
# Create segments directory
mkdir -p videos/video_name/segments

# Download each segment
# Extract base URL from M3U8 URL
BASE_URL="https://example.com/path/to/video"

# For each segment in the M3U8:
curl -o videos/video_name/segments/seg-1-v1-a1.ts "${BASE_URL}/seg-1-v1-a1.ts"
curl -o videos/video_name/segments/seg-2-v1-a1.ts "${BASE_URL}/seg-2-v1-a1.ts"
# ... repeat for all segments

# Or use a loop for all segments:
while IFS= read -r line; do
    if [[ $line == *.ts ]]; then
        filename=$(basename "$line")
        curl -o "videos/video_name/segments/$filename" "${BASE_URL}/$line"
        # Throttle to avoid overwhelming server
        sleep 2
    fi
done < videos/video_name/index.m3u8
```

**Important considerations**:
- **Throttling**: Add delays (2-10 seconds) between requests to be respectful to servers
- **Resume capability**: Use `curl -C -` to resume interrupted downloads
- **Authentication**: Some streams require cookies or tokens - capture these from HAR file
- **Segment naming**: Usually follows pattern like `seg-N-v1-a1.ts` where N is the sequence number

---

## Phase 2: Video Processing

### Step 2.1: Combine TS Segments into Complete Video

Once all segments are downloaded, combine them into a single MP4 file:

#### Method 1: FFmpeg Concatenation (Recommended)

```bash
# Create a file list for FFmpeg
# This tells FFmpeg which files to combine and in what order
cat > videos/video_name/concat.txt << EOF
file 'segments/seg-1-v1-a1.ts'
file 'segments/seg-2-v1-a1.ts'
file 'segments/seg-3-v1-a1.ts'
# ... list all segments in order
EOF

# Run FFmpeg to concatenate
ffmpeg -f concat -safe 0 -i videos/video_name/concat.txt \
       -c copy \
       videos/video_name/video_name.mp4
```

**FFmpeg options explained**:

- `-f concat`: Use the concat demuxer (file concatenation format)
- `-safe 0`: Allow absolute file paths in concat file
  - **Alternative**: Use relative paths and set `-safe 1` for stricter security
  - **Use `-safe 0` when**: Paths contain special characters or absolute paths needed
  - **Use `-safe 1` when**: All paths are relative and you want extra validation

- `-i concat.txt`: Input file containing list of segments
- `-c copy`: Copy streams without re-encoding
  - **Pros**: Very fast (no re-encoding), preserves original quality
  - **Cons**: Requires all segments to have identical codecs/parameters
  - **Alternative**: `-c:v libx264 -c:a aac` to re-encode (slower but fixes compatibility issues)

- Output: `videos/video_name/video_name.mp4`

#### Method 2: Direct TS Concatenation (Simpler but less reliable)

```bash
# TS files can sometimes be concatenated directly
cat videos/video_name/segments/seg-*.ts > videos/video_name/combined.ts

# Then convert to MP4
ffmpeg -i videos/video_name/combined.ts \
       -c copy \
       videos/video_name/video_name.mp4
```

**When to use**:
- Use Method 1 (concat demuxer) for most cases - more reliable
- Use Method 2 only if segments are simple and codec-compatible

#### Auto-generating the Concat File

```bash
# Generate concat file automatically from segments directory
cd videos/video_name
ls segments/seg-*-v1-a1.ts | sort -V | \
  awk '{print "file '\''" $0 "'\''"}' > concat.txt

# Then run ffmpeg as shown above
```

**Sorting considerations**:
- `sort -V`: Version sort (seg-1, seg-2, seg-10 sorts correctly)
- Without `-V`: Lexical sort would give seg-1, seg-10, seg-2 (wrong order)

### Step 2.2: Verify Video Integrity

After combining, verify the output:

```bash
# Check video duration and properties
ffprobe -v error \
        -show_entries format=duration,size,bit_rate \
        -show_entries stream=codec_name,codec_type \
        -of default=noprint_wrappers=1 \
        videos/video_name/video_name.mp4

# Play the video to verify
ffplay videos/video_name/video_name.mp4  # FFmpeg's built-in player
# or
vlc videos/video_name/video_name.mp4     # VLC media player
```

**What to check**:
- Duration matches expected video length
- No corruption or stuttering
- Audio and video are synchronized

---

## Phase 3: Audio Extraction

### Step 3.1: Extract Audio from Video

Convert video to audio-only format:

```bash
ffmpeg -i videos/video_name/video_name.mp4 \
       -vn \
       -acodec mp3 \
       -ab 192k \
       videos/video_name/video_name.mp3
```

**FFmpeg options explained**:

- `-i input.mp4`: Input video file
- `-vn`: Disable video recording (audio only)
- `-acodec mp3`: Audio codec to use
  - **Alternative codecs**:
    - `libmp3lame`: Explicit MP3 encoder (same as `mp3` usually)
    - `aac`: Better quality at same bitrate, more compatible
    - `libopus`: Best quality/size ratio for modern players
    - `libvorbis`: Open source, good quality
    - `copy`: Copy audio without re-encoding (if already in desired format)

- `-ab 192k` or `-b:a 192k`: Audio bitrate
  - **Bitrate guidelines**:
    - `64k`: Minimum for speech-only content (podcasts)
    - `92k`: Good balance for audiobooks (recommended for final M4B)
    - `128k`: Standard quality music/speech
    - `192k`: High quality, used for intermediate processing
    - `256k`: Very high quality, usually overkill for speech
    - `320k`: Maximum for MP3

  - **Pros/cons of different bitrates**:
    - **Lower (64-92k)**: Smaller file size, acceptable for speech
    - **Higher (192-256k)**: Better quality, larger files, preserves audio details

- Output: `videos/video_name/video_name.mp3`

### Step 3.2: Alternative Audio Formats

```bash
# Extract as AAC (better quality than MP3 at same bitrate)
ffmpeg -i videos/video_name/video_name.mp4 \
       -vn \
       -acodec aac \
       -b:a 128k \
       videos/video_name/video_name.m4a

# Extract as high-quality WAV (lossless, for further processing)
ffmpeg -i videos/video_name/video_name.mp4 \
       -vn \
       -acodec pcm_s16le \
       videos/video_name/video_name.wav

# Copy audio stream without re-encoding (preserves original)
ffmpeg -i videos/video_name/video_name.mp4 \
       -vn \
       -acodec copy \
       videos/video_name/video_name.aac  # or .m4a depending on original codec
```

**When to use each format**:
- **MP3**: Universal compatibility, good for intermediate files
- **AAC/M4A**: Better quality than MP3, native format for M4B audiobooks
- **WAV**: Lossless, use only for editing/processing, then convert to compressed format
- **Copy**: Fastest, no quality loss, but format limited by source codec

### Step 3.3: Audio Quality Adjustments

```bash
# Mono conversion (reduces file size, suitable for single speaker)
ffmpeg -i input.mp4 -vn -acodec mp3 -ab 128k -ac 1 output.mp3

# Stereo (keep both channels)
ffmpeg -i input.mp4 -vn -acodec mp3 -ab 192k -ac 2 output.mp3

# Normalize audio volume
ffmpeg -i input.mp4 -vn -acodec mp3 -ab 192k -af loudnorm output.mp3

# Apply audio filters
ffmpeg -i input.mp4 -vn -acodec mp3 -ab 192k \
       -af "highpass=f=100, lowpass=f=3000, loudnorm" \
       output.mp3
```

**Audio options**:
- `-ac 1`: Mono (1 channel)
  - **Use when**: Single speaker, lecture content, saves 50% space
- `-ac 2`: Stereo (2 channels)
  - **Use when**: Music, spatial audio important

- `-af loudnorm`: Loudness normalization filter
  - **Purpose**: Makes all audio files similar volume level
  - **Important for**: Audiobooks with multiple chapters from different sources
  - **Alternative**: `-af volume=2.0` (simple volume multiplication)

- **Audio filters** (`-af`):
  - `highpass=f=100`: Remove low-frequency rumble (<100Hz)
  - `lowpass=f=3000`: Remove high-frequency noise (>3000Hz)
  - Combine filters with commas: `-af "filter1,filter2,filter3"`

### Step 3.4: Batch Process All Videos

```bash
# Process all videos in the videos directory
for video_dir in videos/*/; do
    name=$(basename "$video_dir")
    video_file="$video_dir/${name}.mp4"
    audio_file="$video_dir/${name}.mp3"

    if [ -f "$video_file" ] && [ ! -f "$audio_file" ]; then
        echo "Processing: $name"
        ffmpeg -i "$video_file" -vn -acodec mp3 -ab 192k "$audio_file"
    fi
done
```

---

## Phase 4: Audiobook Creation

### Step 4.1: Prepare Chapter Information

Create a metadata file listing all chapters and their order:

**File: `audiobook_metadata.json`**

```json
{
  "audiobook": {
    "title": "Your Course Title",
    "author": "Instructor Name",
    "year": "2024",
    "genre": "Audiobook",
    "description": "Description of the course content",
    "language": "English"
  },
  "audio_settings": {
    "bitrate": "92k",
    "codec": "aac",
    "channels": "mono"
  },
  "chapter_name_overrides": {
    "intro_video": "Introduction",
    "1_1_lesson": "1.1 First Lesson Title",
    "1_2_lesson": "1.2 Second Lesson Title"
  }
}
```

**Field explanations**:
- `bitrate`: "92k" is optimal for speech audiobooks (balance of quality/size)
  - "64k": Minimum acceptable
  - "128k": Higher quality, larger file
- `codec`: "aac" is standard for M4B audiobooks
- `channels`: "mono" for single speaker, "stereo" for music/multi-source

### Step 4.2: Create Concatenation List

List all audio files in the desired order:

```bash
# Create file list for FFmpeg
# Order matters - this becomes your audiobook chapter sequence
cat > videos/audio_concat.txt << EOF
file 'mbt_introduction/mbt_introduction.mp3'
file '1_1_1_ttar/1_1_1_ttar.mp3'
file '1_1_2_trauma/1_1_2_trauma.mp3'
# ... list all audio files in chapter order
EOF
```

**Important**:
- Use absolute paths OR paths relative to concat file location
- Order determines audiobook chapter sequence
- Verify filenames match exactly (case-sensitive)

### Step 4.3: Generate Chapter Timestamps

You need chapter timestamps for the final audiobook. Calculate these by summing durations:

```bash
# Get duration of each audio file
ffprobe -v error -show_entries format=duration \
        -of default=noprint_wrappers=1:nokey=1 \
        video_name/video_name.mp3
```

Create chapters file:

**File: `videos/chapters.txt`**

```
00:00:00.000 Introduction
00:15:23.456 1.1 First Lesson
00:32:10.789 1.2 Second Lesson
01:05:44.123 2.1 Third Lesson
```

**Format**: `HH:MM:SS.mmm Chapter Title`
- Timestamps are cumulative (each chapter starts where previous ended)
- Use 3 decimal places for milliseconds
- First chapter always starts at 00:00:00.000

#### Auto-generate Chapters File

```bash
# Script to auto-generate chapters.txt
current_time=0

while IFS= read -r line; do
    if [[ $line =~ ^file\ \'(.*)\'$ ]]; then
        filepath="${BASH_REMATCH[1]}"
        duration=$(ffprobe -v error -show_entries format=duration \
                   -of default=noprint_wrappers=1:nokey=1 "$filepath")

        # Extract chapter name from filepath
        filename=$(basename "$filepath" .mp3)
        chapter_name=$(echo "$filename" | tr '_' ' ' | sed 's/\b\(.\)/\u\1/g')

        # Format timestamp
        hours=$((current_time / 3600))
        minutes=$(( (current_time % 3600) / 60 ))
        seconds=$(echo "$current_time % 60" | bc)
        timestamp=$(printf "%02d:%02d:%06.3f" $hours $minutes $seconds)

        echo "$timestamp $chapter_name"

        # Add duration to running total
        current_time=$(echo "$current_time + $duration" | bc)
    fi
done < videos/audio_concat.txt > videos/chapters.txt
```

### Step 4.4: Combine Audio Files into Single M4B

Now create the M4B audiobook file:

```bash
# Step 1: Concatenate all audio files
ffmpeg -f concat -safe 0 -i videos/audio_concat.txt \
       -c:a aac \
       -b:a 92k \
       -ac 1 \
       -vn \
       -f mp4 \
       -movflags +faststart \
       videos/temp_audiobook.m4b
```

**FFmpeg options for M4B creation**:

- `-f concat`: Concatenate multiple files
- `-safe 0`: Allow absolute paths

- `-c:a aac`: Audio codec AAC (standard for audiobooks)
  - AAC provides better quality than MP3 at lower bitrates
  - Native format for M4B/M4A containers

- `-b:a 92k`: Audio bitrate 92 kbps
  - Optimal for speech-only audiobooks
  - Balance between quality and file size
  - Apple Books recommendation for audiobooks

- `-ac 1`: Mono audio (1 channel)
  - Reduces file size by ~50% vs stereo
  - Perfect for single-speaker content
  - Use `-ac 2` if stereo is important

- `-vn`: No video stream
  - Important even though source is audio - prevents any video metadata

- `-f mp4`: Force MP4 container format
  - M4B is an MP4 container with different extension
  - Ensures proper container structure

- `-movflags +faststart`: Optimize for streaming
  - Moves metadata to beginning of file
  - Allows playback to start before complete download
  - **Pro**: Better user experience in audiobook players
  - **Con**: Slightly increases processing time
  - **Always use for audiobooks**

### Step 4.5: Add Metadata to M4B

Add audiobook information:

```bash
# Add metadata using FFmpeg
ffmpeg -i videos/temp_audiobook.m4b \
       -c copy \
       -metadata title="Your Course Title" \
       -metadata artist="Instructor Name" \
       -metadata album_artist="Instructor Name" \
       -metadata album="Your Course Title" \
       -metadata date="2024" \
       -metadata genre="Audiobook" \
       -metadata comment="Course description here" \
       -metadata:s:a:0 media_type=2 \
       videos/audiobook_with_metadata.m4b
```

**Metadata fields explained**:

- `title`: Audiobook title (shows in player)
- `artist`: Author/narrator name
- `album_artist`: Primary author (for library grouping)
- `album`: Often same as title for audiobooks
- `date`: Publication year
- `genre`: "Audiobook" for proper categorization
- `comment`: Long-form description

- `media_type=2`: Critical for audiobooks
  - `0` = Movie
  - `1` = Music
  - `2` = Audiobook
  - `6` = Music Video
  - `9` = Short Film
  - `10` = TV Show
  - **Set to 2** to make it appear in Audiobook section of players

**Alternative tools for metadata**:

```bash
# Using AtomicParsley (more audiobook-specific)
AtomicParsley videos/audiobook.m4b \
  --title "Course Title" \
  --artist "Instructor" \
  --album "Course Title" \
  --genre "Audiobook" \
  --year "2024" \
  --stik Audiobook \
  --comment "Description" \
  --overWrite

# Using MP4Box
MP4Box -itags title="Course Title":artist="Instructor":album="Course Title" \
       videos/audiobook.m4b
```

### Step 4.6: Add Chapter Markers

This is the critical step that makes the audiobook navigable:

#### Method 1: Using mp4chaps (Simplest)

```bash
# mp4chaps expects a chapters file with same base name as M4B
# Format: audiobook.chapters.txt for audiobook.m4b

cp videos/chapters.txt videos/audiobook.chapters.txt

# Import chapters into M4B
mp4chaps -i videos/audiobook.m4b

# Verify chapters were added
mp4chaps -l videos/audiobook.m4b
```

**mp4chaps options**:
- `-i`: Import chapters from .chapters.txt file
- `-l`: List chapters in file
- `-e`: Export chapters to text file
- `-r`: Remove all chapters
- `-o`: Optimize (remove and re-add chapters)

**Chapter file format for mp4chaps**:
```
00:00:00.000 Chapter 1 Title
00:15:23.456 Chapter 2 Title
00:32:10.789 Chapter 3 Title
```

#### Method 2: Using MP4Box (More compatible)

```bash
# Import chapters using MP4Box
MP4Box -add videos/audiobook.m4b:chap=videos/chapters.txt \
       videos/audiobook_chaptered.m4b

# Or add to existing file
MP4Box -chap videos/chapters.txt videos/audiobook.m4b
```

**MP4Box chapter options**:
- `-add file:chap=chapters.txt`: Add audio with chapter file
- `-chap chapters.txt`: Add chapters to existing file
- `-chapter`: Alternative chapter import method

**MP4Box chapter format** (more flexible):
```
CHAPTER01=00:00:00.000
CHAPTER01NAME=Introduction
CHAPTER02=00:15:23.456
CHAPTER02NAME=Lesson 1.1
```

Or Nero format:
```
CHAPTER1=00:00:00.000
CHAPTER1NAME=Introduction
CHAPTER2=00:15:23.456
CHAPTER2NAME=Lesson 1.1
```

#### Method 3: Using FFmpeg Metadata File (Most compatible)

```bash
# Create FFmpeg metadata file
cat > videos/ffmpeg_metadata.txt << EOF
;FFMETADATA1

[CHAPTER]
TIMEBASE=1/1000
START=0
END=923456
title=Introduction

[CHAPTER]
TIMEBASE=1/1000
START=923456
END=1930789
title=1.1 First Lesson

[CHAPTER]
TIMEBASE=1/1000
START=1930789
END=3944123
title=1.2 Second Lesson
EOF

# Apply metadata with chapters
ffmpeg -i videos/audiobook_with_metadata.m4b \
       -i videos/ffmpeg_metadata.txt \
       -map_metadata 1 \
       -map_chapters 1 \
       -c copy \
       videos/audiobook_final.m4b
```

**FFmpeg metadata format**:
- `;FFMETADATA1`: Required header
- `TIMEBASE=1/1000`: Millisecond precision
- `START`: Chapter start in milliseconds
- `END`: Chapter end in milliseconds
- `title`: Chapter name

**Converting timestamps to milliseconds**:
```bash
# HH:MM:SS.mmm to milliseconds
# Example: 00:15:23.456 = (15*60 + 23)*1000 + 456 = 923456

# Conversion formula:
ms = (hours * 3600 + minutes * 60 + seconds) * 1000 + milliseconds
```

### Step 4.7: Set Audiobook Atom

Final step to ensure it's recognized as an audiobook:

```bash
# Using AtomicParsley to set media type
AtomicParsley videos/audiobook_final.m4b \
  --stik Audiobook \
  --overWrite

# Verify it worked
AtomicParsley videos/audiobook_final.m4b -t
```

**Media type values** (`--stik`):
- `Movie`: Video content
- `Normal`: Music
- `Audiobook`: Audiobook (this is what you want)
- `Music Video`: Music videos
- `TV Show`: TV episodes
- `Booklet`: PDF booklet

### Step 4.8: Rename to Final Name

```bash
# Rename to clean filename
mv videos/audiobook_final.m4b videos/your_course_title.m4b
```

---

## Phase 5: Chapter Enhancement

### Understanding Chapter Compatibility Issues

Different audiobook players support chapters differently:
- **Apple Books (iOS)**: Very strict, requires specific chapter format
- **Audiobook apps (BookPlayer, Prologue)**: May have compatibility issues
- **Desktop players (VLC, iTunes)**: Usually more forgiving

### Step 5.1: Verify Chapters

Before distribution, verify chapters work:

```bash
# Method 1: Use mp4chaps to list chapters
mp4chaps -l videos/your_course_title.m4b

# Method 2: Use ffprobe to check chapters
ffprobe -v quiet -print_format json -show_chapters \
        videos/your_course_title.m4b

# Method 3: Use MP4Box
MP4Box -info videos/your_course_title.m4b | grep -A 20 Chapter
```

Expected output should show all chapter names and timestamps.

### Step 5.2: Fix Chapter Compatibility Issues

If chapters don't work on iOS:

#### Method A: Convert to Nero Chapter Format

```bash
# Extract chapters
MP4Box -dump-chap-ogg videos/your_course_title.m4b \
       -out videos/chapters_nero.txt

# Remove old chapters and add in Nero format
MP4Box videos/your_course_title.m4b -rem 3 \
       -out videos/temp_no_chapters.m4b

MP4Box videos/temp_no_chapters.m4b \
       -add videos/your_course_title.m4b:chap \
       -chap videos/chapters_nero.txt \
       -new videos/your_course_title_fixed.m4b

# Clean up
rm videos/temp_no_chapters.m4b
```

**MP4Box track numbers**:
- Track 1: Usually audio
- Track 2: Sometimes video/metadata
- Track 3: Often chapters
- Use `-info` to identify tracks: `MP4Box -info file.m4b`

#### Method B: Rebuild with FFmpeg Chapter Metadata

```bash
# Extract existing chapters
ffprobe -v quiet -print_format json -show_chapters \
        videos/your_course_title.m4b > chapters.json

# Create FFmpeg metadata file from JSON
# (manually or with script - see previous FFmpeg metadata format)

# Rebuild with embedded chapters
ffmpeg -i videos/your_course_title.m4b \
       -i videos/ffmpeg_metadata.txt \
       -map_metadata 1 \
       -map_chapters 1 \
       -c copy \
       -f mp4 \
       videos/your_course_title_fixed.m4b
```

### Step 5.3: Alternative Chapter Methods

If standard methods fail:

#### Using QuickTime Chapter Track

Some players support QuickTime chapter tracks:

```bash
# Create QuickTime chapter track using MP4Box
MP4Box -add videos/your_course_title.m4b \
       -add videos/chapters.txt:chap:name=Chapters \
       videos/output_with_qt_chapters.m4b
```

#### Using Cue Points (Podcast-style)

```bash
# Add cue points as metadata
ffmpeg -i input.m4b \
       -metadata chapter_00_start="00:00:00.000" \
       -metadata chapter_00_title="Introduction" \
       -metadata chapter_01_start="00:15:23.456" \
       -metadata chapter_01_title="Chapter 1" \
       -c copy output.m4b
```

---

## Metadata Configuration

### Sample audiobook_metadata.json

```json
{
  "audiobook": {
    "title": "Complete Video Course Audiobook",
    "author": "Dr. Jane Expert",
    "year": "2024",
    "genre": "Educational",
    "description": "Comprehensive course covering advanced topics in detail",
    "language": "en-US",
    "publisher": "Self Published",
    "copyright": "© 2024 Author Name"
  },
  "audio_settings": {
    "bitrate": "92k",
    "codec": "aac",
    "channels": "mono",
    "sample_rate": "44100"
  },
  "chapter_name_overrides": {
    "intro": "Introduction to the Course",
    "1_1_overview": "1.1 Course Overview",
    "1_2_basics": "1.2 Fundamental Concepts",
    "2_1_advanced": "2.1 Advanced Techniques"
  }
}
```

### Automatic Chapter Name Generation

Chapter names can be auto-generated from filenames:

**Pattern examples**:
- `mbt_introduction` → "Introduction"
- `1_1_2_trauma` → "1.1.2 Trauma"
- `1_3_3_embodiment_practices` → "1.3.3 Embodiment Practices"

**Conversion rules**:
1. Special case: `mbt_introduction` → "Introduction"
2. Parse pattern: `{chapter}_{section}_{lesson}_{title}`
3. Convert underscores to spaces
4. Title case each word
5. Format as: "{chapter}.{section}.{lesson} {Title}"

---

## Troubleshooting

### Issue: Chapters Don't Appear on iOS

**Symptoms**: M4B plays fine but no chapter markers in Apple Books or BookPlayer

**Solutions**:
1. Verify media_type is set to 2 (Audiobook):
   ```bash
   AtomicParsley file.m4b -t | grep stik
   ```

2. Try different chapter method (mp4chaps vs MP4Box vs FFmpeg)

3. Use Nero chapter format:
   ```bash
   MP4Box -dump-chap-ogg file.m4b -out chapters.txt
   # Edit to Nero format
   MP4Box file.m4b -chap chapters.txt
   ```

4. Ensure chapter timestamps don't overlap or have gaps

### Issue: Audio/Video Out of Sync

**Symptoms**: Audio doesn't match video or has gaps

**Solutions**:
1. Re-encode instead of copying streams:
   ```bash
   ffmpeg -f concat -safe 0 -i concat.txt \
          -c:v libx264 -c:a aac -b:a 192k \
          output.mp4
   ```

2. Fix timestamps:
   ```bash
   ffmpeg -fflags +genpts -i input.mp4 -c copy output.mp4
   ```

3. Check segment compatibility:
   ```bash
   # Verify all segments have same codec
   for f in segments/*.ts; do
       ffprobe -v error -select_streams v:0 \
               -show_entries stream=codec_name \
               -of default=nokey=1:noprint_wrappers=1 "$f"
   done
   ```

### Issue: Large File Size

**Symptoms**: Final M4B is too large

**Solutions**:
1. Reduce bitrate:
   ```bash
   # From 192k to 92k
   ffmpeg -i input.m4b -c:a aac -b:a 92k -vn output.m4b
   ```

2. Convert to mono:
   ```bash
   ffmpeg -i input.m4b -c:a aac -ac 1 -b:a 64k output.m4b
   ```

3. Lower sample rate:
   ```bash
   ffmpeg -i input.m4b -c:a aac -ar 22050 -b:a 64k output.m4b
   ```

**File size comparison** (for 10-hour audiobook):
- Stereo 192k: ~1.7 GB
- Stereo 128k: ~1.1 GB
- Mono 92k: ~400 MB (recommended)
- Mono 64k: ~280 MB (minimum acceptable)

### Issue: Segments Download Fails

**Symptoms**: Some TS segments won't download or return 403/404 errors

**Solutions**:
1. Add proper headers:
   ```bash
   curl -H "User-Agent: Mozilla/5.0" \
        -H "Referer: https://course-site.com" \
        -H "Origin: https://course-site.com" \
        -o segment.ts \
        "https://cdn.example.com/segment.ts"
   ```

2. Check for authentication tokens in HAR file:
   ```bash
   # Look for Authorization headers or cookies
   # Add them to curl command:
   curl -H "Cookie: session=abc123..." \
        -o segment.ts "URL"
   ```

3. Try different quality stream (lower resolution URL)

### Issue: FFmpeg Not Found

**Symptoms**: `ffmpeg: command not found`

**Solutions**:
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install ffmpeg

# macOS with Homebrew
brew install ffmpeg

# Check installation
ffmpeg -version
```

### Issue: Metadata Not Showing

**Symptoms**: Audiobook plays but shows no title/author

**Solutions**:
1. Verify metadata was added:
   ```bash
   ffprobe -v error -show_format file.m4b | grep TAG
   ```

2. Re-apply metadata:
   ```bash
   AtomicParsley file.m4b \
     --title "Title" \
     --artist "Author" \
     --stik Audiobook \
     --overWrite
   ```

3. Check player compatibility (some players don't read all metadata fields)

### Issue: Audiobook Won't Import to iTunes/Books

**Symptoms**: File rejected when importing

**Solutions**:
1. Ensure file extension is `.m4b` (not `.m4a` or `.mp4`)

2. Verify it's a valid MP4 container:
   ```bash
   ffprobe file.m4b  # Should not show errors
   ```

3. Set audiobook media type:
   ```bash
   AtomicParsley file.m4b --stik Audiobook --overWrite
   ```

4. Rebuild file with correct settings:
   ```bash
   ffmpeg -i broken.m4b \
          -c:a aac -b:a 92k \
          -f mp4 -movflags +faststart \
          fixed.m4b
   ```

---

## Advanced Tips

### Batch Processing Multiple Courses

```bash
# Process entire directory tree
find videos -name "*.mp4" -type f | while read video; do
    audio="${video%.mp4}.mp3"
    if [ ! -f "$audio" ]; then
        ffmpeg -i "$video" -vn -acodec mp3 -ab 192k "$audio"
    fi
done
```

### Creating Chapter Thumbnails

Some players support chapter artwork:

```bash
# Extract frame at each chapter point
ffmpeg -i video.mp4 -ss 00:15:23 -vframes 1 chapter1.jpg

# Add artwork to chapter (using MP4Box)
MP4Box -add video.m4b:name=Audio \
       -add chapter1.jpg:name=Chapter1Cover \
       output.m4b
```

### Quality Testing

```bash
# Compare audio quality
# Extract 30-second samples at different bitrates
for bitrate in 64k 92k 128k 192k; do
    ffmpeg -i source.mp3 -ss 00:05:00 -t 30 \
           -acodec aac -b:a $bitrate \
           test_${bitrate}.m4a
done

# Listen and compare file sizes
ls -lh test_*.m4a
```

### Automated Pipeline Script

```bash
#!/bin/bash
# Complete audiobook creation pipeline

VIDEO_DIR="videos"
OUTPUT_DIR="audiobooks"

# 1. Extract all audio
for dir in "$VIDEO_DIR"/*/; do
    name=$(basename "$dir")
    ffmpeg -i "$dir/${name}.mp4" -vn -acodec mp3 -ab 192k "$dir/${name}.mp3"
done

# 2. Create concat list
find "$VIDEO_DIR" -name "*.mp3" -type f | sort | \
    awk '{print "file '\''" $0 "'\''"}' > "$VIDEO_DIR/concat.txt"

# 3. Generate chapters
# (use timestamp calculation script from earlier)

# 4. Build M4B
ffmpeg -f concat -safe 0 -i "$VIDEO_DIR/concat.txt" \
       -c:a aac -b:a 92k -ac 1 -vn \
       -f mp4 -movflags +faststart \
       "$OUTPUT_DIR/temp.m4b"

# 5. Add metadata
AtomicParsley "$OUTPUT_DIR/temp.m4b" \
  --title "Course Title" \
  --artist "Author" \
  --stik Audiobook \
  --overWrite

# 6. Add chapters
cp "$VIDEO_DIR/chapters.txt" "$OUTPUT_DIR/temp.chapters.txt"
mp4chaps -i "$OUTPUT_DIR/temp.m4b"

# 7. Rename final file
mv "$OUTPUT_DIR/temp.m4b" "$OUTPUT_DIR/course_title.m4b"

echo "Audiobook created: $OUTPUT_DIR/course_title.m4b"
```

---

## Reference Tables

### Audio Codec Comparison

| Codec | Quality | File Size | Compatibility | Use Case |
|-------|---------|-----------|---------------|----------|
| AAC | Excellent | Small | High | **Recommended for M4B** |
| MP3 | Good | Medium | Universal | Intermediate files |
| Opus | Excellent | Smallest | Medium | Modern players |
| Vorbis | Good | Small | Medium | Open source preference |
| WAV | Perfect | Very Large | High | Editing/processing only |

### Bitrate Recommendations

| Content Type | Minimum | Recommended | High Quality |
|--------------|---------|-------------|--------------|
| Speech (mono) | 64k | 92k | 128k |
| Speech (stereo) | 96k | 128k | 192k |
| Music/Mixed | 128k | 192k | 256k |

### File Size Estimates (10-hour audiobook)

| Settings | File Size | Quality |
|----------|-----------|---------|
| Mono 64k AAC | 280 MB | Acceptable |
| Mono 92k AAC | 400 MB | **Recommended** |
| Stereo 128k AAC | 560 MB | High |
| Stereo 192k AAC | 850 MB | Very High |

### FFmpeg Speed Presets

| Preset | Speed | Quality | Use When |
|--------|-------|---------|----------|
| ultrafast | Fastest | Lowest | Testing |
| fast | Fast | Good | Quick processing |
| medium | Medium | Good | **Default** |
| slow | Slow | Better | Final output |
| veryslow | Slowest | Best | Maximum quality needed |

---

## Appendix: Common Command Reference

### Quick Command Reference

```bash
# Extract audio from video
ffmpeg -i input.mp4 -vn -acodec mp3 -ab 192k output.mp3

# Combine videos
ffmpeg -f concat -safe 0 -i concat.txt -c copy output.mp4

# Create M4B audiobook
ffmpeg -f concat -safe 0 -i concat.txt \
       -c:a aac -b:a 92k -ac 1 \
       -f mp4 -movflags +faststart output.m4b

# Add chapters
mp4chaps -i audiobook.m4b

# Set audiobook type
AtomicParsley audiobook.m4b --stik Audiobook --overWrite

# Get media info
ffprobe -v error -show_format -show_streams file.m4b

# List chapters
mp4chaps -l audiobook.m4b
ffprobe -v quiet -print_format json -show_chapters audiobook.m4b
```

---

## Conclusion

This handbook provides a complete manual process for creating audiobooks from video files. The workflow is:

1. **Acquire** video segments via HAR file analysis and download
2. **Combine** TS segments into complete MP4 videos
3. **Extract** audio from videos
4. **Concatenate** audio files with chapter metadata
5. **Package** as M4B audiobook with proper metadata
6. **Enhance** with chapters and audiobook-specific settings

Each step uses standard command-line tools (FFmpeg, MP4Box, mp4chaps, AtomicParsley) that work across platforms. While the process can be automated with scripts, understanding each command and its options allows you to customize for your specific needs and troubleshoot issues effectively.

For automation, consider writing scripts that wrap these commands while maintaining the flexibility to adjust parameters based on your source material quality and target audience requirements.
