<script>
  import { onMount } from 'svelte';

  let value = 6;
  let angle = 0;
  let hasOrientation = false;
  let hasMotion = false;
  let rolling = false;
  let mode = 'ready';
  let permissionMessage = '';
  let statusMessage = 'READY TO PLAY';
  let helperMessage = 'Hold it flat, face up for 6 · face down for 1';
  let toastMessage = '';
  let toastTimer;
  let lastShake = 0;
  let lastVector = null;
  let motionHits = 0;
  let rollTimer;

  const faces = {
    1: [4],
    2: [0, 8],
    3: [0, 4, 8],
    4: [0, 2, 6, 8],
    5: [0, 2, 4, 6, 8],
    6: [0, 2, 3, 5, 6, 8]
  };

  $: label = value === 6
    ? 'FACE UP · LUCKY SIX'
    : value === 1
      ? 'FACE DOWN · SNAKE EYES'
      : `TILT LOCKED · ${value} OF 6`;

  function showToast(message) {
    toastMessage = message;
    clearTimeout(toastTimer);
    toastTimer = setTimeout(() => (toastMessage = ''), 2600);
  }

  function setMode(nextMode, message) {
    mode = nextMode;
    statusMessage = nextMode === 'live'
      ? 'SENSORS LIVE'
      : nextMode === 'demo'
        ? 'DEMO MODE'
        : nextMode === 'denied'
          ? 'SENSORS BLOCKED'
          : 'READY TO PLAY';
    helperMessage = message;
  }

  function updateOrientation(event) {
    if (Number.isFinite(event.beta) && Number.isFinite(event.gamma)) {
      const beta = (event.beta * Math.PI) / 180;
      const gamma = (event.gamma * Math.PI) / 180;
      const normalZ = Math.cos(beta) * Math.cos(gamma);
      angle = (Math.acos(Math.max(-1, Math.min(1, normalZ))) * 180) / Math.PI;
      hasOrientation = true;
    }
  }

  function updateMotion(event) {
    hasMotion = true;
    const acceleration = event.acceleration;
    let strength = 0;

    if (acceleration && [acceleration.x, acceleration.y, acceleration.z].some(Number.isFinite)) {
      strength = Math.hypot(acceleration.x || 0, acceleration.y || 0, acceleration.z || 0);
    } else {
      const gravity = event.accelerationIncludingGravity;
      if (gravity && [gravity.x, gravity.y, gravity.z].every(Number.isFinite)) {
        const vector = { x: gravity.x, y: gravity.y, z: gravity.z };
        if (lastVector) {
          strength = Math.hypot(
            vector.x - lastVector.x,
            vector.y - lastVector.y,
            vector.z - lastVector.z
          );
        }
        lastVector = vector;
      }
    }

    if (strength > 12) {
      motionHits += 1;
      if (motionHits >= 2) roll(true);
    } else {
      motionHits = Math.max(0, motionHits - 1);
    }
  }

  function setValue(nextValue) {
    value = nextValue;
  }

  function roll(fromShake = false) {
    const now = Date.now();
    if (rolling || now - lastShake < 850) return;

    lastShake = now;
    rolling = true;
    const target = hasOrientation && mode === 'live'
      ? Math.max(1, Math.min(6, 6 - Math.floor((angle + 3) / 36)))
      : Math.floor(Math.random() * 6) + 1;

    clearTimeout(rollTimer);
    rollTimer = setTimeout(() => {
      setValue(target);
      rolling = false;
      if (fromShake) showToast('Shake detected — your number is locked in');
    }, hasOrientation && mode === 'live' ? 950 : 1100);
  }

  async function enableSensors() {
    if (!window.isSecureContext && location.hostname !== 'localhost') {
      setMode('denied', 'Motion sensors need a secure HTTPS connection.');
      permissionMessage = 'Open your secure Vercel link to use motion sensors.';
      return;
    }

    permissionMessage = 'Allow motion access in the Safari prompt.';
    try {
      const permissions = [];
      if (typeof DeviceMotionEvent !== 'undefined' && typeof DeviceMotionEvent.requestPermission === 'function') {
        permissions.push(DeviceMotionEvent.requestPermission());
      }
      if (typeof DeviceOrientationEvent !== 'undefined' && typeof DeviceOrientationEvent.requestPermission === 'function') {
        permissions.push(DeviceOrientationEvent.requestPermission());
      }

      const results = await Promise.all(permissions);
      if (results.includes('denied')) throw new Error('denied');

      window.addEventListener('deviceorientation', updateOrientation, { passive: true });
      window.addEventListener('devicemotion', updateMotion, { passive: true });

      setTimeout(() => {
        if (hasMotion || hasOrientation) {
          setMode('live', 'Tilt is captured on each shake and stays locked.');
          permissionMessage = '';
        } else {
          setMode('demo', 'No sensor signal here · tap to roll a random number.');
          permissionMessage = '';
        }
      }, 1600);
    } catch (error) {
      setMode('denied', 'Tap Enable sensors to try again.');
      permissionMessage = error.message === 'denied'
        ? 'Motion access was denied. Allow it in Safari website settings, then try again.'
        : 'Could not start motion sensors. Tap to try again.';
    }
  }

  function handleRoll() {
    if (mode === 'ready' || mode === 'denied') {
      enableSensors();
      return;
    }
    roll();
  }

  onMount(() => {
    const hasSensorApi = 'DeviceOrientationEvent' in window || 'DeviceMotionEvent' in window;
    if (!hasSensorApi) {
      setMode('demo', 'No motion sensors here · click to roll a random number.');
    } else if (
      typeof window.DeviceMotionEvent?.requestPermission === 'function' ||
      typeof window.DeviceOrientationEvent?.requestPermission === 'function'
    ) {
      permissionMessage = 'Safari needs your okay to use motion sensors.';
    } else {
      setMode('ready', 'Tap start, then shake to roll.');
    }

    

    return () => {
      window.removeEventListener('deviceorientation', updateOrientation);
      window.removeEventListener('devicemotion', updateMotion);
      clearTimeout(toastTimer);
      clearTimeout(rollTimer);
    };
  });
