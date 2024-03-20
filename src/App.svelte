<script lang="ts">
  import p5 from 'p5';

  enum CellType {
    Empty,
    Dirt,
    Sand,
    Water,
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

    p.mouseDragged = (event: MouseEvent) => {
      event.preventDefault();

      if (p.mouseButton === p.RIGHT) {
        const col = Math.floor(p.mouseX / cellSize);
        const row = Math.floor(p.mouseY / cellSize);
        if (col >= 0 && col < cols && row >= 0 && row < rows) {
          place(radius, col, row, CellType.Water);
        }
        return;
      }

      if (p.mouseButton === p.CENTER) {
        const col = Math.floor(p.mouseX / cellSize);
        const row = Math.floor(p.mouseY / cellSize);
        if (col >= 0 && col < cols && row >= 0 && row < rows) {
          place(radius, col, row, CellType.Dirt);
        }
        return;
      }

      const col = Math.floor(p.mouseX / cellSize);
      const row = Math.floor(p.mouseY / cellSize);
      if (col >= 0 && col < cols && row >= 0 && row < rows) {
        place(radius, col, row, CellType.Sand);
      }
    };

    p.draw = () => {
      p.background(0); // Set a light gray background

      for (let i = 0; i < cols; i++) {
        for (let j = rows - 1; j >= 0; j--) {
          if (grid[i][j] === CellType.Sand) {
            // Sand falls through water
            if (
              j < rows - 1 &&
              (grid[i][j + 1] === CellType.Empty ||
                grid[i][j + 1] === CellType.Water)
            ) {
              grid[i][j] = CellType.Empty;
              grid[i][j + 1] = CellType.Sand;
            }
          }
        }
      }

      for (let i = 0; i < cols; i++) {
        for (let j = rows - 1; j >= 0; j--) {
          if (grid[i][j] === CellType.Water) {
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
            // Water tries to flow downwards again if it hasn't moved
            if (!moved && j < rows - 1 && grid[i][j + 1] === CellType.Empty) {
              grid[i][j] = CellType.Empty;
              grid[i][j + 1] = CellType.Water;
            }
          }
        }
      }

      for (let i = 0; i < cols; i++) {
        for (let j = 0; j < rows; j++) {
          if (grid[i][j] === CellType.Dirt) {
            p.fill(120, 84, 59); // Enhanced color for Dirt cells
          } else if (grid[i][j] === CellType.Sand) {
            p.fill(237, 201, 175); // Enhanced color for Sand cells
          } else if (grid[i][j] === CellType.Water) {
            p.fill(64, 164, 223); // Enhanced color for Water cells
          } else {
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
