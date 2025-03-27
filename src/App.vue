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

      <!-- Area de info -->
      <div class="info">
        <!--timer y vidas-->
        <div class="status">
          <p>⏳{{ timeLeft }} </p>
          <p>❤️ {{ lives }}</p>
        </div>
        <!-- cruceta -->
        <div class="controls">
          <button @click="cambiarDireccion('up')"> ↑ </button>
          <div class="row">
            <button @click="cambiarDireccion('left')">←</button>
            <button @click="cambiarDireccion('right')">→</button>
          </div>
          <button @click="cambiarDireccion('down')"> ↓ </button>
        </div>
      </div>

      <!-- Obstáculos -->
      <div
        v-for="(obstacle, index) in obstacles"
        :key="index"
        class="obstacle"
        :style="{ top: obstacle.y + 'px', left: obstacle.x + 'px', width: obstacleSize + 'px', height: obstacleSize + 'px' }"
      ></div>

      <!-- Circulo -->
      <div
        class="circle"
        :class="{ 'flipped': direccionActual === 'left' }"
        :style="{ top: posY + 'px', left: posX + 'px' }"
      ></div>

      <div
        v-for="(wall, index) in walls"
        :key="index"
        class="wall"
        :style="{ top: wall.y + 'px', left: wall.x + 'px', width: wall.width + 'px', height: wall.height + 'px' }"
      ></div>

      <!-- Suelo fijo -->
      <div
        class="suelo"
        :style="{ top: suelo.y + 'px', left: suelo.x + 'px', width: suelo.width + 'px', height: suelo.height + 'px' }"
      ></div>

      <!-- objetivo -->
      <div
        class="cherry"
        :style="{ top: cherryY + 'px', left: cherryX + 'px' }"
      ></div>
    </div>

    <!-- Mensaje -->
    <div v-if="gameOver" class="game-over">
      <h1 v-if="victory">¡Has ganado! 🍒</h1>
      <h1 v-else>¡Perdiste! 💀</h1>
      <button @click="reiniciarJuego">↻</button>
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
      cherryX: 400, // Ajusta esta posición
      cherryY: 500, // Ajusta esta posición
      velocidad: 5,
      direccionActual: 'right',
      autoMoveInterval: null,
      walls: [], // Paredes
      obstacles: [], // Obstáculos 
      obstacleCount: 6, // Cantidad de obstáculos
      obstacleSpeed: 4,
      obstacleSize: 35,
      gameOver: false,
      victory: false,
      circleSize: 40,
      lives: 3, // Vidas 
      timeLeft: 60, // Tiempo por vida
      timer: null, // timer
      invulnerable: false, // Estado de invulnerabilidad tras colisión
      suelo: {
        x: 0, // Desde el borde izquierdo
        y: 550, // Justo debajo del objetivo
        width: 800, // Todo el ancho del área de juego
        height: 50, // Altura del suelo
      },
    };
  },
  mounted() {
    this.generarLaberinto(); // Inicializar paredes y obstáculos
    this.autoMoveInterval = setInterval(this.mover, 50); // Movimiento automático
    this.moverObstaculos(); // Movimiento de obstáculos
    this.startTimer(); // Iniciar timer
  },
  beforeUnmount() {
    clearInterval(this.autoMoveInterval); // Limpiar intervalos al desmontar componente
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

      // Movimiento automático del círculo
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

      // Limitar el movimiento
      if (this.posX < 0) this.posX = 0;
      if (this.posY < 0) this.posY = 0;
      if (this.posX > gameAreaWidth - this.circleSize) this.posX = gameAreaWidth - this.circleSize;
      if (this.posY > gameAreaHeight - this.circleSize) this.posY = gameAreaHeight - this.circleSize;
    },
    verificarColisiones(previousX, previousY) {
      // Verificar colisión con el objetivo
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

      // Verificar colisión con obstáculos
      if (!this.invulnerable) {
        for (let obstacle of this.obstacles) {
          if (
            this.posX < obstacle.x + this.obstacleSize &&
            this.posX + this.circleSize > obstacle.x &&
            this.posY < obstacle.y + this.obstacleSize &&
            this.posY + this.circleSize > obstacle.y
          ) {
            this.loseLife(); // Perder una vida al chocar con un obstáculo
          }
        }
      }
      // Verificar colisión con el suelo
      if (
        this.posX < this.suelo.x + this.suelo.width &&
        this.posX + this.circleSize > this.suelo.x &&
        this.posY < this.suelo.y + this.suelo.height &&
        this.posY + this.circleSize > this.suelo.y
      ) {
        // Si hay colisión con el suelo, revertir el movimiento (si es necesario)
        this.posY = this.suelo.y - this.circleSize;
      }

      // Verificar colisión con paredes
      for (let wall of this.walls) {
        if (
          this.posX < wall.x + wall.width &&
          this.posX + this.circleSize > wall.x &&
          this.posY < wall.y + wall.height &&
          this.posY + this.circleSize > wall.y
        ) {
          // Si hay colisión con una pared revertir el movimiento
          this.posX = previousX;
          this.posY = previousY;
        }
      }
    },
    moverObstaculos() {
  setInterval(() => {
    for (let obstacle of this.obstacles) {
      // Movimiento de los obstáculos
      obstacle.x += obstacle.dx * this.obstacleSpeed;
      obstacle.y += obstacle.dy * this.obstacleSpeed;

      const gameAreaWidth = this.$el.querySelector('.game-area').clientWidth;
      const gameAreaHeight = this.$el.querySelector('.game-area').clientHeight;

      // Verificar los límites del área de juego
      if (obstacle.x <= 0 || obstacle.x >= gameAreaWidth - this.obstacleSize) {
        obstacle.dx = -obstacle.dx; // Rebotar horizontalmente
      }
      if (obstacle.y <= 0 || obstacle.y >= gameAreaHeight - this.obstacleSize) {
        obstacle.dy = -obstacle.dy; // Rebotar verticalmente
      }

      // Verificar colisión con las paredes
      for (let wall of this.walls) {
        if (
          obstacle.x < wall.x + wall.width &&
          obstacle.x + this.obstacleSize > wall.x &&
          obstacle.y < wall.y + wall.height &&
          obstacle.y + this.obstacleSize > wall.y
        ) {
          obstacle.dx = -obstacle.dx; // Rebotar horizontalmente
          obstacle.dy = -obstacle.dy; // Rebotar verticalmente
        }
      }

      // Verificar colisión con el suelo
      if (
        obstacle.x < this.suelo.x + this.suelo.width &&
        obstacle.x + this.obstacleSize > this.suelo.x &&
        obstacle.y + this.obstacleSize > this.suelo.y &&
        obstacle.y < this.suelo.y + this.suelo.height
      ) {
        obstacle.dy = -obstacle.dy; // Rebotar en la dirección vertical
        obstacle.y = this.suelo.y - this.obstacleSize; // Ajustar la posición del obstáculo para que quede justo sobre el suelo
      }
    }
  }, 100);
}
,
    generarLaberinto() {
  // Limpia paredes y obstáculos anteriores
  this.walls = [];
  this.obstacles = [];

  // Función para generar una posición aleatoria sin solaparse con el objetivo ni el círculo
  const generateRandomPosition = (width, height, avoidX, avoidY, avoidRadius = 0) => {
    let x, y;
    let overlap;
    do {
      x = Math.floor(Math.random() * (800 - width)); // Ajusta el tamaño de la área de juego
      y = Math.floor(Math.random() * (600 - height)); // Ajusta el tamaño de la área de juego
      overlap = this.walls.some(wall =>
        x < wall.x + wall.width && x + width > wall.x && y < wall.y + wall.height && y + height > wall.y
      ) || (Math.abs(x - avoidX) < avoidRadius && Math.abs(y - avoidY) < avoidRadius);
    } while (overlap); // Reintentar si se solapa con una pared o está cerca del área prohibida
    return { x, y };
  };

  // Generar paredes aleatorias sin solaparse con el objetivo
  for (let i = 0; i < 8; i++) {
    const { x, y } = generateRandomPosition(100, 30, this.cherryX, this.cherryY, 50);
    this.walls.push({ x, y, width: 90, height: 30 });
  }

  // Generar obstáculos sin solaparse con paredes o cerca del círculo
  for (let i = 0; i < this.obstacleCount; i++) {
    let x, y, dx, dy;
    let overlap;
    do {
      x = Math.floor(Math.random() * (800 - this.obstacleSize)); // Ajusta el tamaño del área de juego
      y = Math.floor(Math.random() * (600 - this.obstacleSize)); // Ajusta el tamaño del área de juego
      dx = Math.random() < 0.5 ? 1 : -1; // Dirección inicial horizontal
      dy = Math.random() < 0.5 ? 1 : -1; // Dirección inicial vertical
      overlap = this.walls.some(wall =>
        x < wall.x + wall.width && x + this.obstacleSize > wall.x && y < wall.y + wall.height && y + this.obstacleSize > wall.y
      ) || (Math.abs(x - this.posX) < 100 && Math.abs(y - this.posY) < 100); // Evita que esté cerca del círculo
    } while (overlap); // Reintentar si se solapa con una pared o está cerca del círculo
    this.obstacles.push({ x, y, dx, dy });
  }
},

    loseLife() {
      this.lives--; // Reducir una vida
      if (this.lives > 0) {
        this.invulnerable = true; // Estado de invulnerabilidad temporal
        setTimeout(() => {
          this.invulnerable = false; // Terminar invulnerabilidad
        }, 1000); // Duración de la invulnerabilidad
        this.posX = 0; // Reiniciar posición del círculo
        this.posY = 0;

        // Solo reiniciar los obstáculos
        this.reiniciarObstaculos(); // Generar nuevos obstáculos sin cambiar las paredes
      } else {
        this.gameOver = true;
        clearInterval(this.timer); // Detener el timer
      }
    },
    reiniciarObstaculos() {
      // Generar obstáculos sin tocar las paredes
      this.obstacles = []; // Limpiar los obstáculos actuales
      for (let i = 0; i < this.obstacleCount; i++) {
        let x, y, dx, dy;
        let overlap;
        do {
          x = Math.floor(Math.random() * (800 - this.obstacleSize)); // Ajusta el tamaño del área de juego
          y = Math.floor(Math.random() * (600 - this.obstacleSize)); // Ajusta el tamaño del área de juego
          dx = Math.random() < 0.5 ? 1 : -1; // Dirección inicial horizontal
          dy = Math.random() < 0.5 ? 1 : -1; // Dirección inicial vertical
          overlap = this.walls.some(wall =>
            x < wall.x + wall.width && x + this.obstacleSize > wall.x && y < wall.y + wall.height && y + this.obstacleSize > wall.y
          ) || (Math.abs(x - this.posX) < 100 && Math.abs(y - this.posY) < 100); // Evita que esté cerca del círculo
        } while (overlap); // Reintentar si se solapa con una pared o está cerca del círculo
        this.obstacles.push({ x, y, dx, dy });
      }
    },
    startTimer() {
      this.timer = setInterval(() => {
        if (this.timeLeft > 0) {
          this.timeLeft--;
        } else {
          this.loseLife();
          this.timeLeft = 60; // Reiniciar tiempo por vida
        }
      }, 1000); // Actualización cada segundo
    },
    reiniciarJuego() {
      this.posX = 0;
      this.posY = 0;
      this.victory = false;
      this.gameOver = false;
      this.lives = 3; // Reiniciar vidas
      this.timeLeft = 60; // Reiniciar tiempo
      this.generarLaberinto(); // Generar un nuevo laberinto
      this.startTimer(); // Iniciar temporizador de nuevo
    },
  },
};
</script>

