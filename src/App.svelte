<script lang="ts">
  import p5 from 'p5';

  const sketch = (p5: p5) => {
    p5.setup = () => {
      p5.createCanvas(window.innerWidth, window.innerHeight);
      p5.background(32); // Set a dark background at the start
    };

    let size = 30; // Initial size of the circle
    let growing = true; // Flag to determine if the circle is growing or shrinking
    const growthRate = 1; // Increased rate at which the circle grows or shrinks for bigger steps
    const minSize = 30; // Minimum size of the circle
    const maxSize = 200; // Maximum size of the circle
    let hue = 0; // Initialize hue for color changes
    let xoff = 0; // Offset for x-coordinate in noise function
    let yoff = 10000; // Offset for y-coordinate in noise function (different to ensure variety)
    // Increase the step size for more movement
    let stepSize = 0.01; // Adjusted step size for more significant movement across the canvas

    let x = 0;
    let y = 0;

    p5.draw = () => {
      // Calculate step size based on circle size and add a small random variation
      const baseStepSize = 2; // Base step size
      const sizeInfluence = (size - minSize) * 0.1; // Influence of circle size on step size
      const randomVariation = p5.random(-0.05, 0.05); // Random variation to add to the step size
      stepSize = baseStepSize + sizeInfluence + randomVariation;

      let alpha = 1; // Alpha value for color transparency
      p5.stroke('#333');
      // p5.noStroke();

      p5.colorMode(p5.HSB, 360, 100, 100, 1); // Set color mode to HSB for hue manipulation
      p5.fill(hue, 100, 100, alpha); // Use the hue variable for color, with full saturation and brightness
      // p5.fill('#fff');

      // To make movement smoother and more all over the place, we adjust the noise scale and step size
      // Increase the noise scale for broader, smoother movement across the canvas
      let noiseScale = 0.001; // Adjusted noise scale for broader movement
      let targetX = p5.map(p5.noise(xoff * noiseScale), 0, 1, 0, p5.width); // Broader noise scale for x
      let targetY = p5.map(p5.noise(yoff * noiseScale), 0, 1, 0, p5.height); // Broader noise scale for y
      x = p5.lerp(x, targetX, 0.1); // Use a faster lerp rate for smoother transitions
      y = p5.lerp(y, targetY, 0.1); // Use a faster lerp rate for smoother transitions

      if (p5.frameCount % 2 === 0) {
        p5.circle(x, y, size); // Draw circle at new, smoothly updated position
        return;
      }

      // Toggle growing state based on size boundaries
      if ((growing && size >= maxSize) || (!growing && size <= minSize)) {
        growing = !growing;
      }
      // Adjust size based on growing state with bigger steps
      size += growing ? growthRate : -growthRate;
      hue = (hue + 1) % 360; // Cycle hue through the color spectrum

      // Increment offsets for Perlin noise to move the circle with more pronounced steps
      xoff += stepSize * 2; // Double the step size for x offset to increase movement variation
      yoff += stepSize * 2; // Double the step size for y offset to increase movement variation
    };
  };

  new p5(sketch);
</script>

<main></main>
