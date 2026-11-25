import fs from "node:fs";
import path from "node:path";
import { execFileSync } from "node:child_process";

function parseArgs(argv) {
  const args = {
    input: "episodes/episodio-01.json",
    aspect: "",
    engine: "free-edge-ffmpeg",
    outputDir: "output",
  };
  for (let i = 2; i < argv.length; i++) {
    const k = argv[i];
    const v = argv[i + 1];
    if (k === "--input" && v) {
      args.input = v;
      i++;
    } else if (k === "--aspect" && v) {
      args.aspect = v;
      i++;
    } else if (k === "--engine" && v) {
      args.engine = v;
      i++;
    } else if (k === "--output" && v) {
      args.outputDir = v;
      i++;
    }
  }
  return args;
}

function run(cmd, cmdArgs) {
  execFileSync(cmd, cmdArgs, { stdio: "inherit" });
}

function commandExists(cmd) {
  try {
    execFileSync("which", [cmd], { stdio: "ignore" });
    return true;
  } catch {
    return false;
  }
}

function cleanSpeechLine(text) {
  return String(text || "")
    .replace(/<[^>]+>/g, " ")
    .replace(/\|[^|]+\|/g, " ")
    .replace(/\s+/g, " ")
    .trim();
}

function getEdgeVoiceConfig(speaker) {
  const s = String(speaker || "").toLowerCase();
  if (s.includes("bombo")) {
    return { voice: "es-MX-JorgeNeural", rate: "-5%", pitch: "+6Hz" };
  }
  if (s.includes("pipa")) {
    return { voice: "es-ES-TrianaNeural", rate: "-2%", pitch: "+10Hz" };
  }
  if (s.includes("narrador")) {
    return { voice: "es-ES-ElviraNeural", rate: "-6%", pitch: "+0Hz" };
  }
  return { voice: "es-ES-ElviraNeural", rate: "-4%", pitch: "+2Hz" };
}

function formatSrtTime(sec) {
  const ms = Math.round((sec % 1) * 1000);
  const total = Math.floor(sec);
  const s = total % 60;
  const m = Math.floor(total / 60) % 60;
  const h = Math.floor(total / 3600);
  return `${String(h).padStart(2, "0")}:${String(m).padStart(2, "0")}:${String(
    s
  ).padStart(2, "0")},${String(ms).padStart(3, "0")}`;
}

