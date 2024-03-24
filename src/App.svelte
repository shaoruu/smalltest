<script lang="ts">
  import p5 from 'p5';

  enum CellType {
    Empty,
    Stone,
    Sand,
    Water,
    Grass,
    Wood,
    Lava,
    Leaves,
    Fire,
  }

  const sketch = (p: p5) => {
    let cols: number;
    let rows: number;
    const cellSize = 10;
    const grid: CellType[][] = [];

    p.setup = () => {
      p.createCanvas(p.windowWidth, p.windowHeight);

      cols = p.floor(p.windowWidth / cellSize);
      rows = p.floor(p.windowHeight / cellSize);

      for (let colIndex = 0; colIndex < cols; colIndex++) {
        grid[colIndex] = [];
        for (let rowIndex = 0; rowIndex < rows; rowIndex++) {
          grid[colIndex][rowIndex] =
            rowIndex >= rows - 3 ? CellType.Stone : CellType.Empty;
        }
      }
    };

    let radius = 3; // Initialize radius
    p.mouseWheel = (event: any) => {
      radius += event.deltaY / 100;
      radius = p.constrain(radius, 1, 10); // Constrain radius between 1 and 10
      radius = p.floor(radius);
    };

    function place(
      radius: number,
      col: number,
      row: number,
      type: CellType,
      chance = 0.2,
    ): void {
      for (let xOffset = -radius; xOffset <= radius; xOffset++) {
        for (let yOffset = -radius; yOffset <= radius; yOffset++) {
          if (p.dist(col, row, col + xOffset, row + yOffset) <= radius) {
            const targetCol = col + xOffset;
            const targetRow = row + yOffset;
            if (
              p.random() < chance &&
              targetCol >= 0 &&
              targetCol < cols &&
              targetRow >= 0 &&
              targetRow < rows
            ) {
              grid[targetCol][targetRow] = type;
            }
          }
        }
      }
    }

    let cellType = CellType.Stone;

    function handleMouseInput() {
      if (p.mouseButton === p.CENTER) {
        cellType = CellType.Empty;
      } else if (p.key === 'w') {
        cellType = CellType.Water;
      } else if (p.key === 's') {
        cellType = CellType.Sand;
      } else if (p.key === 'q') {
        cellType = CellType.Stone;
      } else if (p.key === 'g') {
        cellType = CellType.Grass;
      } else if (p.key === 'o') {
        cellType = CellType.Wood;
      } else if (p.key === 'f') {
        cellType = CellType.Fire;
      } else if (p.key === 'l') {
        cellType = CellType.Leaves;
      } else if (p.key === 'v') {
        cellType = CellType.Lava;
      }

      const col = Math.floor(p.mouseX / cellSize);
      const row = Math.floor(p.mouseY / cellSize);
      if (col >= 0 && col < cols && row >= 0 && row < rows) {
        place(radius, col, row, cellType);
      }
    }

    p.draw = () => {
      p.background(0); // Set a light gray background

      for (let i = 0; i < cols; i++) {
        for (let j = rows - 1; j >= 0; j--) {
          switch (grid[i][j]) {
            case CellType.Sand:
              let sandMoved = false;
              // Sand tries to fall straight down if possible
              if (
                j < rows - 1 &&
                (grid[i][j + 1] === CellType.Empty ||
                  grid[i][j + 1] === CellType.Water)
              ) {
                grid[i][j] = CellType.Empty;
                grid[i][j + 1] = CellType.Sand;
                sandMoved = true;
              }
              // If sand can't fall straight down, it tries to roll off to the sides
              if (!sandMoved) {
                let leftOpen =
                  i > 0 &&
                  (grid[i - 1][j + 1] === CellType.Empty ||
                    grid[i - 1][j + 1] === CellType.Water);
                let rightOpen =
                  i < cols - 1 &&
                  (grid[i + 1][j + 1] === CellType.Empty ||
                    grid[i + 1][j + 1] === CellType.Water);
                if (leftOpen && rightOpen) {
                  // If both sides are open, choose randomly
                  if (Math.random() < 0.5) {
                    grid[i][j] = CellType.Empty;
                    grid[i - 1][j + 1] = CellType.Sand;
                  } else {
                    grid[i][j] = CellType.Empty;
                    grid[i + 1][j + 1] = CellType.Sand;
                  }
                } else if (leftOpen) {
                  grid[i][j] = CellType.Empty;
                  grid[i - 1][j + 1] = CellType.Sand;
                } else if (rightOpen) {
                  grid[i][j] = CellType.Empty;
                  grid[i + 1][j + 1] = CellType.Sand;
                }
              }
              break;
            case CellType.Water:
              let moved = false;
              // Water tries to flow downwards
              if (j < rows - 1 && grid[i][j + 1] === CellType.Empty) {
                grid[i][j] = CellType.Empty;
                grid[i][j + 1] = CellType.Water;
                moved = true;
              }
              // Water spreads out to the sides if it can't move down
              if (!moved) {
                let leftEmpty = i > 0 && grid[i - 1][j] === CellType.Empty;
                let rightEmpty =
                  i < cols - 1 && grid[i + 1][j] === CellType.Empty;
                if (leftEmpty && rightEmpty) {
                  // If both sides are empty, choose randomly
                  if (Math.random() < 0.5) {
                    grid[i - 1][j] = CellType.Water;
                  } else {
                    grid[i + 1][j] = CellType.Water;
                  }
                  grid[i][j] = CellType.Empty;
                  moved = true;
                } else if (leftEmpty) {
                  grid[i - 1][j] = CellType.Water;
                  grid[i][j] = CellType.Empty;
                  moved = true;
                } else if (rightEmpty) {
                  grid[i + 1][j] = CellType.Water;
                  grid[i][j] = CellType.Empty;
                  moved = true;
                }
              }
              break;
            case CellType.Fire:
              // Fire burns wood around it slowly, dissipates over time, and is put out by water
              let fireDissipated = Math.random() < 0.1; // 10% chance for fire to dissipate
              let fireExtinguished = false; // Flag to check if fire is extinguished by water
              if (fireDissipated) {
                grid[i][j] = CellType.Empty; // Fire dissipates
              } else {
                for (let dx = -1; dx <= 1; dx++) {
                  for (let dy = -1; dy <= 1; dy++) {
                    if (dx === 0 && dy === 0) continue; // Skip the fire cell itself
                    let nx = i + dx;
                    let ny = j + dy;
                    if (nx >= 0 && nx < cols && ny >= 0 && ny < rows) {
                      if (
                        (grid[nx][ny] === CellType.Wood ||
                          grid[nx][ny] === CellType.Grass ||
                          grid[nx][ny] === CellType.Leaves) &&
                        Math.random() < 0.1
                      ) {
                        // 10% chance to burn adjacent wood
                        grid[nx][ny] = CellType.Fire;
                      } else if (grid[nx][ny] === CellType.Water) {
                        // Fire is extinguished if adjacent to water
                        grid[i][j] = CellType.Empty;
                        fireExtinguished = true;
                        break; // Exit the loop as the fire has been extinguished
                      }
                    }
                  }
                  if (fireExtinguished) break; // Exit the outer loop as well if the fire has been extinguished
                }
              }
              break;
            case CellType.Lava:
              // Check for water around lava to turn it into stone, including horizontally
              let turnedToStone = false;
              for (let dx = -1; dx <= 1 && !turnedToStone; dx++) {
                for (let dy = -1; dy <= 1 && !turnedToStone; dy++) {
                  if (dx === 0 && dy === 0) continue; // Skip the lava cell itself
                  let nx = i + dx;
                  let ny = j + dy;
                  if (
                    nx >= 0 &&
                    nx < cols &&
                    ny >= 0 &&
                    ny < rows &&
                    grid[nx][ny] === CellType.Water
                  ) {
                    grid[i][j] = CellType.Stone;
                    turnedToStone = true;
                  }
                }
              }
              // If not turned to stone, simulate slow lava flow downwards
              if (!turnedToStone && Math.random() < 0.02) {
                // 2% chance for lava to move
                if (j < rows - 1 && grid[i][j + 1] === CellType.Empty) {
                  grid[i][j] = CellType.Empty;
                  grid[i][j + 1] = CellType.Lava;
                }
              }
              break;
          }
        }
      }

      p.mouseIsPressed && handleMouseInput();

      function addNoiseToColor(
        color: p5.Color,
        noiseLevel: number,
        x: number,
        y: number,
        scale: number = 0.05,
      ): p5.Color {
        let noiseValue = p.noise(x * scale, y * scale) * noiseLevel;
        let noisyColor = p.color(
          p.red(color) + noiseValue - noiseLevel / 2,
          p.green(color) + noiseValue - noiseLevel / 2,
          p.blue(color) + noiseValue - noiseLevel / 2,
          p.alpha(color),
        );
        return noisyColor;
      }

      for (let i = 0; i < cols; i++) {
        for (let j = 0; j < rows; j++) {
          switch (grid[i][j]) {
            case CellType.Stone:
              const stoneColor = addNoiseToColor(p.color('#707070'), 100, i, j);
              p.fill(stoneColor);
              break;
            case CellType.Sand:
              const sandColor = addNoiseToColor(
                p.color(255, 255, 128),
                200,
                i,
                j,
              );
              p.fill(sandColor);
              break;
            case CellType.Water:
              const waterColors = [
                p.color(64, 164, 223, 120),
                p.color(58, 158, 215, 120),
                p.color(70, 170, 231, 120),
              ];
              const waterColorIndex =
                (i + j + Math.floor(p.frameCount / 10)) % waterColors.length;
              p.fill(waterColors[waterColorIndex]);
              break;
            case CellType.Grass:
              p.fill(96, 169, 23); // Enhanced color for Grass cells using RGB values
              break;
            case CellType.Leaves:
              p.fill('rgba(76, 175, 80, 0.5)'); // Semi-transparent different green for Leaves cells
              break;
            case CellType.Wood:
              p.fill('#8B4513'); // Enhanced color for Wood cells
              break;
            case CellType.Fire:
              const orangeShades = ['#FF4500', '#FFA500', '#FF8C00']; // Array of orange shades
              const randomOrange =
                orangeShades[Math.floor(Math.random() * orangeShades.length)]; // Randomly select an orange shade
              p.fill(randomOrange); // Apply the randomly selected orange shade
              break;
            case CellType.Lava:
              // Dynamically fluctuate lava colors based on position for a more varied effect
              // make it even more orange
              const dynamicLavaColors = [
                '#FF4500',
                '#FF0000',
                '#FF0000',
                '#FF4500',
                '#FF0000',
                '#FF6347',
                '#FF4500',
                '#FF0000',
                '#FF0000',
                '#FF4500',
                '#FF0000',
                '#FF0000',
                '#FF4500',
                '#FF0000',
                '#FF6347',
                '#FF4500',
                '#FF0000',
                '#FF0000',
              ]; // Expanded array of lava colors
              const dynamicIndex =
                (i + j + Math.floor(p.frameCount / 25)) %
                dynamicLavaColors.length; // Change color more frequently and based on cell position
              p.fill(dynamicLavaColors[dynamicIndex]); // Apply the dynamically selected lava color
              break;
            default:
              p.noFill(); // No fill for empty cells
          }
          p.noStroke();
          p.rect(i * cellSize, j * cellSize, cellSize, cellSize);
        }
      }
    };
  };

  new p5(sketch);
</script>

<main></main>