</script>

<svelte:head>
  <title>ROLL — The motion dice</title>
  <meta name="description" content="A pocket-sized, motion-powered digital dice." />
  <meta name="theme-color" content="#090a17" />
</svelte:head>

<div class="stars" aria-hidden="true"></div>
<main class="app">
  <header>
    <div class="brand">R<span>O</span>LL</div>
    <div class="badge"><i class:live={mode === 'live'} class:demo={mode === 'demo'} class="dot"></i>{statusMessage}</div>
  </header>

  <section class="intro">
    <div class="eyebrow">A little chance. A lot of magic.</div>
    <h1>Let gravity<br />make the call.</h1>
    <p>Shake your phone. Feel the roll.<br />See where it lands.</p>
  </section>

  <div class:rolling class="stage" aria-label="A six-sided dice">
    <div class="halo"></div><div class="ring"></div><div class="ring two"></div>
    <div class="die-wrap">
      <div class="die" role="img" aria-label={`Dice showing ${value}`}>
        {#each Array(9) as _, index}
          <i class="pip" class:on={faces[value].includes(index)}></i>
        {/each}
      </div>
    </div>
  </div>

  <div class="number" aria-live="polite">{value}</div>
  <div class="number-label">{rolling ? 'THE UNIVERSE IS DECIDING' : label}</div>

  <div class="controls">
    {#if permissionMessage}
      <div class="permission">{permissionMessage} <button on:click={enableSensors}>Enable sensors</button></div>
    {/if}
    <button class="roll" on:click={handleRoll}>
      <span class="button-content"><span class="shake-icon">✦</span><span>{rolling ? 'ROLLING…' : 'SHAKE TO ROLL'}</span></span>
    </button>
    <p class="helper">{helperMessage}</p>
  </div>

  <div class="sensor-note"><i class:live={mode === 'live'} class:demo={mode === 'demo'} class="dot"></i>
    <span>{mode === 'live' ? 'Motion sensing · shake to roll' : mode === 'demo' ? 'Demo mode · tap to roll' : 'Motion sensing · waiting to start'}</span>
  </div>
  <div class="footer">Trust the tilt <span>✧</span> trust the tumble</div>
</main>

{#if toastMessage}<div class="toast" role="status">{toastMessage}</div>{/if}

<style>
  :global(*){box-sizing:border-box}:global(html){min-height:100%;background:#090a17}:global(body){margin:0;min-height:100svh;color:#f6f4ff;font-family:Inter,-apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif;overflow-x:hidden;background:radial-gradient(ellipse at 50% 42%,#20204a 0,#111329 34%,#090a17 75%)}
  .stars{position:fixed;inset:0;pointer-events:none;opacity:.65;background-image:radial-gradient(1px 1px at 10% 18%,#fff 50%,transparent),radial-gradient(1px 1px at 77% 16%,#a8f5ff 50%,transparent),radial-gradient(1px 1px at 90% 68%,#fff 50%,transparent),radial-gradient(1.5px 1.5px at 18% 76%,#d8c4ff 50%,transparent),radial-gradient(1px 1px at 61% 84%,#fff 50%,transparent),radial-gradient(1px 1px at 32% 31%,#fff 50%,transparent);animation:twinkle 5s ease-in-out infinite alternate}
  .app{position:relative;z-index:1;max-width:560px;min-height:100svh;margin:auto;padding:calc(env(safe-area-inset-top) + 24px) 24px calc(env(safe-area-inset-bottom) + 22px);display:flex;flex-direction:column;align-items:center;text-align:center}
  header{width:100%;display:flex;align-items:center;justify-content:space-between}.brand{font-size:14px;font-weight:850;letter-spacing:.34em}.brand span{color:#69e7ff}.badge{display:flex;align-items:center;gap:8px;padding:8px 12px;border:1px solid #ffffff1f;border-radius:99px;background:#ffffff09;font-size:10px;letter-spacing:.12em;color:#cfcee2}.dot{width:7px;height:7px;flex:none;border-radius:50%;background:#9993ad;box-shadow:0 0 12px currentColor}.dot.live{background:#63f2c0;color:#63f2c0}.dot.demo{background:#ffc77d;color:#ffc77d}
  .intro{margin-top:clamp(30px,7vh,64px)}.eyebrow{font-size:10px;text-transform:uppercase;letter-spacing:.3em;color:#bbb4ee}.intro h1{font-size:clamp(29px,8vw,48px);line-height:1.04;letter-spacing:-.055em;margin:12px 0 10px;font-weight:760}.intro p{margin:0;color:#aaa9c3;font-size:14px;line-height:1.55}
  .stage{width:min(76vw,300px);aspect-ratio:1;margin:clamp(26px,5vh,50px) 0 14px;position:relative;display:grid;place-items:center;perspective:1000px}.halo{position:absolute;inset:7%;border-radius:50%;background:conic-gradient(from 15deg,transparent,#b38aff 16%,#72e8ff 34%,transparent 53%,#ff79cb 72%,transparent);filter:blur(20px);opacity:.25;animation:spin 18s linear infinite}.ring{position:absolute;inset:0;border:1px solid #d3caff2e;border-radius:50%;transform:rotateX(69deg) rotateZ(-20deg);box-shadow:0 0 30px #9f88ff1f,inset 0 0 25px #70dbff12}.ring:after{content:"";position:absolute;top:7%;left:14%;width:8px;height:8px;border-radius:50%;background:#a5f6ff;box-shadow:0 0 18px 6px #80dfff}.ring.two{inset:11%;transform:rotateX(69deg) rotateZ(55deg);opacity:.55;border-color:#ff79cb30}
  .die-wrap{position:relative;z-index:2;width:61%;aspect-ratio:1;display:grid;place-items:center;filter:drop-shadow(0 25px 28px #0007);transition:transform .45s cubic-bezier(.2,.85,.3,1)}.die{width:100%;height:100%;position:relative;border-radius:23%;background:linear-gradient(145deg,#fbfaff 0%,#e6e3fa 45%,#b9b6d4 100%);box-shadow:inset 8px 8px 17px #ffffffeB,inset -10px -12px 22px #52487d40,0 0 35px #ae9cff61;display:grid;grid-template:repeat(3,1fr)/repeat(3,1fr);padding:16%;gap:5%;transform:rotate(-9deg);transition:transform .35s,filter .35s}.die:after{content:"";position:absolute;inset:2%;border-radius:21%;border:1px solid #ffffffb8;pointer-events:none}.pip{width:100%;aspect-ratio:1;align-self:center;justify-self:center;border-radius:50%;background:radial-gradient(circle at 35% 30%,#777199,#37324f 74%);box-shadow:inset 1px 2px 3px #0006,0 1px 1px #fff;opacity:0;transform:scale(.4);transition:opacity .18s,transform .25s cubic-bezier(.2,.8,.2,1)}.pip.on{opacity:1;transform:scale(1)}.die-wrap:before{content:"";position:absolute;width:120%;height:18%;border-radius:50%;background:#9d80ff;filter:blur(27px);opacity:.3;bottom:-17%;z-index:-1}.rolling .die{animation:tumble .56s cubic-bezier(.4,0,.6,1) infinite;filter:brightness(1.08)}.rolling .die-wrap{animation:float .75s ease-in-out infinite alternate}
  .number{font-size:clamp(66px,20vw,102px);line-height:.95;font-weight:780;letter-spacing:-.09em;text-shadow:0 0 35px #ae9cff4d;font-variant-numeric:tabular-nums}.number-label{font-size:11px;letter-spacing:.24em;text-transform:uppercase;color:#c8c5df;margin-top:7px;min-height:16px}.controls{width:100%;margin-top:28px}.roll{position:relative;overflow:hidden;border:0;width:100%;min-height:58px;border-radius:18px;color:#101020;background:linear-gradient(100deg,#b8a4ff,#93eaff 58%,#ffc0e8);font:inherit;font-size:15px;font-weight:780;letter-spacing:.02em;box-shadow:0 10px 35px #8076ff3d,inset 0 1px #ffffffcc;cursor:pointer;transition:transform .18s,filter .18s}.roll:before{content:"";position:absolute;inset:0;transform:translateX(-110%);background:linear-gradient(100deg,transparent,#ffffff80,transparent);animation:sheen 4s ease-in-out infinite}.roll:active{transform:scale(.975);filter:brightness(1.1)}.button-content{position:relative;z-index:1;display:flex;justify-content:center;align-items:center;gap:10px}.shake-icon{font-size:19px}.permission{width:100%;padding:0 2px;margin-bottom:10px;color:#d4d1e8;font-size:12px;line-height:1.45}.permission button{background:none;border:0;padding:0;color:#a9eaff;font:inherit;font-weight:700;text-decoration:underline;text-underline-offset:3px;cursor:pointer}.helper{font-size:11px;color:#9291ab;margin:12px 0 0;line-height:1.45}.sensor-note{margin-top:20px;display:flex;align-items:center;justify-content:center;gap:8px;min-height:18px;font-size:11px;color:#aaa8c1}.footer{margin-top:auto;padding-top:22px;font-size:10px;letter-spacing:.16em;text-transform:uppercase;color:#77768e}.footer span{color:#b8a5ff}.toast{position:fixed;z-index:5;left:50%;bottom:calc(env(safe-area-inset-bottom) + 18px);transform:translate(-50%,0);padding:11px 16px;border:1px solid #ffffff24;border-radius:99px;background:#17172be8;backdrop-filter:blur(18px);font-size:12px;color:#eee;white-space:nowrap;animation:fade .25s ease-out}
  @keyframes spin{to{transform:rotate(360deg)}}@keyframes twinkle{to{opacity:.32}}@keyframes tumble{0%{transform:rotate(-9deg) rotateX(0) rotateY(0)}50%{transform:rotate(170deg) rotateX(190deg) rotateY(80deg)}100%{transform:rotate(351deg) rotateX(360deg) rotateY(360deg)}}@keyframes float{to{transform:translateY(-13px) scale(1.03)}}@keyframes sheen{0%,65%{transform:translateX(-110%)}88%,100%{transform:translateX(110%)}}@keyframes fade{from{opacity:0;transform:translate(-50%,15px)}to{opacity:1;transform:translate(-50%,0)}}
  @media(min-height:850px){.stage{margin-top:46px}.controls{margin-top:36px}}@media(max-height:690px){.app{padding-top:calc(env(safe-area-inset-top) + 15px)}.intro{margin-top:22px}.stage{width:min(60vw,230px);margin-top:18px}.controls{margin-top:14px}.footer{padding-top:14px}}@media(prefers-reduced-motion:reduce){:global(*),:global(*:before),:global(*:after){animation-duration:.01ms!important;animation-iteration-count:1!important;transition-duration:.01ms!important}}
</style>