<style scoped>

.cherry {
  position: absolute;
  width: 50px;
  height: 50px;
  /*background-color: green;*/
}
.controls {
  position: absolute; /* Esto posiciona la cruceta de forma absoluta dentro de la game-area */
  /*top: 10%; /* Centra verticalmente la cruceta dentro de la game-area */
  transform: translateY(100%);
  width: 70%;
  margin:10%;
  display: flex;
  flex-direction: column;
  align-items: center;
  border: 2px solid #222; /* Borde alrededor de la cruceta */
  border-radius: 8px;
  padding: 10px;
  background-color: rgba(0, 0, 0, 0.6); /* Fondo translúcido */
}

.info {
  position: absolute;
  left: 805px;
  width: 200px; /* Ancho del cuadro */
  height: 600px; /* Alto del cuadro */
  margin: auto;
  background-color: gray; /* Color gris */
  border: 2px solid black; /* Borde opcional */
}

.row {
  display: flex;
  justify-content: center;
  gap: 10px;
  margin-top: 10px;
  margin-bottom: 10px;
}

button {
  font-size: 20px;
  width: 50px;
  height: 50px;
  background: #777;
  color: white;
  border: 1px solid #222;
  cursor: pointer;
  transition: background-color 0.3s, transform 0.2s;
}