function resolveImagePath(url) {
  if (!url) return null;
  const cleaned = url.replace(/^\//, "");
  const abs = path.resolve(process.cwd(), cleaned);
  if (fs.existsSync(abs)) return abs;
  return null;
}

async function synthesizeEdgeLine(line, speaker, outMp3) {
  const cfg = getEdgeVoiceConfig(speaker);
  const text = cleanSpeechLine(line);
  if (!text) return false;
  try {
    run("edge-tts", [
      "--voice",
      cfg.voice,
      "--rate",
      cfg.rate,
      "--pitch",
      cfg.pitch,
      "--text",
      text,
      "--write-media",
      outMp3,
    ]);
    return fs.existsSync(outMp3);
  } catch {
    return false;
  }
}

async function synthesizeGeminiDialogue(dialogues, outWav) {
  const apiKey = process.env.GEMINI_API_KEY;
  if (!apiKey) return false;
  const { GoogleGenAI } = await import("@google/genai");
  const ai = new GoogleGenAI({ apiKey });

  const speakerMap = new Map();
  for (const d of dialogues) {
    if (!speakerMap.has(d.speaker)) {
      speakerMap.set(d.speaker, {
        speaker: d.speaker,
        voice: d.voiceId || "Kore",
      });
    }
  }

  const inputBlocks = dialogues.map((d) => ({
    type: "text",
    text: `${d.speaker}: ${d.line}`,
    annotations: [
      {
        type: "speech_metadata",
        speaker: d.speaker,
        style:
          d.voiceStyle ||
          "Warm expressive Spanish children's voice, gentle toddler cadence",
      },
    ],
  }));

  const interaction = await ai.interactions.create({
    model: "gemini-3.8-flash-tts",
    input: inputBlocks,
    response_format: { type: "audio" },
    generation_config: {
      speech_config: {
        speakers: Array.from(speakerMap.values()),
      },
    },
  });

  const audio = interaction.output_audio;
  if (!audio?.data) return false;
  fs.writeFileSync(outWav, Buffer.from(audio.data, "base64"));
  return true;
}

async function main() {
  const args = parseArgs(process.argv);
  const episodePath = path.resolve(process.cwd(), args.input);
  if (!fs.existsSync(episodePath)) {
    throw new Error(`No se encontró el archivo de episodio: ${episodePath}`);
  }
  if (!commandExists("ffmpeg")) {
    throw new Error("ffmpeg es obligatorio para ensamblar el episodio.");
  }

  const parsedJson = JSON.parse(fs.readFileSync(episodePath, "utf8"));
  const episode = parsedJson.episode || parsedJson;
  const aspectRatio = args.aspect || episode.aspectRatio || "16:9";
  const width = aspectRatio === "9:16" ? 720 : 1280;
  const height = aspectRatio === "9:16" ? 1280 : 720;

  const epSlug = `episodio-${String(episode.episodeNumber || 1).padStart(
    2,
    "0"
  )}`;
  const tmpDir = path.resolve(process.cwd(), ".render-tmp", epSlug);
  const outDir = path.resolve(process.cwd(), args.outputDir);
  fs.rmSync(tmpDir, { recursive: true, force: true });
  fs.mkdirSync(tmpDir, { recursive: true });
  fs.mkdirSync(outDir, { recursive: true });

  const hasEdgeTts = commandExists("edge-tts");
  const scenes = episode.scenes || [];
  const sceneVideoFiles = [];
  const sceneAudioFiles = [];
  const srtLines = [];
  let srtIndex = 1;

  const fallbackImg = path.resolve(
    process.cwd(),
    "src/assets/images/episode_01_scene_meadow_1795566233525.jpg"
  );

  for (let idx = 0; idx < scenes.length; idx++) {
    const scene = scenes[idx];
    const dur = Number(scene.durationSec || 15);
    const startSec = Number(scene.startTimeSec ?? idx * 15);
    const padNum = String(idx + 1).padStart(2, "0");

    const dialogues = scene.dialogues || [];
    if (dialogues.length > 0) {
      const slice = dur / dialogues.length;
      dialogues.forEach((d, j) => {
        const s0 = startSec + j * slice + 0.3;
        const s1 = startSec + (j + 1) * slice - 0.3;
        srtLines.push(
          `${srtIndex++}\n${formatSrtTime(s0)} --> ${formatSrtTime(
            s1
          )}\n${d.speaker}: ${cleanSpeechLine(d.line)}\n`
        );
      });
    }

    const sceneWav = path.join(tmpDir, `scene_${padNum}_voice.wav`);
    let generatedVoice = false;

    if (args.engine === "gemini-tts" && dialogues.length > 0) {
      const rawGeminiWav = path.join(tmpDir, `scene_${padNum}_raw.wav`);
      try {
        if (await synthesizeGeminiDialogue(dialogues, rawGeminiWav)) {
          run("ffmpeg", [
            "-y",
            "-i",
            rawGeminiWav,
            "-af",
            `apad=whole_dur=${dur}`,
            "-t",
            String(dur),
            "-ar",
            "44100",
            "-ac",
            "2",
            sceneWav,
          ]);
          generatedVoice = true;
        }
      } catch (e) {
        console.warn(`Gemini TTS falló en escena ${padNum}:`, e.message);
      }
    }

    if (!generatedVoice && hasEdgeTts && dialogues.length > 0) {
      const lineFiles = [];
      for (let j = 0; j < dialogues.length; j++) {
        const lineMp3 = path.join(tmpDir, `scene_${padNum}_l${j}.mp3`);
        const ok = await synthesizeEdgeLine(
          dialogues[j].line,
          dialogues[j].speaker,
          lineMp3
        );
        if (ok) lineFiles.push(lineMp3);
      }
      if (lineFiles.length > 0) {
        const concatTxt = path.join(tmpDir, `scene_${padNum}_audio.txt`);
        const silenceWav = path.join(tmpDir, `silence_045.wav`);
        if (!fs.existsSync(silenceWav)) {
          run("ffmpeg", [
            "-y",
            "-f",
            "lavfi",
            "-i",
            "anullsrc=r=44100:cl=stereo",
            "-t",
            "0.45",
            silenceWav,
          ]);
        }
        const entries = [];
        lineFiles.forEach((f, j) => {
          entries.push(`file '${f}'`);
          if (j < lineFiles.length - 1) entries.push(`file '${silenceWav}'`);
        });
        fs.writeFileSync(concatTxt, entries.join("\n"));

        const rawMerged = path.join(tmpDir, `scene_${padNum}_merged.wav`);
        run("ffmpeg", [
          "-y",
          "-f",
          "concat",
          "-safe",
          "0",
          "-i",
          concatTxt,
          "-ar",
          "44100",
          "-ac",
          "2",
          rawMerged,
        ]);
        run("ffmpeg", [
          "-y",
          "-i",
          rawMerged,
          "-af",
          `apad=whole_dur=${dur}`,
          "-t",
          String(dur),
          "-ar",
          "44100",
          "-ac",
          "2",
          sceneWav,
        ]);
        generatedVoice = true;
      }
    }

    if (!generatedVoice) {
      run("ffmpeg", [
        "-y",
        "-f",
        "lavfi",
        "-i",
        "anullsrc=r=44100:cl=stereo",
        "-t",
        String(dur),
        sceneWav,
      ]);
    }
    sceneAudioFiles.push(sceneWav);

    const sceneMp4 = path.join(tmpDir, `scene_${padNum}_video.mp4`);
    const imgPath = resolveImagePath(scene.keyframeUrl) || fallbackImg;
    const frames = dur * 24;
    const zoomExpr =
      idx % 2 === 0
        ? `min(zoom+0.0007,1.12)`
        : `if(lte(zoom,1.0),1.12,max(1.001,zoom-0.0007))`;

    const vf = [
      `scale=${width * 2}:${height * 2}:force_original_aspect_ratio=increase`,
      `crop=${width * 2}:${height * 2}`,
      `zoompan=z='${zoomExpr}':x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':d=${frames}:s=${width}x${height}:fps=24`,
      `fade=t=in:st=0:d=0.35`,
      `fade=t=out:st=${Math.max(0, dur - 0.35)}:d=0.35`,
      `format=yuv420p`,
    ].join(",");

    run("ffmpeg", [
      "-y",
      "-loop",
      "1",
      "-i",
      imgPath,
      "-t",
      String(dur),
      "-vf",
      vf,
      "-c:v",
      "libx264",
      "-preset",
      "veryfast",
      "-pix_fmt",
      "yuv420p",
      "-r",
      "24",
      sceneMp4,
    ]);
    sceneVideoFiles.push(sceneMp4);
  }

  const srtPath = path.join(outDir, `${epSlug}.srt`);
  fs.writeFileSync(srtPath, srtLines.join("\n"));

  const videoListPath = path.join(tmpDir, "videos.txt");
  fs.writeFileSync(
    videoListPath,
    sceneVideoFiles.map((f) => `file '${f}'`).join("\n")
  );
  const fullVideoNoAudio = path.join(tmpDir, "video_only.mp4");
  run("ffmpeg", [
    "-y",
    "-f",
    "concat",
    "-safe",
    "0",
    "-i",
    videoListPath,
    "-c",
    "copy",
    fullVideoNoAudio,
  ]);

  const audioListPath = path.join(tmpDir, "audios.txt");
  fs.writeFileSync(
    audioListPath,
    sceneAudioFiles.map((f) => `file '${f}'`).join("\n")
  );
  const fullVoices = path.join(tmpDir, "voices_only.wav");
  run("ffmpeg", [
    "-y",
    "-f",
    "concat",
    "-safe",
    "0",
    "-i",
    audioListPath,
    "-c",
    "copy",
    fullVoices,
  ]);

  const totalSec = scenes.reduce((acc, s) => acc + Number(s.durationSec || 15), 0) || 180;
  const musicPath = path.join(tmpDir, "underscore.wav");
  run("ffmpeg", [
    "-y",
    "-f",
    "lavfi",
    "-i",
    `aevalsrc='0.06*sin(2*PI*261.63*t)*abs(sin(PI*t/2))+0.04*sin(2*PI*329.63*t)*abs(sin(PI*(t+0.5)/2))+0.03*sin(2*PI*392.00*t)*abs(sin(PI*(t+1)/2))':s=44100:d=${totalSec}`,
    "-af",
    "lowpass=f=900,aecho=0.6:0.7:120:0.3",
    musicPath,
  ]);

  const finalMp4 = path.join(
    outDir,
    `${epSlug}-${aspectRatio.replace(":", "x")}.mp4`
  );
  const escapedSrt = srtPath.replace(/\\/g, "/").replace(/:/g, "\\:");

  run("ffmpeg", [
    "-y",
    "-i",
    fullVideoNoAudio,
    "-i",
    fullVoices,
    "-i",
    musicPath,
    "-filter_complex",
    `[2:a]volume=0.45[bg];[1:a][bg]amix=inputs=2:duration=first:dropout_transition=2[aout];[0:v]subtitles='${escapedSrt}':force_style='FontSize=20,PrimaryColour=&H00FFFFFF,OutlineColour=&H001E293B,BorderStyle=1,Outline=2,MarginV=28'[vout]`,
    "-map",
    "[vout]",
    "-map",
    "[aout]",
    "-c:v",
    "libx264",
    "-preset",
    "medium",
    "-crf",
    "21",
    "-c:a",
    "aac",
    "-b:a",
    "192k",
    "-shortest",
    finalMp4,
  ]);

  console.log(`Episodio renderizado con éxito: ${finalMp4}`);
}

main().catch((err) => {
  console.error("Error renderizando episodio:", err);
  process.exit(1);
});
