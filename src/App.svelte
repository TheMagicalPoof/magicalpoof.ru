<script>
  import { onMount } from "svelte";

  const baseStarCount = 30;
  const baseViewportArea = 1960 * 1080;
  const starDots = Array.from({ length: 9 }, (_, index) => index + 1);
  const hazeWidth = 84;
  const hazeHeight = 30;
  const hazeHues = [54, 176, 204, 112, 326, 24];
  const projects = [
    {
      title: "CAPS TABLE",
      url: "https://caps.magicalpoof.ru/",
      image: "/projects/caps-table-icon.png"
    },
    {
      title: "GLYPH",
      url: "https://glyph.magicalpoof.ru/",
      image: "/projects/glyph-icon.png"
    },
    {
      title: "CUTE PLANT",
      url: "https://t.me/cute_plant_bot",
      image: "/projects/cute-plant-icon.png"
    }
  ];

  let stars = [];
  let nextStarId = 0;
  let maxStars = baseStarCount;
  let spawnTimeout;
  let logoAnimationFrame;
  let logoHazeCanvas;
  let currentPage = "bio";

  function lerp(start, end, amount) {
    return start + amount * (end - start);
  }

  function fade(value) {
    return value * value * value * (value * (value * 6 - 15) + 10);
  }

  function smoothstep(edgeStart, edgeEnd, value) {
    const amount = Math.min(1, Math.max(0, (value - edgeStart) / (edgeEnd - edgeStart)));

    return amount * amount * (3 - 2 * amount);
  }

  function hashNoise(x, y, seed) {
    const value = Math.sin(x * 127.1 + y * 311.7 + seed * 74.7) * 43758.5453;

    return value - Math.floor(value);
  }

  function valueNoise(x, y, seed) {
    const x0 = Math.floor(x);
    const y0 = Math.floor(y);
    const localX = x - x0;
    const localY = y - y0;
    const top = lerp(hashNoise(x0, y0, seed), hashNoise(x0 + 1, y0, seed), fade(localX));
    const bottom = lerp(hashNoise(x0, y0 + 1, seed), hashNoise(x0 + 1, y0 + 1, seed), fade(localX));

    return lerp(top, bottom, fade(localY));
  }

  function layeredNoise(x, y, time) {
    return (
      valueNoise(x * 0.09 + time * 0.18, y * 0.09 - time * 0.13, 3.1) * 0.56 +
      valueNoise(x * 0.2 - time * 0.1, y * 0.2 + time * 0.11, 8.8) * 0.3 +
      valueNoise(x * 0.48 + time * 0.06, y * 0.48 - time * 0.05, 14.5) * 0.14
    );
  }

  function hslToRgb(hue, saturation, lightness) {
    const chroma = (1 - Math.abs(2 * lightness - 1)) * saturation;
    const huePrime = hue / 60;
    const x = chroma * (1 - Math.abs((huePrime % 2) - 1));
    const base =
      huePrime < 1 ? [chroma, x, 0] :
      huePrime < 2 ? [x, chroma, 0] :
      huePrime < 3 ? [0, chroma, x] :
      huePrime < 4 ? [0, x, chroma] :
      huePrime < 5 ? [x, 0, chroma] :
      [chroma, 0, x];
    const match = lightness - chroma / 2;

    return base.map((channel) => Math.round((channel + match) * 255));
  }

  function drawLogoHaze(timestamp) {
    if (!logoHazeCanvas) {
      logoAnimationFrame = window.requestAnimationFrame(drawLogoHaze);
      return;
    }

    if (logoHazeCanvas.width !== hazeWidth || logoHazeCanvas.height !== hazeHeight) {
      logoHazeCanvas.width = hazeWidth;
      logoHazeCanvas.height = hazeHeight;
    }

    const context = logoHazeCanvas.getContext("2d");
    const imageData = context.createImageData(hazeWidth, hazeHeight);
    const time = timestamp * 0.001;
    const driftX = Math.sin(time * 0.46) * 9 + Math.sin(time * 0.17 + 1.8) * 6;
    const driftY = Math.cos(time * 0.34) * 4 + Math.sin(time * 0.23 + 2.4) * 3;
    const pulse = 0.9 + valueNoise(time * 0.7, time * 0.43, 22.2) * 0.18;

    for (let y = 0; y < hazeHeight; y++) {
      for (let x = 0; x < hazeWidth; x++) {
        const index = (y * hazeWidth + x) * 4;
        const nx = x / (hazeWidth - 1);
        const ny = y / (hazeHeight - 1);
        const edgeFade =
          smoothstep(0, 0.14, nx) *
          smoothstep(1, 0.86, nx) *
          smoothstep(0, 0.16, ny) *
          smoothstep(1, 0.84, ny);
        const band = Math.exp(-Math.pow((ny - 0.5) / 0.29, 2));
        const wave = layeredNoise(x + driftX, y + driftY, time);
        const colorNoise = valueNoise(x * 0.16 - driftX * 0.24, y * 0.18 + driftY * 0.28, 31.7);
        const colorCycle = time * 0.12;
        const colorShift = Math.floor(colorCycle);
        const colorBlend = smoothstep(0, 1, colorCycle - colorShift);
        const patchPhase = valueNoise(x * 0.09 + 80, y * 0.09 - 40, 44.4);
        const hueIndex =
          (Math.floor(colorNoise * hazeHues.length) +
          colorShift +
          (patchPhase < colorBlend ? 1 : 0)) % hazeHues.length;
        const hue = hazeHues[hueIndex] + (wave - 0.5) * 10;
        const alpha = Math.round(edgeFade * band * (18 + wave * 66) * pulse);
        const [red, green, blue] = hslToRgb((hue + 360) % 360, 0.98, 0.54 + wave * 0.1);

        imageData.data[index] = red;
        imageData.data[index + 1] = green;
        imageData.data[index + 2] = blue;
        imageData.data[index + 3] = alpha;
      }
    }

    context.putImageData(imageData, 0, 0);
    logoAnimationFrame = window.requestAnimationFrame(drawLogoHaze);
  }

  function randomInteger(min, max) {
    return Math.floor(Math.random() * (max - min + 1)) + min;
  }

  function getMaxStars() {
    const viewportArea = window.innerWidth * window.innerHeight;
    const mobileDensity = window.innerWidth <= 640 ? 0.9 : 1;
    const starCount = Math.round((viewportArea / baseViewportArea) * baseStarCount * mobileDensity);

    return Math.max(2, Math.min(90, starCount));
  }

  function updateStarLimit() {
    maxStars = getMaxStars();
    stars = stars.slice(0, maxStars);
  }

  function getSpawnDelay() {
    if (maxStars <= 5) {
      return randomInteger(420, 1200);
    }

    if (maxStars <= 15) {
      return randomInteger(260, 760);
    }

    return randomInteger(180, 620);
  }

  function getSpawnAmount() {
    const amounts =
      maxStars <= 5
        ? [1, 1, 1, 2]
        : maxStars <= 15
          ? [1, 1, 1, 2]
          : [1, 1, 1, 1, 2, 2, 3];

    return amounts[randomInteger(0, amounts.length - 1)];
  }

  function createStar() {
    return {
      id: nextStarId++,
      frame: 1,
      top: Math.floor(Math.random() * window.innerHeight),
      left: Math.floor(Math.random() * window.innerWidth)
    };
  }

  function placeStars() {
    const freeSlots = maxStars - stars.length;

    if (freeSlots <= 0) {
      return;
    }

    const amount = Math.min(getSpawnAmount(), freeSlots);

    stars = [
      ...stars,
      ...Array.from({ length: amount }, createStar)
    ];
  }

  function scheduleStarSpawn() {
    spawnTimeout = window.setTimeout(() => {
      placeStars();
      scheduleStarSpawn();
    }, getSpawnDelay());
  }

  function updatePageFromHash() {
    const page = window.location.hash.replace("#", "");

    currentPage = ["projects", "shop", "links"].includes(page) ? page : "bio";
  }

  function animateStars() {
    stars = stars
      .map((star) => ({ ...star, frame: star.frame + 1 }))
      .filter((star) => star.frame <= 7);
  }

  onMount(() => {
    updateStarLimit();
    updatePageFromHash();

    const animationInterval = window.setInterval(animateStars, 235);
    scheduleStarSpawn();
    logoAnimationFrame = window.requestAnimationFrame(drawLogoHaze);

    window.addEventListener("resize", updateStarLimit);
    window.addEventListener("hashchange", updatePageFromHash);

    return () => {
      window.clearInterval(animationInterval);
      window.clearTimeout(spawnTimeout);
      window.cancelAnimationFrame(logoAnimationFrame);
      window.removeEventListener("resize", updateStarLimit);
      window.removeEventListener("hashchange", updatePageFromHash);
    };
  });
