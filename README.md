const canvas = document.getElementById("gameCanvas");
const ctx = canvas.getContext("2d");
const scoreEl = document.getElementById("score");
const bestEl = document.getElementById("best");
const overlay = document.getElementById("gameOverlay");
const overlayTitle = document.getElementById("overlayTitle");
const overlaySubtitle = document.getElementById("overlaySubtitle");
const startButton = document.getElementById("startButton");

const lanes = [canvas.width * 0.28, canvas.width * 0.5, canvas.width * 0.72];
const groundY = canvas.height - 64;

const state = {
  running: false,
  gameOver: false,
  score: 0,
  best: Number(localStorage.getItem("temple-run-best") || 0),
  speed: 420,
  distance: 0,
  spawnTimer: 0.8,
  coinTimer: 0.9,
  obstacles: [],
  coins: []
};

const player = {
  lane: 1,
  x: lanes[1],
  width: 48,
  height: 58,
  jumpOffset: 0,
  jumpVelocity: 0,
  sliding: false,
  slideTimer: 0
};

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}

function updateHud() {
  scoreEl.textContent = Math.floor(state.score);
  bestEl.textContent = state.best;
}

function resetPlayer() {
  player.lane = 1;
  player.x = lanes[1];
  player.jumpOffset = 0;
  player.jumpVelocity = 0;
  player.sliding = false;
  player.slideTimer = 0;
}

function resetGameState() {
  state.running = false;
  state.gameOver = false;
  state.score = 0;
  state.speed = 420;
  state.distance = 0;
  state.spawnTimer = 0.8;
  state.coinTimer = 0.9;
  state.obstacles = [];
  state.coins = [];
  resetPlayer();
  updateHud();
}

function showOverlay(title, subtitle) {
  overlayTitle.textContent = title;
  overlaySubtitle.textContent = subtitle;
  overlay.classList.add("visible");
}

function hideOverlay() {
  overlay.classList.remove("visible");
}

function startGame() {
  resetGameState();
  state.running = true;
  hideOverlay();
}

function endGame() {
  if (!state.running) return;

  state.running = false;
  state.gameOver = true;
  state.best = Math.max(state.best, Math.floor(state.score));
  localStorage.setItem("temple-run-best", String(state.best));
  updateHud();

  showOverlay("Game Over", `Final score: ${Math.floor(state.score)}`);
}

function jump() {
  if (!state.running) {
    startGame();
    return;
  }

  if (player.jumpOffset <= 0 && !player.sliding) {
    player.jumpVelocity = 920;
  }
}

function slide() {
  if (!state.running) {
    startGame();
    return;
  }

  if (player.jumpOffset <= 0) {
    player.sliding = true;
    player.slideTimer = 0.7;
  }
}

function moveLane(direction) {
  if (!state.running) {
    startGame();
    return;
  }

  const nextLane = clamp(player.lane + direction, 0, 2);
  player.lane = nextLane;
  player.x = lanes[player.lane];
}

function spawnObstacle() {
  const lane = Math.floor(Math.random() * 3);
  const isRock = Math.random() < 0.7;
  const obstacle = {
    lane,
    x: lanes[lane],
    y: -80,
    width: isRock ? 52 : 72,
    height: isRock ? 54 : 42,
    type: isRock ? "rock" : "log",
    color: isRock ? "#876b4e" : "#9a6239"
  };

  state.obstacles.push(obstacle);
}

function spawnCoinRow() {
  const lane = Math.floor(Math.random() * 3);
  const count = 3 + Math.floor(Math.random() * 3);

  for (let i = 0; i < count; i += 1) {
    state.coins.push({
      lane,
      x: lanes[lane],
      y: -40 - i * 30,
      radius: 12,
      collected: false
    });
  }
}

function getPlayerBox() {
  const bodyHeight = player.sliding ? 30 : player.height;
  const topY = groundY - bodyHeight - player.jumpOffset;
  return {
    x: player.x - player.width / 2 + 5,
    y: topY,
    width: player.width - 10,
    height: bodyHeight
  };
}

function updatePlayer(dt) {
  if (player.jumpVelocity !== 0 || player.jumpOffset > 0) {
    player.jumpOffset += player.jumpVelocity * dt;
    player.jumpVelocity -= 1800 * dt;

    if (player.jumpOffset <= 0) {
      player.jumpOffset = 0;
      player.jumpVelocity = 0;
    }
  }

  if (player.sliding) {
    player.slideTimer -= dt;
    if (player.slideTimer <= 0) {
      player.sliding = false;
      player.slideTimer = 0;
    }
  }

  player.x += (lanes[player.lane] - player.x) * 0.26;
}

