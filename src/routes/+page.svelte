
<script>
  import { onMount } from 'svelte';

  let alpha = $state(null);
  let beta = $state(null);
  let gamma = $state(null);
  let message = $state('Click the button to start.');

  let listening = false;

  function handleOrientation(event) {
    alpha = event.alpha;
    beta = event.beta;
    gamma = event.gamma;
  }

  async function startOrientation() {
    if (!('DeviceOrientationEvent' in window)) {
      message = 'Device orientation is not supported.';
      return;
    }

    try {
      // Required by iOS Safari
      if (
        typeof DeviceOrientationEvent.requestPermission === 'function'
      ) {
        const permission =
          await DeviceOrientationEvent.requestPermission();

        if (permission !== 'granted') {
          message = 'Permission denied.';
          return;
        }
      }

      window.addEventListener(
        'deviceorientation',
        handleOrientation
      );

      listening = true;
      message = 'Reading device orientation...';
    } catch (error) {
      message = 'Could not access device orientation.';
      console.error(error);
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

<h1>Device Orientation</h1>

<button onclick={startOrientation} disabled={listening}>
  Start orientation
</button>

<p>{message}</p>
<p>Alpha: {alpha ?? 'Waiting for data...'}</p>
<p>Beta: {beta ?? 'Waiting for data...'}</p>
<p>Gamma: {gamma ?? 'Waiting for data...'}</p>
