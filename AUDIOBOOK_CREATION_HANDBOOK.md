# Audiobook Creation Manual
## Converting Video Files to Professional M4B Audiobooks

This comprehensive guide explains how to transform a collection of video files into a professional-quality M4B audiobook with proper metadata, chapter markers, and cover art.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [File Organization](#file-organization)
4. [Phase 1: Audio Extraction](#phase-1-audio-extraction)
5. [Phase 2: Metadata Preparation](#phase-2-metadata-preparation)
6. [Phase 3: Audiobook Assembly](#phase-3-audiobook-assembly)
7. [Phase 4: Adding Metadata](#phase-4-adding-metadata)
8. [Phase 5: Chapter Markers](#phase-5-chapter-markers)
9. [Phase 6: Cover Art and Final Touches](#phase-6-cover-art-and-final-touches)
10. [Troubleshooting](#troubleshooting)
11. [Technical Reference](#technical-reference)

---

## Overview

### What This Process Does

Converts a collection of video files (typically course lectures or presentations) into a single M4B audiobook file with:
- High-quality audio optimized for speech
- Automatic chapter markers for easy navigation
- Complete metadata (title, author, year, description, etc.)
- Embedded cover artwork
- Proper audiobook media type for compatibility with audiobook players

### The Complete Pipeline

```
Videos (MP4)
    ↓
Audio Extraction (MP3 @ 192k)
    ↓
Concatenation & Encoding (M4B @ 92k AAC)
    ↓
Metadata Addition
    ↓
Chapter Markers
    ↓
Cover Art
    ↓
Final M4B Audiobook
```

### Time Investment

- **Setup**: 15-30 minutes
- **Audio extraction**: 5-10 minutes per hour of video
- **Encoding**: 15-30 minutes per hour of final audio (longest step)
- **Metadata/Chapters/Art**: 5 minutes total

For a 10-hour course, expect approximately 2-3 hours of processing time.

---

## Prerequisites

### Required Software

#### 1. FFmpeg (Audio/Video Processing)
```bash
# Ubuntu/Debian
sudo apt install ffmpeg

# macOS
brew install ffmpeg

# Verify installation
ffmpeg -version
ffprobe -version
```

**What it does**: Extracts audio from video, concatenates files, encodes to AAC

#### 2. AtomicParsley (MP4 Metadata Tool)
```bash
# Ubuntu/Debian
sudo apt install atomicparsley

# macOS
brew install atomicparsley

# Verify installation
AtomicParsley --version
```

**What it does**: Adds cover art and sets the audiobook media type flag

### Optional but Recommended

- **Python 3.8+**: For automation scripts (metadata management)
- **Text editor**: For creating metadata and concatenation files
- **Image editor**: For creating/resizing cover art (600x600px recommended)

---

## File Organization

### Directory Structure

Create this folder structure before starting:

```
project/
├── videos/              # Source video files (MP4)
├── audio/               # Extracted audio files (MP3)
├── audiobook/           # Final output and working files
│   ├── audio_concat.txt
│   ├── chapters.txt
│   ├── temp_audiobook.m4b
│   └── Final_Audiobook.m4b
├── cover_art.jpg        # Cover image
└── metadata.json        # Course metadata (optional)
```

### File Naming Convention

**Critical**: Files must be named to sort in playback order:

```
S1T1-introduction.mp4
S1T2-getting-started.mp4
S1T3-key-concepts.mp4
S2T1-advanced-topics.mp4
S2T2-practical-examples.mp4
```

Format: `S{session}T{topic}-descriptive-name.mp4`

- **S**: Session/section number
- **T**: Topic/chapter number within that session
- Use hyphens (not spaces) in filenames
- Keep filenames under 200 characters

---

## Phase 1: Audio Extraction

### Step 1.1: Extract Audio from Videos

Navigate to your project directory and run:

```bash
cd project
mkdir -p audio

for video in videos/*.mp4; do
    # Get the base filename without extension
    basename=$(basename "$video" .mp4)

    # Extract audio
    ffmpeg -i "$video" \
           -vn \
           -acodec mp3 \
           -ab 192k \
           -y \
           "audio/${basename}.mp3"
done
```

### Understanding the FFmpeg Options

**Command breakdown**:
```bash
ffmpeg -i "$video" \        # Input video file
       -vn \                # No video (audio only)
       -acodec mp3 \        # Use MP3 codec
       -ab 192k \           # Audio bitrate: 192 kbps
       -y \                 # Overwrite if exists
       "audio/output.mp3"   # Output file
```

**Why these settings?**

- **`-vn`**: Disables video stream, only processes audio
- **`-acodec mp3`**: MP3 is ideal for intermediate files (widely compatible)
- **`-ab 192k`**: High quality for intermediate processing
  - Speech remains clear even after re-encoding
  - Good balance between quality and file size
  - Alternative: `128k` for smaller files, `256k` for maximum quality

### Step 1.2: Get Audio Duration Information

For each extracted audio file, get its duration:

```bash
for audio in audio/*.mp3; do
    duration=$(ffprobe -v error \
                       -show_entries format=duration \
                       -of default=noprint_wrappers=1:nokey=1 \
                       "$audio")
    echo "$(basename "$audio"): ${duration} seconds"
done > audiobook/durations.txt
```

**Why track durations?**
- Needed to calculate chapter start/end times
- Helps estimate total audiobook length
- Useful for progress tracking during encoding

---

## Phase 2: Metadata Preparation

### Step 2.1: Create Metadata File (Optional but Recommended)

Create a JSON file `metadata.json` to track all course information:

```json
{
  "course_title": "Your Course Name",
  "cover_image": "cover_art.jpg",
  "audiobook": {
    "title": "Your Course Name",
    "author": "Instructor Name",
    "year": "2024",
    "genre": "Audiobook",
    "description": "Course description here",
    "language": "en",
    "publisher": "Publisher Name",
    "copyright": "© 2024 Copyright Holder"
  },
  "sessions": [
    {
      "session_num": 1,
      "session_title": "Introduction",
      "topics": [
        {
          "topic_num": 1,
          "topic_title": "Getting Started",
          "audio_file": "S1T1-getting-started.mp3",
          "duration_seconds": 667.5
        }
      ]
    }
  ]
}
```

### Step 2.2: Create Audio Concatenation List

Create `audiobook/audio_concat.txt`:

```text
file '/absolute/path/to/audio/S1T1-introduction.mp3'
file '/absolute/path/to/audio/S1T2-getting-started.mp3'
file '/absolute/path/to/audio/S1T3-key-concepts.mp3'
```

**Important notes**:
- Use **absolute paths** (not relative)
- One file per line
- Format: `file '/path/to/file.mp3'` (with quotes)
- Files are processed in order listed
- Blank lines are ignored

**To generate automatically**:
```bash
for audio in audio/*.mp3; do
    echo "file '$(realpath "$audio")'"
done > audiobook/audio_concat.txt
```

### Step 2.3: Create Chapter Markers File

Create `audiobook/chapters.txt` in FFmpeg metadata format:

```text
;FFMETADATA1

[CHAPTER]
TIMEBASE=1/1000
START=0
END=667498
title=S1T1: Introduction

[CHAPTER]
TIMEBASE=1/1000
START=667498
END=1335145
title=S1T2: Getting Started
```

**Understanding the format**:
- `;FFMETADATA1`: Required header
- `TIMEBASE=1/1000`: Time is in milliseconds
- `START`: Chapter start time in milliseconds
- `END`: Chapter end time in milliseconds (= next chapter's START)
- `title`: Chapter name displayed in players

**Calculating timestamps**:
```python
# Example: Calculate cumulative timestamps
current_time = 0.0
for topic in topics:
    duration = topic['duration_seconds']
    start_ms = int(current_time * 1000)
    end_ms = int((current_time + duration) * 1000)

    print(f"[CHAPTER]")
    print(f"TIMEBASE=1/1000")
    print(f"START={start_ms}")
    print(f"END={end_ms}")
    print(f"title={topic['title']}")
    print()

    current_time += duration
```

---

## Phase 3: Audiobook Assembly

### Step 3.1: Concatenate and Encode to M4B

This is the **longest step** in the process. For a 10-hour audiobook, expect 15-30 minutes.

```bash
ffmpeg -f concat \
       -safe 0 \
       -i audiobook/audio_concat.txt \
       -c:a aac \
       -b:a 92k \
       -ac 1 \
       -vn \
       -f mp4 \
       -movflags +faststart \
       -progress pipe:1 \
       -y \
       audiobook/temp_audiobook.m4b
```

### Understanding the Encoding Options

**Command breakdown**:

```bash
-f concat                    # Input format: concatenation
-safe 0                      # Allow absolute paths
-i audio_concat.txt          # Input: list of files to concatenate
```

**Audio codec settings**:
```bash
-c:a aac                     # Use AAC codec (best for audiobooks)
-b:a 92k                     # Bitrate: 92 kbps
-ac 1                        # Audio channels: 1 (mono)
```

**Output format**:
```bash
-vn                          # No video stream
-f mp4                       # MP4 container (M4B is MP4)
-movflags +faststart         # Optimize for streaming
```

**Progress monitoring**:
```bash
-progress pipe:1             # Output progress info
```

### Why These Settings?

**AAC vs MP3**:
- AAC has better quality at lower bitrates
- Better suited for speech content
- Native support in iOS/macOS audiobook apps

**92k bitrate**:
- Optimal for speech (voices remain clear)
- Significantly smaller than music quality (128k-256k)
- For comparison:
  - 10 hours @ 92k = ~400 MB
  - 10 hours @ 128k = ~550 MB
  - 10 hours @ 192k = ~825 MB

**Mono (1 channel)**:
- Single speaker content doesn't need stereo
- Reduces file size by ~50%
- No quality loss for speech

**faststart flag**:
- Moves metadata to beginning of file
- Allows streaming before full download
- Essential for web/mobile playback

### Monitoring Progress

The `-progress pipe:1` flag outputs progress information:

```
frame=0
fps=0.00
stream_0_0_q=0.0
total_size=0
out_time_us=0
out_time_ms=0
out_time=00:00:00.000000
dup_frames=0
drop_frames=0
speed=   0x
progress=continue
```

Key field: **`out_time`** shows current position in output

---

## Phase 4: Adding Metadata

### Step 4.1: Add Audiobook Metadata with FFmpeg

```bash
ffmpeg -i audiobook/temp_audiobook.m4b \
       -c copy \
       -metadata title="Your Course Title" \
       -metadata artist="Instructor Name" \
       -metadata album_artist="Instructor Name" \
       -metadata album="Your Course Title" \
       -metadata date="2024" \
       -metadata genre="Audiobook" \
       -metadata comment="Course description here" \
       -metadata:s:a:0 media_type=2 \
       -y \
       audiobook/temp_with_metadata.m4b
```

### Understanding Metadata Fields

**Standard metadata**:
- `title`: Audiobook title
- `artist`: Narrator/instructor (shows as "Author" in most players)
- `album_artist`: Same as artist (ensures proper grouping)
- `album`: Album name (often same as title)
- `date`: Publication year
- `genre`: "Audiobook" or "Self-Help", "Education", etc.
- `comment`: Description text

**Critical setting**:
```bash
-metadata:s:a:0 media_type=2
```

**What this does**:
- Sets the audio stream's media type to "Audiobook" (type 2)
- Required for iOS Books app to recognize file as audiobook
- Without this, file appears as music/podcast

**Media type values**:
- 0 = Movie
- 1 = Music
- 2 = Audiobook
- 6 = Music Video
- 9 = Short Film

**Copy codec** (`-c copy`):
- Does NOT re-encode audio
- Only updates metadata
- Very fast (seconds, not minutes)
- No quality loss

---

## Phase 5: Chapter Markers

### Step 5.1: Embed Chapter Markers

```bash
ffmpeg -i audiobook/temp_with_metadata.m4b \
       -i audiobook/chapters.txt \
       -map 0 \
       -map_metadata 0 \
       -map_chapters 1 \
       -c copy \
       -y \
       audiobook/temp_with_chapters.m4b
```

### Understanding Chapter Mapping

**Multiple inputs**:
```bash
-i temp_with_metadata.m4b    # Input 0: M4B with audio and metadata
-i chapters.txt              # Input 1: Chapter markers
```

**Mapping strategy**:
```bash
-map 0                       # Copy all streams from input 0 (audio)
-map_metadata 0              # Keep metadata from input 0 (title, artist, etc.)
-map_chapters 1              # Add chapters from input 1 (chapters.txt)
```

**Why this order matters**:
- `-map_metadata 0`: Preserves title, artist, genre, etc.
- `-map_chapters 1`: Adds chapters WITHOUT overwriting metadata
- Using `-map_metadata 1` would REPLACE all metadata with only chapters

**Copy codec** (`-c copy`):
- No re-encoding (fast, no quality loss)
- Only modifies container metadata

### Verifying Chapters

Check if chapters were added:

```bash
ffprobe -v quiet \
        -print_format json \
        -show_chapters \
        audiobook/temp_with_chapters.m4b
```

Output should show:
```json
{
  "chapters": [
    {
      "id": 0,
      "time_base": "1/1000",
      "start": 0,
      "start_time": "0.000000",
      "end": 667498,
      "end_time": "667.498000",
      "tags": {
        "title": "S1T1: Introduction"
      }
    },
    ...
  ]
}
```

---

## Phase 6: Cover Art and Final Touches

### Step 6.1: Prepare Cover Image

**Recommended specifications**:
- **Format**: JPG or PNG
- **Size**: 600x600 pixels minimum, 3000x3000 maximum
- **Aspect ratio**: 1:1 (square)
- **File size**: Under 1 MB
- **Color space**: RGB

**Resize image if needed**:
```bash
# Using ImageMagick
convert cover_art.jpg -resize 600x600 -quality 90 cover_art_resized.jpg

# Using FFmpeg
ffmpeg -i cover_art.jpg -vf scale=600:600 cover_art_resized.jpg
```

### Step 6.2: Add Cover Art

```bash
AtomicParsley audiobook/temp_with_chapters.m4b \
              --artwork cover_art.jpg \
              --overWrite
```

**What happens**:
- Embeds image as MP4 artwork atom
- Image shows in all audiobook players
- `--overWrite`: Replaces original file (no separate output)

### Step 6.3: Set Audiobook Type Flag

```bash
AtomicParsley audiobook/temp_with_chapters.m4b \
              --stik Audiobook \
              --overWrite
```

**What `--stik` does**:
- Sets the "content type" in iTunes metadata
- Value "Audiobook" tells apps how to handle the file
- Different from the media_type we set earlier (both are needed)

**Available values**:
- Movie
- Music
- Audiobook
- Music Video
- TV Show

### Step 6.4: Create Final File

```bash
mv audiobook/temp_with_chapters.m4b audiobook/Final_Audiobook.m4b
```

Or rename to match your content:
```bash
mv audiobook/temp_with_chapters.m4b "audiobook/Course_Name_2024.m4b"
```

### Step 6.5: Cleanup Temporary Files

```bash
rm audiobook/temp_audiobook.m4b
rm audiobook/temp_with_metadata.m4b
```

Keep these for reference:
- `audio_concat.txt`
- `chapters.txt`
- `durations.txt`

---

## Troubleshooting

### Audio Extraction Issues

**Problem**: FFmpeg fails with "Unknown encoder 'mp3'"

**Solution**: Install with MP3 support:
```bash
# Ubuntu/Debian
sudo apt install ffmpeg libmp3lame0

# macOS
brew reinstall ffmpeg --with-lame
```

---

**Problem**: Extracted audio is very quiet

**Solution**: Add volume normalization:
```bash
ffmpeg -i video.mp4 -vn -acodec mp3 -ab 192k -af "volume=1.5" output.mp3
```

Adjust `1.5` as needed (1.0 = no change, 2.0 = double volume)

---

### Concatenation Issues

**Problem**: "Unsafe file name" error

**Solution**: Add `-safe 0` flag:
```bash
ffmpeg -f concat -safe 0 -i audio_concat.txt ...
```

---

**Problem**: Audio files play in wrong order

**Solution**: Check `audio_concat.txt`:
- Ensure files are listed in correct order
- Use absolute paths
- Verify paths exist: `cat audio_concat.txt | cut -d"'" -f2 | xargs ls -l`

---

**Problem**: Gaps or clicks between chapters

**Solution**: This usually indicates mismatched audio formats. Re-extract all audio with identical settings:
```bash
ffmpeg -i video.mp4 -vn -acodec mp3 -ab 192k -ar 44100 output.mp3
```

The `-ar 44100` ensures consistent sample rate.

---

### Chapter Issues

**Problem**: Chapters don't appear in player

**Solution**:
1. Verify chapters file format (must start with `;FFMETADATA1`)
2. Check timestamp calculations (END of one = START of next)
3. Ensure `-map_chapters 1` references correct input
4. Test with VLC: File → Information → Codec Details

---

**Problem**: Chapter timestamps are off

**Cause**: Calculated durations don't match actual audio

**Solution**: Get precise durations:
```bash
for audio in audio/*.mp3; do
    ffprobe -v error -show_entries format=duration \
            -of default=noprint_wrappers=1:nokey=1 "$audio"
done
```

Recalculate chapter timestamps with exact values.

---

### Metadata Issues

**Problem**: Metadata doesn't show in player

**Solution**: Some players cache metadata. Try:
1. Reimport the file
2. Clear player cache
3. Test in different player (VLC, iTunes, BookPlayer)

---

**Problem**: Title shows but not author

**Solution**: Set both `artist` and `album_artist`:
```bash
-metadata artist="Name" -metadata album_artist="Name"
```

---

### Cover Art Issues

**Problem**: AtomicParsley: "command not found"

**Solution**: Install AtomicParsley:
```bash
# Ubuntu/Debian
sudo apt install atomicparsley

# macOS
brew install atomicparsley
```

---

**Problem**: Cover art too large / file size increased dramatically

**Solution**: Resize image before embedding:
```bash
ffmpeg -i cover.jpg -vf scale=600:600 cover_small.jpg
```

Target: 600x600 pixels, under 500 KB

---

**Problem**: Cover art doesn't display

**Solution**:
1. Verify image format (JPG/PNG only)
2. Check aspect ratio (should be square)
3. Try re-embedding with different image
4. Some players require both artwork and stik type set

---

### File Size Issues

**Problem**: Final file is too large

**Solutions**:
1. Lower bitrate: `-b:a 64k` (minimum for speech)
2. Ensure mono: `-ac 1`
3. Check if video accidentally included: Use `-vn`

**Typical sizes**:
- 10 hours @ 64k mono = ~275 MB
- 10 hours @ 92k mono = ~395 MB
- 10 hours @ 128k stereo = ~1100 MB

---

**Problem**: Final file won't play on iOS

**Solution**: Ensure:
1. File extension is `.m4b`
2. Container is MP4: `-f mp4`
3. Codec is AAC: `-c:a aac`
4. Media type set: `-metadata:s:a:0 media_type=2`
5. Stik type set: `--stik Audiobook`

---

### Performance Issues

**Problem**: Encoding is extremely slow

**Solutions**:
1. Check CPU usage: `top` or `htop`
2. Use hardware acceleration (if available):
```bash
ffmpeg -hwaccel auto -i input ... output
```
3. Lower quality temporarily for testing: `-b:a 64k`
4. Process in parallel (split into sections)

---

**Problem**: Running out of disk space

**Solution**:
1. Extract audio directly to M4B (skip MP3 intermediate)
2. Process in batches, deleting videos after extraction
3. Use lower bitrate for extraction: `-ab 128k`

---

## Technical Reference

### Complete Command Summary

**1. Extract audio from all videos**:
```bash
for video in videos/*.mp4; do
    ffmpeg -i "$video" -vn -acodec mp3 -ab 192k -y "audio/$(basename "$video" .mp4).mp3"
done
```

**2. Create concatenation list**:
```bash
for audio in audio/*.mp3; do echo "file '$(realpath "$audio")'"; done > audiobook/audio_concat.txt
```

**3. Create chapters file** (manual or scripted based on durations)

**4. Concatenate and encode**:
```bash
ffmpeg -f concat -safe 0 -i audiobook/audio_concat.txt \
       -c:a aac -b:a 92k -ac 1 -vn -f mp4 -movflags +faststart \
       -y audiobook/temp_audiobook.m4b
```

**5. Add metadata**:
```bash
ffmpeg -i audiobook/temp_audiobook.m4b -c copy \
       -metadata title="Title" \
       -metadata artist="Author" \
       -metadata album_artist="Author" \
       -metadata album="Title" \
       -metadata date="2024" \
       -metadata genre="Audiobook" \
       -metadata:s:a:0 media_type=2 \
       -y audiobook/temp_with_metadata.m4b
```

**6. Add chapters**:
```bash
ffmpeg -i audiobook/temp_with_metadata.m4b -i audiobook/chapters.txt \
       -map 0 -map_metadata 0 -map_chapters 1 -c copy \
       -y audiobook/temp_with_chapters.m4b
```

**7. Add cover art and set type**:
```bash
AtomicParsley audiobook/temp_with_chapters.m4b --artwork cover.jpg --overWrite
AtomicParsley audiobook/temp_with_chapters.m4b --stik Audiobook --overWrite
```

**8. Rename to final**:
```bash
mv audiobook/temp_with_chapters.m4b audiobook/Final_Audiobook.m4b
```

---

### FFmpeg Options Reference

#### Audio Codec Options
- `-acodec mp3` or `-c:a mp3`: Use MP3 codec
- `-acodec aac` or `-c:a aac`: Use AAC codec
- `-c copy`: Copy without re-encoding

#### Bitrate Options
- `-ab 64k` or `-b:a 64k`: 64 kbps (minimum for speech)
- `-ab 92k` or `-b:a 92k`: 92 kbps (optimal for audiobooks)
- `-ab 128k` or `-b:a 128k`: 128 kbps (high quality speech)
- `-ab 192k` or `-b:a 192k`: 192 kbps (very high quality)

#### Channel Options
- `-ac 1`: Mono (1 channel)
- `-ac 2`: Stereo (2 channels)

#### Sample Rate Options
- `-ar 22050`: 22.05 kHz (low quality)
- `-ar 44100`: 44.1 kHz (standard)
- `-ar 48000`: 48 kHz (high quality)

#### Container Options
- `-f mp4`: MP4 container (for M4B)
- `-f mp3`: MP3 container
- `-movflags +faststart`: Optimize MP4 for streaming

#### Stream Selection
- `-vn`: No video
- `-an`: No audio
- `-map 0`: Map all streams from input 0
- `-map 0:a`: Map only audio from input 0

---

### AtomicParsley Options Reference

#### Artwork
```bash
--artwork /path/to/image.jpg    # Add cover art
```

#### Media Type
```bash
--stik Audiobook                # Set as audiobook
--stik Music                    # Set as music
--stik Movie                    # Set as movie
```

#### Metadata
```bash
--title "Title"                 # Set title
--artist "Artist"               # Set artist
--album "Album"                 # Set album
--year "2024"                   # Set year
--genre "Genre"                 # Set genre
--comment "Description"         # Set description
```

#### Other Options
```bash
--overWrite                     # Modify file in-place
--output /path/to/output.m4b    # Write to new file
```

---

### Quality vs File Size Guide

**For 1 hour of content**:

| Bitrate | Channels | File Size | Quality | Use Case |
|---------|----------|-----------|---------|----------|
| 32k | Mono | 14 MB | Poor | Not recommended |
| 64k | Mono | 28 MB | Acceptable | Limited storage |
| 92k | Mono | 40 MB | Good | **Recommended** |
| 128k | Mono | 56 MB | Very Good | High quality |
| 96k | Stereo | 85 MB | Good | Music/ambience |
| 128k | Stereo | 110 MB | Very Good | Music content |

**For 10-hour audiobook**:
- 64k mono: ~275 MB
- 92k mono: ~395 MB
- 128k mono: ~550 MB

---

### File Format Comparison

| Format | Codec | Container | Chapters | Metadata | iOS Support | Android |
|--------|-------|-----------|----------|----------|-------------|---------|
| M4B | AAC | MP4 | Yes | Full | Excellent | Good |
| M4A | AAC | MP4 | Yes | Full | Good | Good |
| MP3 | MP3 | MP3 | Limited | Limited | Good | Excellent |
| OGG | Vorbis | OGG | Yes | Good | Poor | Good |

**Why M4B?**
- Native iOS audiobook format
- Better compression than MP3
- Full chapter support
- Complete metadata support
- Recognized by audiobook apps

---

### Metadata Fields Mapping

| FFmpeg Field | AtomicParsley | iTunes/iOS Display | Purpose |
|--------------|---------------|-------------------|---------|
| title | --title | Title | Audiobook title |
| artist | --artist | Author | Narrator/author name |
| album_artist | --albumArtist | Album Artist | Same as artist |
| album | --album | Album | Usually same as title |
| date | --year | Year | Publication year |
| genre | --genre | Genre | Category |
| comment | --comment | Description | Long description |
| media_type=2 | --stik Audiobook | Media Type | Audiobook flag |

---

## Advanced Techniques

### Batch Processing Multiple Courses

Process multiple courses in parallel:

```bash
#!/bin/bash
for course_dir in courses/*/; do
    (
        cd "$course_dir"
        ./extract_audio.sh
        ./create_audiobook.sh
    ) &
done
wait
```

### Variable Bitrate (VBR) Encoding

For better quality at similar file sizes:

```bash
ffmpeg -i input.mp3 -c:a aac -q:a 2 -ac 1 output.m4b
```

Quality scale: 0 (highest) to 9 (lowest), 2 ≈ 128k CBR

### Adding Silence Between Chapters

```bash
# Create 2 seconds of silence
ffmpeg -f lavfi -i anullsrc=r=44100:cl=mono -t 2 silence.mp3

# Modify concat list to include silence between chapters
file 'chapter1.mp3'
file 'silence.mp3'
file 'chapter2.mp3'
file 'silence.mp3'
```

### Speed Adjustment

Speed up or slow down content:

```bash
# 1.25x speed (25% faster)
ffmpeg -i input.mp3 -filter:a "atempo=1.25" output.mp3

# For changes > 2x, chain filters:
# 2.5x = 2.0 * 1.25
ffmpeg -i input.mp3 -filter:a "atempo=2.0,atempo=1.25" output.mp3
```

### Noise Reduction

Remove background noise:

```bash
# Create noise profile from first 1 second
ffmpeg -i input.mp3 -t 1 -af "highpass=f=200,lowpass=f=3000" noise_profile.mp3

# Apply noise reduction
ffmpeg -i input.mp3 -af "highpass=f=200,lowpass=f=3000" output.mp3
```

### Volume Normalization

Ensure consistent volume across chapters:

```bash
# Analyze audio
ffmpeg -i input.mp3 -af "volumedetect" -f null -

# Output shows: mean_volume: -23.5 dB
# Normalize to -20 dB
ffmpeg -i input.mp3 -af "volume=3.5dB" output.mp3
```

---

## Resources

### Tools Documentation
- **FFmpeg**: https://ffmpeg.org/documentation.html
- **AtomicParsley**: http://atomicparsley.sourceforge.net/
- **M4B Specification**: https://developer.apple.com/documentation/

### Testing Your Audiobook
- **VLC Media Player**: https://www.videolan.org/ (all platforms)
- **BookPlayer**: https://github.com/TortugaPower/BookPlayer (iOS)
- **Voice Audiobook Player**: https://voiceapp.in/ (Android)

### Online Validators
- **MediaInfo**: https://mediaarea.net/en/MediaInfo (inspect file details)
- **FFprobe**: Built into FFmpeg, command-line file inspector

---

## Appendix: Complete Automation Script

```bash
#!/bin/bash
# audiobook_creator.sh - Complete automation script

set -e  # Exit on error

# Configuration
PROJECT_DIR="$(pwd)"
VIDEOS_DIR="$PROJECT_DIR/videos"
AUDIO_DIR="$PROJECT_DIR/audio"
AUDIOBOOK_DIR="$PROJECT_DIR/audiobook"
COVER_ART="$PROJECT_DIR/cover_art.jpg"

# Metadata
TITLE="Your Audiobook Title"
AUTHOR="Author Name"
YEAR="2024"
GENRE="Audiobook"
DESCRIPTION="Audiobook description"

# Audio settings
EXTRACT_BITRATE="192k"
FINAL_BITRATE="92k"
CHANNELS="1"

echo "==================================="
echo "Audiobook Creation Script"
echo "==================================="

# Step 1: Create directories
echo -e "\n[1/8] Creating directories..."
mkdir -p "$AUDIO_DIR" "$AUDIOBOOK_DIR"

# Step 2: Extract audio
echo -e "\n[2/8] Extracting audio from videos..."
for video in "$VIDEOS_DIR"/*.mp4; do
    basename=$(basename "$video" .mp4)
    output="$AUDIO_DIR/${basename}.mp3"

    if [ -f "$output" ]; then
        echo "  ⊘ Skipping: $basename (already exists)"
    else
        echo "  ↓ Extracting: $basename"
        ffmpeg -i "$video" -vn -acodec mp3 -ab "$EXTRACT_BITRATE" \
               -loglevel error -stats -y "$output"
    fi
done

# Step 3: Create concatenation list
echo -e "\n[3/8] Creating concatenation list..."
> "$AUDIOBOOK_DIR/audio_concat.txt"
for audio in "$AUDIO_DIR"/*.mp3; do
    echo "file '$(realpath "$audio")'" >> "$AUDIOBOOK_DIR/audio_concat.txt"
done
echo "  ✓ Listed $(wc -l < "$AUDIOBOOK_DIR/audio_concat.txt") audio files"

# Step 4: Create chapters (simplified - equal duration chapters)
echo -e "\n[4/8] Creating chapter markers..."
current_time=0
chapter_num=1
> "$AUDIOBOOK_DIR/chapters.txt"
echo ";FFMETADATA1" >> "$AUDIOBOOK_DIR/chapters.txt"
echo "" >> "$AUDIOBOOK_DIR/chapters.txt"

for audio in "$AUDIO_DIR"/*.mp3; do
    duration=$(ffprobe -v error -show_entries format=duration \
                       -of default=noprint_wrappers=1:nokey=1 "$audio")

    start_ms=$(echo "$current_time * 1000" | bc | cut -d. -f1)
    current_time=$(echo "$current_time + $duration" | bc)
    end_ms=$(echo "$current_time * 1000" | bc | cut -d. -f1)

    basename=$(basename "$audio" .mp3)

    cat >> "$AUDIOBOOK_DIR/chapters.txt" << EOF
[CHAPTER]
TIMEBASE=1/1000
START=$start_ms
END=$end_ms
title=$basename

EOF

    chapter_num=$((chapter_num + 1))
done
echo "  ✓ Created $((chapter_num - 1)) chapters"

# Step 5: Concatenate and encode
echo -e "\n[5/8] Creating M4B audiobook (this may take a while)..."
ffmpeg -f concat -safe 0 -i "$AUDIOBOOK_DIR/audio_concat.txt" \
       -c:a aac -b:a "$FINAL_BITRATE" -ac "$CHANNELS" \
       -vn -f mp4 -movflags +faststart \
       -loglevel error -stats \
       -y "$AUDIOBOOK_DIR/temp_audiobook.m4b"
echo "  ✓ Encoding complete"

# Step 6: Add metadata
echo -e "\n[6/8] Adding metadata..."
ffmpeg -i "$AUDIOBOOK_DIR/temp_audiobook.m4b" -c copy \
       -metadata title="$TITLE" \
       -metadata artist="$AUTHOR" \
       -metadata album_artist="$AUTHOR" \
       -metadata album="$TITLE" \
       -metadata date="$YEAR" \
       -metadata genre="$GENRE" \
       -metadata comment="$DESCRIPTION" \
       -metadata:s:a:0 media_type=2 \
       -loglevel error -stats \
       -y "$AUDIOBOOK_DIR/temp_with_metadata.m4b"
echo "  ✓ Metadata added"

# Step 7: Add chapters
echo -e "\n[7/8] Adding chapter markers..."
ffmpeg -i "$AUDIOBOOK_DIR/temp_with_metadata.m4b" \
       -i "$AUDIOBOOK_DIR/chapters.txt" \
       -map 0 -map_metadata 0 -map_chapters 1 -c copy \
       -loglevel error -stats \
       -y "$AUDIOBOOK_DIR/temp_with_chapters.m4b"
echo "  ✓ Chapters added"

# Step 8: Add cover art and finalize
echo -e "\n[8/8] Adding cover art and finalizing..."
if [ -f "$COVER_ART" ]; then
    AtomicParsley "$AUDIOBOOK_DIR/temp_with_chapters.m4b" \
                  --artwork "$COVER_ART" \
                  --overWrite > /dev/null 2>&1
    echo "  ✓ Cover art added"
else
    echo "  ⚠ Cover art not found: $COVER_ART"
fi

AtomicParsley "$AUDIOBOOK_DIR/temp_with_chapters.m4b" \
              --stik Audiobook \
              --overWrite > /dev/null 2>&1
echo "  ✓ Audiobook type set"

# Rename to final
FINAL_FILE="$AUDIOBOOK_DIR/${TITLE// /_}.m4b"
mv "$AUDIOBOOK_DIR/temp_with_chapters.m4b" "$FINAL_FILE"

# Cleanup
rm -f "$AUDIOBOOK_DIR/temp_audiobook.m4b"
rm -f "$AUDIOBOOK_DIR/temp_with_metadata.m4b"

# Summary
FILE_SIZE=$(du -h "$FINAL_FILE" | cut -f1)
DURATION=$(ffprobe -v error -show_entries format=duration \
                   -of default=noprint_wrappers=1:nokey=1 "$FINAL_FILE")
HOURS=$(echo "$DURATION / 3600" | bc -l | xargs printf "%.1f")

echo ""
echo "==================================="
echo "✅ AUDIOBOOK CREATED SUCCESSFULLY!"
echo "==================================="
echo "  📁 File: $FINAL_FILE"
echo "  📦 Size: $FILE_SIZE"
echo "  ⏱  Duration: $HOURS hours"
echo "==================================="
```

Save this as `audiobook_creator.sh`, make it executable (`chmod +x audiobook_creator.sh`), and run it.

---

## Summary

This manual has covered the complete process of creating professional audiobooks from video files:

1. **Audio Extraction**: Converting video to high-quality intermediate audio
2. **Metadata Preparation**: Organizing content structure and information
3. **Assembly**: Concatenating and encoding to optimal M4B format
4. **Metadata Addition**: Adding title, author, and audiobook flags
5. **Chapters**: Creating navigable chapter markers
6. **Finalization**: Adding cover art and final touches

The resulting M4B file will:
- Play on all major platforms
- Display proper metadata
- Support chapter navigation
- Show cover artwork
- Be recognized as an audiobook (not music)

**Key takeaways**:
- Use AAC codec at 92k bitrate for speech
- Mono audio reduces file size without quality loss
- Both FFmpeg and AtomicParsley steps are necessary
- Absolute paths prevent concatenation errors
- Chapter timestamps must be cumulative and in milliseconds

With these tools and techniques, you can create professional-quality audiobooks from any video course or presentation series.