function updateWorld(dt) {
  state.speed += dt * 6;
  state.distance += state.speed * dt;
  state.score += dt * 20;

  state.spawnTimer -= dt;
  if (state.spawnTimer <= 0) {
    spawnObstacle();
    state.spawnTimer = Math.max(0.7, 1.6 - state.distance / 1500) + Math.random() * 0.5;
  }

  state.coinTimer -= dt;
  if (state.coinTimer <= 0) {
    spawnCoinRow();
    state.coinTimer = Math.max(0.8, 1.5 - state.distance / 2200) + Math.random() * 0.6;
  }

  for (let i = state.obstacles.length - 1; i >= 0; i -= 1) {
    const obstacle = state.obstacles[i];
    obstacle.y += state.speed * dt;

    if (obstacle.y > canvas.height + 120) {
      state.obstacles.splice(i, 1);
    }
  }

  for (let i = state.coins.length - 1; i >= 0; i -= 1) {
    const coin = state.coins[i];
    coin.y += state.speed * dt;

    if (coin.y > canvas.height + 80) {
      state.coins.splice(i, 1);
    }
  }
}

function intersects(a, b) {
  return (
    a.x < b.x + b.width &&
    a.x + a.width > b.x &&
    a.y < b.y + b.height &&
    a.y + a.height > b.y
  );
}

function handleCollisions() {
  const playerBox = getPlayerBox();

  for (const obstacle of state.obstacles) {
    if (obstacle.lane !== player.lane) continue;

    const obstacleBox = {
      x: obstacle.x - obstacle.width / 2,
      y: obstacle.y,
      width: obstacle.width,
      height: obstacle.height
    };

    if (intersects(playerBox, obstacleBox)) {
      endGame();
      return;
    }
  }

  for (const coin of state.coins) {
    if (coin.lane !== player.lane || coin.collected) continue;

    const dx = Math.abs(coin.x - player.x);
    const dy = Math.abs(coin.y - (groundY - player.height / 2 - player.jumpOffset));

    if (dx < 28 && dy < 28) {
      coin.collected = true;
      state.score += 100;
      state.coins = state.coins.filter((item) => item !== coin);
    }
  }
}

function update(dt) {
  updatePlayer(dt);
  updateWorld(dt);
  handleCollisions();
  updateHud();
}

