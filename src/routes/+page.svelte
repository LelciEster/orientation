<script>
    import { onMount } from 'svelte';
 
    let alpha = $state(null);
    let beta = $state(null);
    let gamma = $state(null);
    let status = $state('Sensor noch nicht gestartet');
 
    function updateOrientation(event) {
        alpha = event.alpha;
        beta = event.beta;
        gamma = event.gamma;
    }
 
    async function startSensor() {
        if (!window.DeviceOrientationEvent) {
            status = 'Device orientation not supported';
            return;
        }
 
        // Berechtigung für iPhone
        if (typeof DeviceOrientationEvent.requestPermission === 'function') {
            try {
                const permission = await DeviceOrientationEvent.requestPermission();
 
                if (permission !== 'granted') {
                    status = 'Permission denied';
                    return;
                }
            } catch (error) {
                status = 'Permission error';
                return;
            }
        }
 
        window.removeEventListener('deviceorientation', updateOrientation);
        window.addEventListener('deviceorientation', updateOrientation);
 
        status = 'Waiting for sensor data...';
    }
 
    onMount(() => {
        return () => {
            window.removeEventListener('deviceorientation', updateOrientation);
        };
    });
</script>
 
<h1>Device Orientation</h1>
 
<button onclick={startSensor}>Start Sensor</button>
 
<p>{status}</p>
 
<p>Alpha: {alpha}</p>
<p>Beta: {beta}</p>
<p>Gamma: {gamma}</p>
 