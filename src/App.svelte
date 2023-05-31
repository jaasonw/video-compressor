<script lang="ts">
  import { createFFmpeg, fetchFile } from "@ffmpeg/ffmpeg";
  import { onMount } from "svelte";

  const ffmpeg = createFFmpeg({ log: true });

  let files: FileList | null = null;
  let converted = "";
  let ready = false;
  let messages = [];

  onMount(async () => {
    await ffmpeg.load();
    messages = [...messages, "FFmpeg Loaded"];
    ready = true;
  });

  const convert = async (size, quality, fps, audio) => {
    const targetSize = size * 1024 * 1024;
    const fileFormat = files[0].name.split(".").at(-1);
    ffmpeg.FS("writeFile", `test.${fileFormat}`, await fetchFile(files[0]));
    messages = [...messages, `Loaded ${files[0].name}`];

    let numIterations = 0;
    let currentSize = files[0].size;
    let data = ffmpeg.FS("readFile", `test.${fileFormat}`);
    while (currentSize > targetSize) {
      if (numIterations > 5) {
        alert(
          "Could not compress to target size after 5 iterations. Try adjusting your settings and try again"
        );
        return;
      }
      await ffmpeg.run(
        "-i",
        `test.${fileFormat}`,
        "-c:v",
        "libx264",
        "-preset",
        "ultrafast",
        "-crf",
        fps,
        "-vf",
        `scale=-2:${quality}`,
        audio == "yes" ? "-c:a" : " -c:an",
        "copy",
        "output.mp4"
      );

      data = ffmpeg.FS("readFile", "output.mp4");
      currentSize = data.byteLength;
      numIterations++;
      messages = [
        ...messages,
        `Iteration ${numIterations}: ${currentSize} bytes`,
      ];
    }
    messages = [...messages, `Finished compressing to ${currentSize} bytes`];
    converted = URL.createObjectURL(
      new Blob([data.buffer], { type: "video/mp4" })
    );
  };

  const submit = (e: any) => {
    const formData = new FormData(e.target);

    const data = {} as any;
    for (let field of formData) {
      const [key, value] = field;
      data[key] = value;
    }
    console.log(data);
    convert(data.size, data.quality, data.fps, data.audio);
  };

  $: if (files) {
    console.log(files);

    for (const file of files) {
      console.log(`${file.name}: ${file.size} bytes`);
    }
  }
</script>

{#if !ready}
  <h1>Loading...</h1>
{:else}
  {#if files}
    <video
      id="player"
      width="500"
      controls
      src={URL.createObjectURL(files[0])}
    />
  {/if}

  <input bind:files type="file" id="uploader" />
  <pre>
    <code
      >{#each messages as message}{message}<br />{/each}</code
    >
  </pre>

  <form on:submit|preventDefault={submit} id="settings">
    <h1>what is this?</h1>
    this is a webapp that takes an input video file and aggressively compresses it
    until it fits within the 25mb upload limit for discord of 100mb for nitro users.
    it uses ffmpeg to do this, and it's all done in the browser so your video never
    leaves your computer.
    <label for="size">Size</label>
    <select name="size" id="size">
      <option value="25">25mb</option>
      <option value="100">100mb</option>
    </select>
    <label for="quality">Quality</label>
    <select name="quality">
      <option value="1080">1080p</option>
      <option value="720">720p</option>
      <option value="480">480p</option>
      <option value="360">360p</option>
      <option value="240">240p</option>
      <option value="144">144p</option>
    </select>
    <label for="fps">FPS</label>
    <select id="fps" name="fps">
      <option value="60">60fps</option>
      <option value="30">30fps</option>
      <option value="15">15fps</option>
    </select>
    <label for="audio">Audio</label>
    <select id="audio" name="audio">
      <option value="yes">Yes</option>
      <option value="no">No</option>
    </select>

    <button type="submit"> Convert </button>
  </form>
  <br />
  {#if converted}
    <video id="player" width="500" controls src={converted} />
    <br />
    <a href={converted} download="output.mp4">Download</a>
  {/if}
  <!-- {/else} -->
{/if}
