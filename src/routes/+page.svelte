<script>
  import { onMount } from 'svelte';

    let alpha = $state(null);
    let beta = $state(null);
    let gamma = $state(null);

onMount(() => {
    if (window.DeviceOrientationEvent) {
        window.addEventListener('deviceorientation', (event) => {
            alpha = event.alpha;
            beta = event.beta;
            gamma = event.gamma;
        });
    } else {
        console.log("Device orientation not supported");
    }
});
</script>

<h1>device orientierung</h1>
    
<p>Alpha: {alpha}</p>
<p>Beta: {beta}</p>
<p>Gamma: {gamma}</p>