function drawBackground() {
  const sky = ctx.createLinearGradient(0, 0, 0, canvas.height);
  sky.addColorStop(0, "#0f2e4a");
  sky.addColorStop(0.42, "#1b4161");
  sky.addColorStop(0.42, "#9d673d");
  sky.addColorStop(1, "#4d2a1c");
  ctx.fillStyle = sky;
  ctx.fillRect(0, 0, canvas.width, canvas.height);

  ctx.fillStyle = "rgba(255, 214, 120, 0.9)";
  ctx.beginPath();
  ctx.arc(760, 90, 42, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = "rgba(15, 36, 48, 0.6)";
  for (let i = 0; i < 6; i += 1) {
    const x = i * 180 + (state.distance * 0.15) % 180;
    ctx.beginPath();
    ctx.moveTo(x, 260);
    ctx.lineTo(x + 90, 150);
    ctx.lineTo(x + 180, 260);
    ctx.closePath();
    ctx.fill();
  }

  ctx.fillStyle = "rgba(44, 22, 13, 0.38)";
  ctx.fillRect(0, groundY + 2, canvas.width, canvas.height - groundY);

  for (let i = 0; i < canvas.width; i += 48) {
    ctx.fillStyle = "rgba(255, 210, 120, 0.18)";
    ctx.fillRect(i, groundY + 10, 20, 6);
    ctx.fillRect(i + 10, groundY + 18, 18, 6);
  }
}

function drawCoin(coin) {
  ctx.beginPath();
  ctx.arc(coin.x, coin.y, coin.radius, 0, Math.PI * 2);
  ctx.fillStyle = "#ffd34d";
  ctx.fill();

  ctx.beginPath();
  ctx.arc(coin.x - 3, coin.y - 3, coin.radius * 0.6, 0, Math.PI * 2);
  ctx.fillStyle = "#fff0aa";
  ctx.fill();

  ctx.strokeStyle = "#d7a21a";
  ctx.lineWidth = 3;
  ctx.beginPath();
  ctx.arc(coin.x, coin.y, coin.radius - 3, 0, Math.PI * 2);
  ctx.stroke();
}

function drawObstacle(obstacle) {
  const x = obstacle.x - obstacle.width / 2;
  const y = obstacle.y;

  if (obstacle.type === "rock") {
    ctx.fillStyle = obstacle.color;
    ctx.beginPath();
    ctx.moveTo(x + 10, y + obstacle.height);
    ctx.lineTo(x + obstacle.width * 0.2, y + 10);
    ctx.lineTo(x + obstacle.width * 0.6, y);
    ctx.lineTo(x + obstacle.width - 8, y + obstacle.height);
    ctx.closePath();
    ctx.fill();
  } else {
    ctx.fillStyle = obstacle.color;
    ctx.fillRect(x, y, obstacle.width, obstacle.height);
    ctx.fillStyle = "rgba(0,0,0,0.16)";
    ctx.fillRect(x + 8, y + 8, obstacle.width - 16, 8);
  }
}

function drawRunner() {
  const x = player.x;
  const topY = groundY - player.height - player.jumpOffset;
  const actualY = player.sliding ? topY + 24 : topY;
  const drawHeight = player.sliding ? 28 : player.height;
  const stride = Math.sin(state.distance * 0.09) * 8;
  const bodyX = x - 20;

  // shadow
  ctx.fillStyle = "rgba(0,0,0,0.25)";
  ctx.beginPath();
  ctx.ellipse(x, groundY + 12, 26, 8, 0, 0, Math.PI * 2);
  ctx.fill();

  // body
  ctx.fillStyle = "#f7d594";
  ctx.beginPath();
  ctx.arc(x, actualY + 10, 12, 0, Math.PI * 2);
  ctx.fill();

  ctx.fillStyle = player.sliding ? "#77dd77" : "#d85d45";
  const bodyW = player.sliding ? 38 : 28;
  const bodyH = player.sliding ? 14 : 30;
  ctx.fillRect(x - bodyW / 2, actualY + 22, bodyW, bodyH);

  ctx.strokeStyle = "#f0bf7a";
  ctx.lineWidth = 5;
  ctx.beginPath();
  ctx.moveTo(x - 12, actualY + 22);
  ctx.lineTo(x - 18, actualY + 35 + stride * 0.35);
  ctx.moveTo(x + 12, actualY + 22);
  ctx.lineTo(x + 18, actualY + 35 - stride * 0.35);
  ctx.moveTo(x - 8, actualY + 32);
  ctx.lineTo(x - 16, actualY + 52 + stride);
  ctx.moveTo(x + 8, actualY + 32);
  ctx.lineTo(x + 16, actualY + 52 - stride);
  ctx.stroke();

  ctx.fillStyle = "#f2e6cf";
  ctx.fillRect(bodyX, actualY + 18, 6, 14);
  ctx.fillRect(bodyX + 28, actualY + 18, 6, 14);

  if (player.sliding) {
    ctx.fillStyle = "#77dd77";
    ctx.fillRect(x - 22, actualY + 18, 44, 12);
  }
}

function draw() {
  drawBackground();

  for (const coin of state.coins) {
    drawCoin(coin);
  }

  for (const obstacle of state.obstacles) {
    drawObstacle(obstacle);
  }

  drawRunner();
}

function tick(timestamp) {
  if (!tick.lastTime) tick.lastTime = timestamp;
  const dt = Math.min((timestamp - tick.lastTime) / 1000, 0.033);
  tick.lastTime = timestamp;

  if (state.running) {
    update(dt);
  }

  draw();
  requestAnimationFrame(tick);
}

window.addEventListener("keydown", (event) => {
  if (["Space", "ArrowUp", "ArrowDown", "ArrowLeft", "ArrowRight"].includes(event.code)) {
    event.preventDefault();
  }

  if (event.code === "Space" || event.code === "ArrowUp") {
    jump();
  } else if (event.code === "ArrowDown") {
    slide();
  } else if (event.code === "ArrowLeft" || event.code === "KeyA") {
    moveLane(-1);
  } else if (event.code === "ArrowRight" || event.code === "KeyD") {
    moveLane(1);
  } else if (event.code === "Enter" && !state.running) {
    startGame();
  }
});

startButton.addEventListener("click", () => {
  startGame();
});

for (const button of document.querySelectorAll(".control-btn")) {
  button.addEventListener("click", () => {
    const action = button.dataset.action;
    if (action === "left") moveLane(-1);
    if (action === "right") moveLane(1);
    if (action === "jump") jump();
    if (action === "slide") slide();
  });
}

updateHud();
showOverlay("Temple Run", "Press Space or tap Start to begin");
requestAnimationFrame(tick);

