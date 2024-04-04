<script lang="ts">
  import p5, { Vector } from 'p5';
  import { onMount } from 'svelte';

  const sketch = (p: p5) => {
    const boidsCount = 100;
    const boids: Boid[] = [];

    p.setup = () => {
      p.createCanvas(p.windowWidth, p.windowHeight);

      for (let i = 0; i < boidsCount; i++) {
        boids.push(new Boid(p));
      }
    };

    p.mousePressed = () => {
      boids.push(
        new Boid(p, { defaultPosition: { x: p.mouseX, y: p.mouseY } }),
      );
    };

    p.draw = () => {
      p.background(0);

      for (let boid of boids) {
        boid.update(boids, p);
        boid.draw(p);
      }
    };
  };

  onMount(() => {
    const instance = new p5(sketch);
    return () => {
      instance.remove();
    };
  });

  class Boid {
    position: p5.Vector;
    velocity: p5.Vector;
    acceleration: p5.Vector;

    maxSpeed: number;
    maxForce: number;
    perceptionRadius: number;

    separationFactor: number;
    alignmentFactor: number;
    cohesionFactor: number;
    boundingFactor: number;
    avoidMouseFactor: number;

    debugForces: { force: p5.Vector; color: string }[];

    constructor(
      p5: p5,
      {
        maxSpeed = 5,
        maxForce = 0.1,
        perceptionRadius = 50,
        separationFactor = 0.05,
        alignmentFactor = 0.05,
        cohesionFactor = 0.05,
        boundingFactor = 0.3,
        avoidMouseFactor = 0.5,
        defaultPosition = { x: p5.windowWidth / 2, y: p5.windowHeight / 2 },
      } = {},
    ) {
      this.position = p5.createVector(defaultPosition.x, defaultPosition.y);
      this.velocity = p5
        .createVector(p5.random(-1, 1), p5.random(-1, 1))
        .setMag(maxSpeed);
      this.acceleration = p5.createVector(0, 0);

      this.maxSpeed = maxSpeed;
      this.maxForce = maxForce;
      this.perceptionRadius = perceptionRadius;

      this.separationFactor = separationFactor;
      this.alignmentFactor = alignmentFactor;
      this.cohesionFactor = cohesionFactor;
      this.boundingFactor = boundingFactor;
      this.avoidMouseFactor = avoidMouseFactor;

      // Debugging info
      this.debugForces = [];
    }

    update(boids: Boid[], p5: p5): void {
      let alignmentForce = this.alignment(boids, p5);
      let cohesionForce = this.cohesion(boids, p5);
      let separationForce = this.separation(boids, p5);
      let boundingForce = this.bounding(p5);
      let avoidMouseForce = this.avoidMouse(p5);

      this.acceleration.mult(0); // Reset acceleration to 0 before applying forces
      this.applyForce(alignmentForce);
      this.applyForce(cohesionForce);
      this.applyForce(separationForce);
      this.applyForce(boundingForce);
      this.applyForce(avoidMouseForce);

      this.velocity.add(this.acceleration);
      this.velocity.limit(this.maxSpeed);
      this.position.add(this.velocity);

      // Save forces for debugging
      this.debugForces = [
        { force: alignmentForce, color: '#FF6347' }, // Tomato
        { force: cohesionForce, color: '#4682B4' }, // SteelBlue
        { force: separationForce, color: '#32CD32' }, // LimeGreen
        { force: boundingForce, color: '#FFD700' }, // Gold
        { force: avoidMouseForce, color: '#9932CC' }, // DarkOrchid
      ];
    }

    applyForce(force: p5.Vector): void {
      this.acceleration.add(force);
    }

    draw(p5: p5): void {
      p5.stroke(255);
      p5.strokeWeight(2);
      p5.fill(255);
      p5.push();
      p5.translate(this.position.x, this.position.y);

      const trianglePoints = [
        p5.createVector(10, 0), // top point of the triangle
        p5.createVector(-5, -5), // bottom left point of the triangle
        p5.createVector(-5, 5), // bottom right point of the triangle
      ];

      p5.push();
      p5.rotate(this.velocity.heading()); // Correct the heading direction
      p5.beginShape();
      for (let point of trianglePoints) {
        p5.vertex(point.x, point.y);
      }
      p5.endShape(p5.CLOSE);
      p5.pop();

      // Draw debugging info
      for (let debugForce of this.debugForces) {
        p5.stroke(debugForce.color);
        const factor = 50;
        p5.line(
          0,
          0,
          debugForce.force.x * factor, // Corrected direction for debugging lines
          debugForce.force.y * factor,
        );
      }

      p5.pop();
    }

    avoidMouse = (p5: p5): Vector => {
      let mouse = p5.createVector(p5.mouseX, p5.mouseY);
      let steer = Vector.sub(this.position, mouse);
      if (steer.mag() < this.perceptionRadius) {
        // Only avoid mouse if within perception radius
        steer.setMag(this.maxSpeed);
        steer.mult(this.avoidMouseFactor);
        steer.limit(this.maxForce);
        return steer;
      }
      return p5.createVector(0, 0);
    };

    bounding = (p5: p5): Vector => {
      let desired = null;
      let edgeDistanceX = Math.min(this.position.x, p5.width - this.position.x);
      let edgeDistanceY = Math.min(
        this.position.y,
        p5.height - this.position.y,
      );

      if (edgeDistanceX < this.perceptionRadius) {
        desired = p5.createVector(
          this.position.x < p5.width / 2 ? this.maxSpeed : -this.maxSpeed,
          this.velocity.y,
        );
      }

      if (edgeDistanceY < this.perceptionRadius) {
        if (desired === null) {
          desired = p5.createVector(
            this.velocity.x,
            this.position.y < p5.height / 2 ? this.maxSpeed : -this.maxSpeed,
          );
        } else {
          desired.add(
            p5.createVector(
              this.velocity.x,
              this.position.y < p5.height / 2 ? this.maxSpeed : -this.maxSpeed,
            ),
          );
          desired.div(2); // Average the desired vector if both conditions are met
        }
      }

      if (desired !== null) {
        desired.normalize();
        desired.mult(this.maxSpeed);
        let steer = Vector.sub(desired, this.velocity);
        steer.mult(this.boundingFactor);
        steer.limit(this.maxForce);
        return steer;
      }

      return p5.createVector(0, 0);
    };

    separation(boids: Boid[], p5: p5): Vector {
      let steering = p5.createVector();
      let total = 0;
      for (let other of boids) {
        let d = p5.dist(
          this.position.x,
          this.position.y,
          other.position.x,
          other.position.y,
        );
        if (other != this && d < this.perceptionRadius) {
          // Use this.perceptionRadius instead of a passed value
          let diff = Vector.sub(this.position, other.position);
          diff.div(d * d); // Weight by distance squared for more realistic avoidance
          steering.add(diff);
          total++;
        }
      }
      if (total > 0) {
        steering.div(total);
        steering.setMag(this.maxSpeed);
        steering.sub(this.velocity);
        steering.mult(this.separationFactor);
        steering.limit(this.maxForce);
      }
      return steering;
    }

    alignment(boids: Boid[], p5: p5): Vector {
      let averageVelocity = p5.createVector();
      let total = 0;
      for (let other of boids) {
        let distance = p5.dist(
          this.position.x,
          this.position.y,
          other.position.x,
          other.position.y,
        );
        if (other != this && distance < this.perceptionRadius) {
          // Use this.perceptionRadius instead of a passed value
          averageVelocity.add(other.velocity);
          total++;
        }
      }
      if (total > 0) {
        averageVelocity.div(total);
        averageVelocity.setMag(this.maxSpeed);
        averageVelocity.sub(this.velocity);
        averageVelocity.mult(this.alignmentFactor);
        averageVelocity.limit(this.maxForce);
      }
      return averageVelocity;
    }

    cohesion(boids: Boid[], p5: p5): Vector {
      let averagePosition = p5.createVector();
      let total = 0;
      for (let other of boids) {
        let distance = p5.dist(
          this.position.x,
          this.position.y,
          other.position.x,
          other.position.y,
        );
        if (other != this && distance < this.perceptionRadius) {
          // Use this.perceptionRadius instead of a passed value
          averagePosition.add(other.position);
          total++;
        }
      }
      if (total > 0) {
        averagePosition.div(total);
        let steer = Vector.sub(averagePosition, this.position);
        steer.setMag(this.maxSpeed);
        steer.sub(this.velocity);
        steer.mult(this.cohesionFactor);
        steer.limit(this.maxForce);
        return steer;
      } else {
        return p5.createVector(0, 0);
      }
    }
  }
</script>

<main>
  <div class="controls"></div>
</main>

<style>
  .controls {
    position: fixed;
    bottom: 0;
    right: 0;
    margin: 8px;
  }
</style>
