
<script>
  import { onMount } from 'svelte';

  let alpha = $state(null);
  let beta = $state(null);
  let gamma = $state(null);
  let message = $state('Press the button to enable orientation.');

  let listening = false;

  function handleOrientation(event) {
    alpha = event.alpha;
    beta = event.beta;
    gamma = event.gamma;

    if (
      event.alpha === null &&
      event.beta === null &&
      event.gamma === null
    ) {
      message = 'No orientation data received.';
    } else {
      message = 'Orientation is working!';
    }
  }

  async function startOrientation() {
    message = 'Checking device support...';

    if (!window.isSecureContext) {
      message = 'Error: Open this website using HTTPS.';
      return;
    }

    if (!('DeviceOrientationEvent' in window)) {
      message = 'Error: This browser does not support orientation.';
      return;
    }

    try {
      // iOS Safari requires permission after a button click.
      if (
        typeof DeviceOrientationEvent.requestPermission ===
        'function'
      ) {
        message = 'Requesting motion permission...';

        const permission =
          await DeviceOrientationEvent.requestPermission();

        if (permission !== 'granted') {
          message = 'Permission denied. Check Safari settings.';
          return;
        }
      }

      window.addEventListener(
        'deviceorientation',
        handleOrientation
      );

      listening = true;
      message = 'Permission granted. Waiting for sensor data...';
    } catch (error) {
      console.error(error);
      message = `Error: ${error.message}`;
    }
  }

  onMount(() => {
    return () => {
      window.removeEventListener(
        'deviceorientation',
        handleOrientation
      );
    };
  });
</script>

<svelte:head>
  <title>Device Orientation Test</title>
  <meta
    name="viewport"
    content="width=device-width, initial-scale=1"
  />
</svelte:head>

<h1>Device Orientation</h1>

<button onclick={startOrientation} disabled={listening}>
  Enable Orientation
</button>

<p><strong>Status:</strong> {message}</p>

<p>Alpha: {alpha ?? 'Waiting...'}</p>
<p>Beta: {beta ?? 'Waiting...'}</p>
<p>Gamma: {gamma ?? 'Waiting...'}</p>