button:hover {
  background-color: #444;
  transform: scale(1.1);
}

.status {
  position: absolute;
  top: 20px;
  right: 20px;
  background-color: rgba(0, 0, 0, 0.6); /* Fondo translucido */
  color: white;
  padding: 10px;
  border-radius: 8px;
  font-size: 25px;
  z-index: 10; /* Asegura que esté por encima de otros elementos */
  display: flex;
  flex-direction: column;
  align-items: flex-end;
}

.status p {
  margin: 5px 0;
}

.game-over {
  position: fixed; /* Fijamos el contenedor en la pantalla */
  top: 50%; /* Lo posicionamos a la mitad de la altura */
  left: 50%; /* Lo posicionamos a la mitad del ancho */
  transform: translate(-50%, -50%); /* Ajuste para centrar correctamente */
  text-align: center; /* Asegura que el texto esté centrado */
  background-color: rgba(0, 0, 0, 0.7); /* Fondo translúcido para resaltar el mensaje */
  color: white;
  padding: 20px;
  border-radius: 10px;
  z-index: 100; /* Asegura que esté por encima de otros elementos */
}

.game-over h1 {
  font-size: 36px;
  margin-bottom: 20px;
}

.game-over button {
  font-size: 18px;
  padding: 10px 20px;
  background-color: #ff6347;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.game-area {
  position: relative;
  width: 800px;
  height: 600px;
  background-image: url('@/assets/fondo-mario.png');
  background-size: cover;
  right: 105px;
  border: 2px solid #000;
  margin: 20px auto;
}
.circle {
  position: absolute;
  width: 50px;
  height: 50px;
  background-image: url('@/assets/mario3.gif');
  background-size: cover;
  /*border-radius: 50%;*/
}
.flipped {
  transform: scaleX(-1); /* Invierte la imagen horizontalmente */
}
.wall {
  position: absolute;
  background-image: url('@/assets/bloques.png'); 
  background-size: contain;
}
.obstacle {
  position: absolute;
  background-image: url('@/assets/Koopa.gif'); 
  background-size: cover;
  width: 55px;
  height: 55px;
}
.cherry {
  position: absolute;
  background-image: url('@/assets/tubo.png'); 
  background-size: cover;
}
.suelo{
  position: absolute;
  background-image: url('@/assets/pared.png'); 
  background-size: contain;
}
</style>