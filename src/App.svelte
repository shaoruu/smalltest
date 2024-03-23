<script lang="ts">
  import p5 from 'p5';

  enum CellType {
    Empty,
    Dirt,
    Sand,
    Water,
    Grass,
    Rock,
    Wood,
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

      for (let i = 0; i < cols; i++) {
        grid[i] = [];
        for (let j = 0; j < rows; j++) {
          if (j >= rows - 3) {
            grid[i][j] = CellType.Dirt;
          } else {
            grid[i][j] = CellType.Empty;
          }
        }
      }
    };

    let radius = 3; // Initialize radius
    p.mouseWheel = (event: any) => {
      radius += event.deltaY / 100;
      radius = p.constrain(radius, 1, 10); // Constrain radius between 1 and 10
      radius = p.floor(radius);
    };

    function place(radius: number, x: number, y: number, type: CellType) {
      for (let i = -radius; i <= radius; i++) {
        for (let j = -radius; j <= radius; j++) {
          if (p.dist(x, y, x + i, y + j) <= radius) {
            const newX = x + i;
            const newY = y + j;
            if (newX >= 0 && newX < cols && newY >= 0 && newY < rows) {
              grid[newX][newY] = type;
            }
          }
        }
      }
    }

    function handleMouseInput() {
      let cellType = CellType.Sand; // Default to Sand
      if (p.mouseButton === p.RIGHT) {
        cellType = CellType.Water;
      } else if (p.mouseButton === p.CENTER) {
        cellType = CellType.Dirt;
      } else if (p.key === 'g') {
        cellType = CellType.Grass;
      } else if (p.key === 'r') {
        cellType = CellType.Rock;
      } else if (p.key === 'w') {
        cellType = CellType.Wood;
      } else if (p.key === 'f') {
        cellType = CellType.Fire;
      } else if (p.key === 'l') {
        cellType = CellType.Leaves;
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
          }
        }
      }

      if (p.mouseIsPressed) {
        handleMouseInput();
      }

      for (let i = 0; i < cols; i++) {
        for (let j = 0; j < rows; j++) {
          switch (grid[i][j]) {
            case CellType.Dirt:
              p.fill('#8C6A5D'); // Enhanced color for Dirt cells
              break;
            case CellType.Sand:
              p.fill('#EADFB4'); // Enhanced color for Sand cells
              break;
            case CellType.Water:
              p.fill(64, 164, 223, 120); // Enhanced color for Water cells
              break;
            case CellType.Grass:
              p.fill('#60A917'); // Enhanced color for Grass cells
              break;
            case CellType.Leaves:
              p.fill('rgba(76, 175, 80, 0.5)'); // Semi-transparent different green for Leaves cells
              break;
            case CellType.Rock:
              p.fill('#787878'); // Enhanced color for Rock cells
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
