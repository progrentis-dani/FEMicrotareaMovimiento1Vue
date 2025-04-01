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

      <!-- Pantallazo rojo -->
      <div v-if="pantallazoRojo" class="pantallazo-rojo"></div>

      <!-- Area de info -->
      <div class="info">
        <!--timer y vidas-->
        <div class="status">
          <p>⏳{{ timeLeft }} </p>
          <p>❤️ {{ lives }}</p>
        </div>
        <!-- cruceta -->
        <div class="controls">
          <button @click="cambiarDireccion('up')" :class="{'clicked':buttonClicked === 'up'}">↑</button>
          <div class="row">
            <button @click="cambiarDireccion('left')" :class="{'clicked':buttonClicked === 'left'}">←</button>
            <button @click="cambiarDireccion('right')" :class="{'clicked':buttonClicked === 'right'}">→</button>
          </div>
          <button @click="cambiarDireccion('down')" :class="{'clicked':buttonClicked === 'down'}">↓</button>
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

      <!-- Mensaje -->
    <div v-if="gameOver" class="game-over">
      <h1 v-if="victory">¡Has ganado! 🍒</h1>
      <h1 v-else>¡Perdiste! 💀</h1>
      <!--<button @click="reiniciarJuego">↻</button>-->
    </div>
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
      buttonClicked:null,
      pantallazoRojo: false,
      direccionActual: 'up',
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
    this.generarLaberinto(); // Inicializar paredes y obstaculos
    this.autoMoveInterval = setInterval(this.mover, 50); // Movimiento automático
    this.moverObstaculos(); // Movimiento de obstaculos
    this.startTimer(); // Iniciar timer
  },
  beforeUnmount() {
    clearInterval(this.autoMoveInterval); // Limpiar intervalos al desmontar componente
    clearInterval(this.timer);
  },
  methods: {
    cambiarDireccion(nuevaDireccion) {
      this.direccionActual = nuevaDireccion;
      this.buttonClicked = nuevaDireccion;
      setTimeout(() => {
        this.buttonClicked = null;
      }, 150);  // Tiempo para que el boton se mantenga rojo
    },
    mover() {
      if (this.gameOver) return;
      const previousX = this.posX;
      const previousY = this.posY;

      // Movimiento automatico
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
      // Verificar colision con el objetivo
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

      // Verificar colision con obstaculos
      if (!this.invulnerable) {
        for (let obstacle of this.obstacles) {
          if (
            this.posX < obstacle.x + this.obstacleSize &&
            this.posX + this.circleSize > obstacle.x &&
            this.posY < obstacle.y + this.obstacleSize &&
            this.posY + this.circleSize > obstacle.y
          ) {
            this.loseLife(); // Perder una vida
          }
        }
      }
      // Verificar colision
      if (
        this.posX < this.suelo.x + this.suelo.width &&
        this.posX + this.circleSize > this.suelo.x &&
        this.posY < this.suelo.y + this.suelo.height &&
        this.posY + this.circleSize > this.suelo.y
      ) {
        // Si hay colisin revertir el movimiento
        this.posY = this.suelo.y - this.circleSize;
      }

      // Verificar colision con paredes
      for (let wall of this.walls) {
        if (
          this.posX < wall.x + wall.width &&
          this.posX + this.circleSize > wall.x &&
          this.posY < wall.y + wall.height &&
          this.posY + this.circleSize > wall.y
        ) {
          // Si hay colision con una pared revertir el movimiento
          this.posX = previousX;
          this.posY = previousY;
        }
      }
    },
    moverObstaculos() {
  setInterval(() => {
    for (let obstacle of this.obstacles) {
      // Movimiento de los obstaculos
      obstacle.x += obstacle.dx * this.obstacleSpeed;
      obstacle.y += obstacle.dy * this.obstacleSpeed;

      const gameAreaWidth = this.$el.querySelector('.game-area').clientWidth;
      const gameAreaHeight = this.$el.querySelector('.game-area').clientHeight;

      // Verificar los limites del area de juego
      if (obstacle.x <= 0 || obstacle.x >= gameAreaWidth - this.obstacleSize) {
        obstacle.dx = -obstacle.dx; // Rebotar horizontalmente
      }
      if (obstacle.y <= 0 || obstacle.y >= gameAreaHeight - this.obstacleSize) {
        obstacle.dy = -obstacle.dy; // Rebotar verticalmente
      }

      // Verificar colision con las paredes
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

      // colision con el suelo
      if (
        obstacle.x < this.suelo.x + this.suelo.width &&
        obstacle.x + this.obstacleSize > this.suelo.x &&
        obstacle.y + this.obstacleSize > this.suelo.y &&
        obstacle.y < this.suelo.y + this.suelo.height
      ) {
        obstacle.dy = -obstacle.dy; // rebotar 
        obstacle.y = this.suelo.y - this.obstacleSize; 
      }
    }
  }, 100);
},

  generarLaberinto() {
  // limpiar paredes y obst
  this.walls = [];
  this.obstacles = [];

  const generateRandomPosition = (width, height, avoidX, avoidY, avoidRadius = 0) => {
    let x, y;
    let overlap;
    do {
      x = Math.floor(Math.random() * (800 - width)); 
      y = Math.floor(Math.random() * (600 - height)); 
      overlap = this.walls.some(wall =>
        x < wall.x + wall.width && x + width > wall.x && y < wall.y + wall.height && y + height > wall.y
      ) || (Math.abs(x - avoidX) < avoidRadius && Math.abs(y - avoidY) < avoidRadius);
    } while (overlap); 
    return { x, y };
  };

  for (let i = 0; i < 8; i++) {
  const { x, y } = generateRandomPosition(100, 30, this.cherryX, this.cherryY, 50);
  
  const wallWidth = 90;
  const wallHeight = 30;
  const overlapWithCircle = (
    this.posX < x + wallWidth &&
    this.posX + this.circleSize > x &&
    this.posY < y + wallHeight &&
    this.posY + this.circleSize > y
  );

  if (!overlapWithCircle) {
    this.walls.push({ x, y, width: 90, height: 30 });
  } else {
    i--; // reintentar si hay solapamiento
  }
}

  // Generar obstaculos sin solaparse con paredes o cerca del circulo
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
      ) || (Math.abs(x - this.posX) < 100 && Math.abs(y - this.posY) < 100); // evita que este cerca del círculo
    } while (overlap); // reintentar si se solapa
    this.obstacles.push({ x, y, dx, dy });
  }
},

    loseLife() {
        this.lives--; // reducir una vida
        if (this.lives > 0) {
          this.invulnerable = true; 
          setTimeout(() => {
            this.invulnerable = false; 
          }, 1000); 
          this.posX = 0; 
          this.posY = 0;

          this.direccionActual = 'up';  

          this.pantallazoRojo = true; // mostrar el pantallazo rojo
          setTimeout(() => {
            this.pantallazoRojo = false; // ocultar el pantallazo 
          }, 200);

          this.mover();

          // solo reiniciar los obstaculos
          this.reiniciarObstaculos(); 
        } else {
          this.gameOver = true;
          clearInterval(this.timer); // detener el timer
        }
      },
    reiniciarObstaculos() {
      this.obstacles = [];
      for (let i = 0; i < this.obstacleCount; i++) {
        let x, y, dx, dy;
        let overlap;
        do {
          x = Math.floor(Math.random() * (800 - this.obstacleSize)); // ajusta el tamaño del area de juego
          y = Math.floor(Math.random() * (600 - this.obstacleSize)); // ajusta el tamaño del area de juego
          dx = Math.random() < 0.5 ? 1 : -1; // direccion inicial horizontal
          dy = Math.random() < 0.5 ? 1 : -1; // direccion inicial vertical
          overlap = this.walls.some(wall =>
            x < wall.x + wall.width && x + this.obstacleSize > wall.x && y < wall.y + wall.height && y + this.obstacleSize > wall.y
          ) || (Math.abs(x - this.posX) < 100 && Math.abs(y - this.posY) < 100); // evita que este cerca del circulo
        } while (overlap); // reintentar si se solapa con una pared o está cerca del circulo
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
    /*reiniciarJuego() {
      this.posX = 0;
      this.posY = 0;
      this.victory = false;
      this.gameOver = false;
      this.lives = 3; // Reiniciar vidas
      this.timeLeft = 60; // Reiniciar tiempo
      this.generarLaberinto(); // Generar un nuevo laberinto
      this.startTimer(); // Iniciar temporizador de nuevo
    },*/
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
  position: absolute;
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

button.clicked {
  background-color: red !important;  /* Cambia el fondo a rojo */
  border-color: red;  /* Opcional: cambia el borde también a rojo */
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
.pantallazo-rojo {
  position: absolute;
  width: 800px;
  height: 600px;
  background-color: rgba(245, 90, 90, 0.6);
  z-index: 1000; /* Asegura que esté por encima de todos los elementos */
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