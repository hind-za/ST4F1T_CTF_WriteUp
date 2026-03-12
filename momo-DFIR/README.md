# CTF DFIR — momo Challenge Write-up
---
Event: ST4F1T Category: DFIR
---
## Challenge Description

A friend of **CR0M80** was chilling and listening to the radio station **ash.hadaFM** when suddenly he started hearing a really weird noise — something completely unrecognizable, not like regular music or speech. He had no idea what it was. The only clue CR0M80 could give him was a file called `momo.pdf`. Our job is to figure out what was going on and find the flag.

---

## Step 1 — Check the file type

The first thing I did is to check the type of the file. Just because something is named `.pdf` doesn't mean it actually is one.

```bash
file momo.pdf
```

<img width="939" height="112" alt="image" src="https://github.com/user-attachments/assets/9ad2054f-7640-4fe3-94af-1d26e049d0a0" />

It's not a PDF — it's a **WAV audio file** disguised with a `.pdf` extension. I renamed it properly.

```bash
cp momo.pdf ctf_momo.wav
```

---

## Step 2 — Checking if there is something hidden inside

A WAV file disguised as a PDF is already suspicious. So there could be more data hidden inside. I used **Binwalk**, a tool that scans files looking for embedded/hidden content.

```bash
binwalk ctf_momo.wav
```

<img width="939" height="162" alt="image" src="https://github.com/user-attachments/assets/02655203-30a2-4d6c-9137-570493960b0e" />

There are **3 things** in this file:
- The main WAV starting at offset `0`
- A **second WAV hidden at offset `5853262`**
- An **MP3 tag at offset `57718400`**

So there's definitely something buried inside. Let's dig it out.

---

## Step 3 — Extracting the Hidden WAV

I used Python to extract the hidden WAV — it's much faster than using `dd` byte by byte.

```bash
python3 -c "
with open('ctf_momo.wav','rb') as f:
    f.seek(5853262)
    data = f.read(57718400 - 5853262)
with open('hidden2.wav','wb') as f:
    f.write(data)
print('Done!')
"
```

<img width="907" height="193" alt="image" src="https://github.com/user-attachments/assets/747cb90c-9d47-4253-ac7b-2a2e8455f78f" />

Let's see what we got:

```bash
file hidden2.wav
exiftool hidden2.wav
```

<img width="951" height="454" alt="image" src="https://github.com/user-attachments/assets/d86bee9f-0408-4449-97c3-8b8f5017ec29" />

We have a **4 minute 30 second audio file** — but the header is corrupted. Let's fix that.

---

## Step 4 — Fixing the Corrupted Header

The file has a broken RIFF header. FFmpeg can repair it by re-encoding with a clean header.

```bash
ffmpeg -i hidden2.wav -ar 48000 -ac 2 -acodec pcm_s16le fixed.wav
```

<img width="1852" height="584" alt="image" src="https://github.com/user-attachments/assets/b8368f36-eebf-43b6-9c90-8d6c746673bd" />

After this, the file is clean and ready to analyze.

---

## Step 5 — What's Inside This Audio? (Spectrogram)

Instead of just listening, I generated a **spectrogram** — a visual representation of the frequencies present in the sound over time.

```bash
sox fixed.wav -n spectrogram -o momo_spectrogram.png
```

<img width="1070" height="709" alt="Screenshot 2026-03-08 064346" src="https://github.com/user-attachments/assets/ffcbd636-cd6e-4ebb-afd9-0ff9bb3961a7" />

What I saw immediately:
- A **bright yellow line constantly at around 2 kHz** — this is a synchronization tone
- Colorful noise patterns above it — encoded image data
- Signal active from **0 to ~145 seconds**, then silence

This is the signature of **SSTV — Slow Scan Television**. SSTV is an old radio technique used to **transmit images over audio**. When you hear it, it sounds like a weird high-pitched noise — exactly what CR0M80's friend heard on the radio. Mystery solved on the scenario side.

---

## Step 6 — Preparing the Audio for SSTV Decoding

SSTV decoders work best with **mono audio**. I converted the stereo file:

```bash
sox fixed.wav -r 48000 -c 1 -b 16 -e signed-integer -L for_qsstv.wav
```

<img width="894" height="77" alt="image" src="https://github.com/user-attachments/assets/42438234-340a-4fef-9509-214321bc509c" />

The `dither clipped` warning is harmless, ignore it.

---

## Step 7 — Decoding the SSTV Signal

I used **QSSTV**, a SSTV decoder available on Kali Linux:

```bash
qsstv
```

Inside QSSTV:
1. Go to `Options` → `Sound` → select **"from file"**
2. Load `for_qsstv.wav`
3. Click **Start**

QSSTV automatically detected the SSTV mode and decoded the image line by line — and there it was, the **flag revealed in the decoded image**.

<img width="1906" height="882" alt="image" src="https://github.com/user-attachments/assets/c0efc83c-3fa9-4402-9477-f5a30ffe348a" />

---
## Flag 
```bash
ESECxGCD{aslan_had_challenge_easy}
```
---
## Tools Used

| Tool | Purpose |
|------|---------|
| `file` | Detect real file type |
| `binwalk` | Find hidden embedded files |
| `python3` | Extract raw bytes from specific offset |
| `ffmpeg` | Repair corrupted WAV header |
| `sox` | Audio conversion + spectrogram generation |
| `exiftool` | Read file metadata |
| `qsstv` | Decode SSTV signal to image |

---


