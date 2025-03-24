<template>
  <div id="app">
    <div class="game-area">
      <!-- Paredes -->
      <div
        v-for="(wall, index) in walls"
        :key="index"
        class="wall"
        :style="{ top: wall.y + 'px', left: wall.x + 'px', width: wall.width + 'px', height: wall.height + 'px' }"
      ></div>

      <!-- obstaculos -->
      <div
        v-for="(obstacle, index) in obstacles"
        :key="index"
        class="obstacle"
        :style="{ top: obstacle.y + 'px', left: obstacle.x + 'px', width: obstacleSize + 'px', height: obstacleSize + 'px' }"
      ></div>

      <!-- circulo -->
      <div
        class="circle"
        :style="{ top: posY + 'px', left: posX + 'px' }"
      ></div>

      <!-- objetivo -->
      <div
        class="cherry"
        :style="{ top: cherryY + 'px', left: cherryX + 'px' }"
      ></div>
    </div>

    <!-- cruceta -->
    <div class="controls">
      <button @click="cambiarDireccion('up')">Arriba</button>
      <div class="row">
        <button @click="cambiarDireccion('left')">Izquierda</button>
        <button @click="cambiarDireccion('right')">Derecha</button>
      </div>
      <button @click="cambiarDireccion('down')">Abajo</button>
    </div>

    <!-- timer y vidas -->
    <div class="status">
      <p>Tiempo restante: {{ timeLeft }} segundos</p>
      <p>Vidas restantes: {{ lives }}</p>
    </div>

    <!-- Mensaje -->
    <div v-if="gameOver" class="game-over">
      <h1 v-if="victory">¡Has ganado! 🍒</h1>
      <h1 v-else>¡Perdiste! 💀</h1>
      <button @click="reiniciarJuego">Reiniciar</button>
    </div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      gridSize: 16,
      cellSize: 50,
      posX: 0,
      posY: 0,
      cherryX: 400, 
      cherryY: 500, 
      velocidad: 5,
      direccionActual: 'right',
      autoMoveInterval: null,
      walls: [], 
      obstacles: [], 
      obstacleCount: 6, // cantidad de obstaculos
      obstacleSpeed: 4,
      obstacleSize: 30,
      gameOver: false,
      victory: false,
      circleSize: 30,
      lives: 3, // vidas 
      timeLeft: 60, // tiempo por vida
      timer: null, // timer
      invulnerable: false, // invulnerabilidad tras colision
    };
  },
  mounted() {
    this.generarLaberinto(); 
    this.autoMoveInterval = setInterval(this.mover, 50); 
    this.moverObstaculos(); 
    this.startTimer(); 
  },
  beforeUnmount() {
    clearInterval(this.autoMoveInterval); // 
    clearInterval(this.timer);
  },
  methods: {
    cambiarDireccion(nuevaDireccion) {
      this.direccionActual = nuevaDireccion;
    },
    mover() {
      if (this.gameOver) return;
      const previousX = this.posX;
      const previousY = this.posY;

      // movimiento automatico del circulo
      switch (this.direccionActual) {
        case 'up':
          this.posY -= this.velocidad;
          break;
        case 'down':
          this.posY += this.velocidad;
          break;
        case 'left':
          this.posX -= this.velocidad;
          break;
        case 'right':
          this.posX += this.velocidad;
          break;
      }

      this.verificarLimites();
      this.verificarColisiones(previousX, previousY);
    },
    verificarLimites() {
      const gameAreaWidth = this.$el.querySelector('.game-area').clientWidth;
      const gameAreaHeight = this.$el.querySelector('.game-area').clientHeight;

      // limitar el movimiento
      if (this.posX < 0) this.posX = 0;
      if (this.posY < 0) this.posY = 0;
      if (this.posX > gameAreaWidth - this.circleSize) this.posX = gameAreaWidth - this.circleSize;
      if (this.posY > gameAreaHeight - this.circleSize) this.posY = gameAreaHeight - this.circleSize;
    },
    verificarColisiones(previousX, previousY) {
      // verificar colisión
      if (
        this.posX < this.cherryX + this.cellSize &&
        this.posX + this.circleSize > this.cherryX &&
        this.posY < this.cherryY + this.cellSize &&
        this.posY + this.circleSize > this.cherryY
      ) {
        this.gameOver = true;
        this.victory = true;
        clearInterval(this.timer); // Detener timer
      }

      // Verificar colisión con obstaculos
      if (!this.invulnerable) {
        for (let obstacle of this.obstacles) {
          if (
            this.posX < obstacle.x + this.obstacleSize &&
            this.posX + this.circleSize > obstacle.x &&
            this.posY < obstacle.y + this.obstacleSize &&
            this.posY + this.circleSize > obstacle.y
          ) {
            this.loseLife(); // Perder una vida al chocar
          }
        }
      }

      // Verificar colision con paredes
      for (let wall of this.walls) {
        if (
          this.posX < wall.x + wall.width &&
          this.posX + this.circleSize > wall.x &&
          this.posY < wall.y + wall.height &&
          this.posY + this.circleSize > wall.y
        ) {
          this.posX = previousX;
          this.posY = previousY;
        }
      }
    },
    moverObstaculos() {
      setInterval(() => {
        for (let obstacle of this.obstacles) {
          obstacle.x += obstacle.dx * this.obstacleSpeed;
          obstacle.y += obstacle.dy * this.obstacleSpeed;

          const gameAreaWidth = this.$el.querySelector('.game-area').clientWidth;
          const gameAreaHeight = this.$el.querySelector('.game-area').clientHeight;

          if (obstacle.x <= 0 || obstacle.x >= gameAreaWidth - this.obstacleSize) {
            obstacle.dx = -obstacle.dx;
          }
          if (obstacle.y <= 0 || obstacle.y >= gameAreaHeight - this.obstacleSize) {
            obstacle.dy = -obstacle.dy;
          }

          for (let wall of this.walls) {
            if (
              obstacle.x < wall.x + wall.width &&
              obstacle.x + this.obstacleSize > wall.x &&
              obstacle.y < wall.y + wall.height &&
              obstacle.y + this.obstacleSize > wall.y
            ) {
              obstacle.dx = -obstacle.dx;
              obstacle.dy = -obstacle.dy;
            }
          }
        }
      }, 100);
    },
    generarLaberinto() {
      this.walls = [];
      this.obstacles = [];

      const generateRandomPosition = (width, height) => {
        let x, y;
        let overlap;
        do {
          x = Math.floor(Math.random() * (800 - width)); // tamaño del area de juego
          y = Math.floor(Math.random() * (600 - height)); // tamaño del area de juego
          overlap = this.walls.some(wall => 
            x < wall.x + wall.width && x + width > wall.x && y < wall.y + wall.height && y + height > wall.y
          );
        } while (overlap); 
        return { x, y };
      };

      // paredes aleatorias
      for (let i = 0; i < 8; i++) {
        const { x, y } = generateRandomPosition(100, 30);
        this.walls.push({ x, y, width: 100, height: 30 });
      }

      for (let i = 0; i < this.obstacleCount; i++) {
        let x, y, dx, dy;
        let overlap;
        do {
          x = Math.floor(Math.random() * (800 - this.obstacleSize)); 
          y = Math.floor(Math.random() * (600 - this.obstacleSize)); 
          dx = Math.random() < 0.5 ? 1 : -1; 
          dy = Math.random() < 0.5 ? 1 : -1; 
          overlap = this.walls.some(wall => 
            x < wall.x + wall.width && x + this.obstacleSize > wall.x && y < wall.y + wall.height && y + this.obstacleSize > wall.y
          );
        } while (overlap); 
        this.obstacles.push({ x, y, dx, dy });
      }
    },
    loseLife() {
      this.lives--;
      if (this.lives > 0) {
        this.invulnerable = true;
        setTimeout(() => {
          this.invulnerable = false; 
        }, 1000); 
        this.posX = 0; 
        this.posY = 0;
      } else {
        this.gameOver = true;
        clearInterval(this.timer); 
      }
    },
    startTimer() {
      this.timer = setInterval(() => {
        if (this.timeLeft > 0) {
          this.timeLeft--;
        } else {
          this.loseLife();
          this.timeLeft = 60;
        }
      }, 1000);
    },
    reiniciarJuego() {
      this.posX = 0;
      this.posY = 0;
      this.victory = false;
      this.gameOver = false;
      this.lives = 3; 
      this.timeLeft = 60; 
      this.generarLaberinto();
      this.startTimer();
    },
  },
};
</script>

<style scoped>
.game-area {
  position: relative;
  width: 800px;
  height: 600px;
  background-color: #f0f0f0;
  border: 2px solid #000;
  margin: 20px auto;
}
.circle {
  position: absolute;
  width: 30px;
  height: 30px;
  background-color: red;
  border-radius: 50%;
}
.cherry {
  position: absolute;
  width: 50px;
  height: 50px;
  background-color: green;
}
.wall {
  position: absolute;
  background-color: gray;
}
.obstacle {
  position: absolute;
  background-color: blue;
}
.controls {
  text-align: center;
}
.row {
  display: flex;
  justify-content: center;
  gap: 10px;
}
.status {
  text-align: center;
}
.game-over {
  text-align: center;
}
</style>
