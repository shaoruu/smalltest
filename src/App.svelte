<script lang="ts">
  import { onMount } from 'svelte';
  import p5 from 'p5';
  import * as THREE from 'three';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

  // add passable options for different factors for alignment, cohesion, separation, etc
  class Boid {
    position: p5.Vector;
    velocity: p5.Vector;
    acceleration: p5.Vector;
    maxForce: number;
    maxSpeed: number;
    perceptionRadius: number;
    alignmentStrength: number;
    cohesionStrength: number;
    separationStrength: number;

    constructor(
      private p: p5,
      options: {
        maxForce?: number;
        maxSpeed?: number;
        perceptionRadius?: number;
        alignmentStrength?: number;
        cohesionStrength?: number;
        separationStrength?: number;
      } = {},
    ) {
      this.position = p.createVector(
        Math.random() * p.width,
        Math.random() * p.height,
      );
      this.velocity = p5.Vector.random2D();
      this.velocity.setMag(Math.random() * 2);
      this.acceleration = p.createVector();
      this.maxForce = options.maxForce || 0.2;
      this.maxSpeed = options.maxSpeed || 4;
      this.perceptionRadius = options.perceptionRadius || 50;
      this.alignmentStrength = options.alignmentStrength || 1;
      this.cohesionStrength = options.cohesionStrength || 1;
      this.separationStrength = options.separationStrength || 1;
    }

    edges() {
      if (this.position.x > this.p.width) {
        this.position.x = 0;
      } else if (this.position.x < 0) {
        this.position.x = this.p.width;
      }
      if (this.position.y > this.p.height) {
        this.position.y = 0;
      } else if (this.position.y < 0) {
        this.position.y = this.p.height;
      }
    }

    align(boids: Boid[]) {
      let steering = this.p.createVector();
      let total = 0;
      for (let other of boids) {
        let d = this.p.dist(
          this.position.x,
          this.position.y,
          other.position.x,
          other.position.y,
        );
        if (other != this && d < this.perceptionRadius) {
          steering.add(other.velocity);
          total++;
        }
      }
      if (total > 0) {
        steering.div(total);
        steering.setMag(this.maxSpeed);
        steering.sub(this.velocity);
        steering.limit(this.maxForce);
      }
      return steering.mult(this.alignmentStrength);
    }

    cohesion(boids: Boid[]) {
      let steering = this.p.createVector();
      let total = 0;
      for (let other of boids) {
        let d = this.p.dist(
          this.position.x,
          this.position.y,
          other.position.x,
          other.position.y,
        );
        if (other != this && d < this.perceptionRadius) {
          steering.add(other.position);
          total++;
        }
      }
      if (total > 0) {
        steering.div(total);
        steering.sub(this.position);
        steering.setMag(this.maxSpeed);
        steering.sub(this.velocity);
        steering.limit(this.maxForce);
      }
      return steering.mult(this.cohesionStrength);
    }

    separation(boids: Boid[]) {
      let steering = this.p.createVector();
      let total = 0;
      for (let other of boids) {
        let d = this.p.dist(
          this.position.x,
          this.position.y,
          other.position.x,
          other.position.y,
        );
        if (other != this && d < this.perceptionRadius) {
          let diff = p5.Vector.sub(this.position, other.position);
          diff.normalize();
          diff.div(d); // Weight by distance
          steering.add(diff);
          total++;
        }
      }
      if (total > 0) {
        steering.div(total);
        steering.setMag(this.maxSpeed);
        steering.sub(this.velocity);
        steering.limit(this.maxForce);
      }
      return steering.mult(this.separationStrength);
    }

    flock(boids: Boid[]) {
      let alignment = this.align(boids);
      this.acceleration.add(alignment);
      let cohesion = this.cohesion(boids);
      this.acceleration.add(cohesion);
      let separation = this.separation(boids);
      this.acceleration.add(separation);
    }

    update() {
      this.position.add(this.velocity);
      this.velocity.add(this.acceleration);
      this.velocity.limit(this.maxSpeed);
      this.acceleration.mult(0);
    }

    show() {
      this.p.strokeWeight(1); // Adjusted stroke weight to be thinner
      this.p.stroke('#FFEC9E'); // Changed stroke color to #FFEC9E
      let angle = this.velocity.heading(); // Calculated the angle of the velocity
      this.p.push(); // Saved the current drawing settings
      this.p.translate(this.position.x, this.position.y); // Moved to the boid's position
      this.p.rotate(angle); // Rotated to the velocity's angle
      this.p.line(0, 0, 10, 0); // Main line of the arrow
      this.p.line(10, 0, 5, 5); // Bottom part of the arrow head
      this.p.line(10, 0, 5, -5); // Top part of the arrow head
      this.p.pop(); // Restored the previous drawing settings
    }
  }

  onMount(() => {
    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(
      75,
      window.innerWidth / window.innerHeight,
    );
    camera.position.z = 3;
    camera.lookAt(0, 0, 0);

    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setClearColor('#FFFBDA'); // Set renderer background to gray
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.outputColorSpace = THREE.SRGBColorSpace;
    document.body.appendChild(renderer.domElement);

    const canvasToDraw = document.createElement('canvas');
    canvasToDraw.width = 400;
    canvasToDraw.height = 400;
    const canvasTexture = new THREE.CanvasTexture(canvasToDraw);
    canvasTexture.magFilter = THREE.NearestFilter;
    canvasTexture.minFilter = THREE.NearestFilter;
    canvasTexture.colorSpace = THREE.SRGBColorSpace;

    const plane = new THREE.Mesh(
      new THREE.BoxGeometry(1, 1, 1),
      new THREE.MeshBasicMaterial({
        map: canvasTexture,
        side: THREE.DoubleSide,
      }),
    );

    scene.add(plane);

    const controls = new OrbitControls(camera, renderer.domElement);

    const instance = new p5((p) => {
      const boids: Boid[] = [];

      p.setup = () => {
        p.createCanvas(400, 400, canvasToDraw);

        for (let i = 0; i < 100; i++) {
          boids.push(new Boid(p));
        }
      };

      p.draw = () => {
        p.background('#322C2B');

        // p.noStroke();
        // p.rectMode(p.CENTER);
        // p.fill(p.random(255), p.random(255), p.random(255));
        // p.rect(
        //   p.random(p.width),
        //   p.random(p.height),
        //   p.random(p.height * 0.1, p.height * 0.5),
        // );

        boids.forEach((boid) => {
          boid.flock(boids);
          boid.update();
          boid.edges();
          boid.show();
        });

        canvasTexture.needsUpdate = true;
      };
    });

    let frame: any;

    const animate = () => {
      frame = requestAnimationFrame(animate);
      renderer.render(scene, camera);
      controls.update();
    };

    animate();

    return () => {
      instance.remove();
      renderer.dispose();
      document.body.removeChild(renderer.domElement);
      cancelAnimationFrame(frame);
    };
  });
</script>

<main></main>

<style>
  html,
  body {
    width: 100vw;
    height: 100vh;
    overflow: hidden;
    margin: 0;
    padding: 0;
  }
</style>
