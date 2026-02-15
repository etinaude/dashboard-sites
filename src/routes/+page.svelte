<script lang="ts">
  import { onMount } from "svelte";
  import { fade } from "svelte/transition";

  let availableSites: string[] = [];

  let delayTime = 20;

  type Frame = {
    type: "iframe" | "img";
    url: string;
    image?: string;
  };

  let isEvenFrame = true;

  let frameA: Frame = {
    type: "iframe",
    url: "",
  };

  let frameB: Frame = {
    type: "iframe",
    url: "",
  };

  const getScreenshot = (url: string) =>
    `https://api.microlink.io/?url=${encodeURIComponent(url)}&screenshot=true&embed=screenshot.url`;

  async function init() {
    const res = await fetch("/sites.txt");
    const text = await res.text();
    availableSites = text
      .split("\n")
      .filter((s) => s.trim().startsWith("http"));
  }

  async function getNextSiteData(): Promise<Frame> {
    const randomIndex = Math.floor(Math.random() * availableSites.length);
    const url = availableSites.splice(randomIndex, 1)[0];

    try {
      // we assume we need a screenshot.
      const response = await fetch(url, { mode: "no-cors" });
      if (!response.ok) throw new Error("Network response was not ok");
      return { url, type: "iframe" };
    } catch (e) {
      return {
        url,
        image: getScreenshot(url),
        type: "img",
      };
    }
  }

  async function rotate() {
    if (availableSites.length === 0) await init();
    isEvenFrame = !isEvenFrame;

    // load next site data after a short delay to allow transition to complete
    setTimeout(async () => {
      if (isEvenFrame) {
        frameA = await getNextSiteData();
      } else {
        frameB = await getNextSiteData();
      }
    }, 1000);
  }

  onMount(async () => {
    await init();
    frameB = await getNextSiteData();
    frameA = await getNextSiteData();
    setInterval(rotate, delayTime * 1000);
  });
</script>

<main>
  {#if isEvenFrame}
    <div class="viewport" transition:fade={{ duration: 100 }}>
      {#if frameA.type === "iframe"}
        IFRAME
        <iframe src={frameA.url} title="site" loading="eager"></iframe>
      {:else}
        IMAGE
        <img src={frameA.image} alt="site screenshot" loading="eager" />
      {/if}
    </div>
    <div class="url-box">
      {frameA.url}
    </div>
  {:else}
    <div class="viewport" transition:fade={{ duration: 100 }}>
      {#if frameB.type === "iframe"}
        <iframe src={frameB.url} title="site" loading="eager"></iframe>
      {:else}
        <img src={frameB.image} alt="site screenshot" loading="eager" />
      {/if}
      <div class="url-box">
        {frameB.url}
      </div>
    </div>
  {/if}
</main>

<style lang="scss">
  :global(body, html) {
    margin: 0;
    padding: 0;
    overflow: hidden;
    background: #000;
    font-family: sans-serif;
  }

  .viewport {
    position: absolute;
    inset: 0;
    width: 100vw;
    height: 100vh;

    iframe,
    img {
      width: 100%;
      height: 100%;
      border: none;
      object-fit: cover;
      background: #fff;
    }
  }

  .url-box {
    position: fixed;
    bottom: 20px;
    left: 20px;
    z-index: 10;
    background: rgba(0, 0, 0, 0.7);
    color: white;
    padding: 8px 12px;
    border-radius: 4px;
    font-size: 0.8rem;
    max-width: 40vw;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    pointer-events: none;
    backdrop-filter: blur(4px);
  }
</style>