</script>

<svelte:head>
  <meta name="description" content="Magical PooF personal site" />
</svelte:head>

<main class="page" aria-label="Magical PooF">
  <header class="site-header">
    <nav class="site-nav" aria-label="Primary navigation">
      <a class:active={currentPage === "bio"} href="#bio">BIO</a>
      <a class:active={currentPage === "projects"} href="#projects">PROJECTS</a>
      <a class:active={currentPage === "shop"} href="#shop">SHOP</a>
      <a class:active={currentPage === "links"} href="#links">LINKS</a>
    </nav>
  </header>

  <div class="bg" aria-hidden="true">
    {#each stars as star (star.id)}
      <div
        class={`star frame-${star.frame}`}
        style={`top: ${star.top}px; left: ${star.left}px;`}
      >
        {#each starDots as dot}
          <div class={`dot dot-${dot}`}></div>
        {/each}
      </div>
    {/each}
  </div>

  {#if currentPage === "projects"}
    <section class="projects-page" aria-label="PROJECTS">
      <div class="projects-grid">
        {#each projects as project}
          <a class="project-card" href={project.url} target="_blank" rel="noreferrer">
            <img src={project.image} alt="" />
            <span>{project.title}</span>
          </a>
        {/each}
      </div>
    </section>
  {:else}
    <section class="card">
      <div class="logo-wrap">
        <canvas bind:this={logoHazeCanvas} class="logo-haze" aria-hidden="true"></canvas>
        <img class="mp" src="/media/magical-poof-logo.png" alt="Magical PooF" />
      </div>
      <div class="bio-copy" aria-label="Bio">
        <p>Engineer, programmer,</p>
        <p>digital gadget enjoyer,</p>
        <p>and certified kitty appreciator.</p>
      </div>
      <img class="license-preview" src="/media/hawaii-license-card.png" alt="Pixel Hawaii driver license card" />
      <div class="line"></div>
      <div class="line"></div>
    </section>
  {/if}
</main>
