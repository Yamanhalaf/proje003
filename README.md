<!DOCTYPE html>
<html lang="tr">
<head>
  <meta charset="UTF-8">
  <title>Yılan Oyunu</title>
  <style>
    body {
      margin: 0;
      background: #000;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      font-family: monospace;
    }

    canvas {
      background-color: #111;
      border: 2px solid #0f0;
    }

    #skor {
      position: absolute;
      top: 10px;
      left: 10px;
      color: #0f0;
      font-size: 20px;
    }

    #restartBtn {
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      padding: 15px 30px;
      font-size: 20px;
      background-color: #00cc00;
      color: white;
      border: none;
      border-radius: 10px;
      cursor: pointer;
      box-shadow: 0 0 10px #00ff00;
      display: none;
    }

    #temaBtn, #modBtn {
      position: absolute;
      bottom: 20px;
      padding: 10px 20px;
      font-size: 16px;
      background-color: #222;
      color: white;
      border: 1px solid #0f0;
      border-radius: 8px;
      cursor: pointer;
      box-shadow: 0 0 5px #0f0;
    }

    #temaBtn { right: 20px; }
    #modBtn  { left: 20px; border-color: #f00; box-shadow: 0 0 5px #f00; }
  </style>
</head>
<body>
  <div id="skor">
    Skor: <span id="skorDeger">0</span><br>
    En Yüksek Skor: <span id="enYuksekSkor">0</span>
  </div>

  <canvas id="game" width="400" height="400"></canvas>
  <button id="restartBtn">Yeniden Başla</button>
  <button id="temaBtn">🎨 Tema Değiştir</button>
  <button id="modBtn">🧱 Duvar Modu</button>

  <script>
    const canvas = document.getElementById("game");
    const ctx = canvas.getContext("2d");

    const gridSize = 20;
    let snake = [{ x: 200, y: 200 }];
    let direction = "RIGHT";
    let food = { x: 100, y: 100 };
    let bonusFood = null;
    let cherryFood = null;
    let skor = 0;
    let delay = 100;
    let bonusCounter = 0;
    let cherryCounter = 0;

    let duvarGecilsinMi = localStorage.getItem("duvarModu") === "gecisli";

    const skorSpan = document.getElementById("skorDeger");
    const enYuksekSkorSpan = document.getElementById("enYuksekSkor");
    const restartBtn = document.getElementById("restartBtn");

    let enYuksekSkor = localStorage.getItem("enYuksekSkor") || 0;
    enYuksekSkorSpan.innerText = enYuksekSkor;

    const temalar = [
      { arka: "#111", yilan: "#0f0", yilanG: "#090", yem: "#f00", cerceve: "#0f0" },
      { arka: "#222", yilan: "#fff", yilanG: "#aaa", yem: "#ff0", cerceve: "#fff" },
      { arka: "#002b36", yilan: "#268bd2", yilanG: "#073642", yem: "#cb4b16", cerceve: "#2aa198" }
    ];

    let temaIndex = parseInt(localStorage.getItem("temaIndex")) || 0;
    let drawTema = temalar[temaIndex];

    const canvasElement = document.querySelector("canvas");
    const temaBtn = document.getElementById("temaBtn");
    const modBtn = document.getElementById("modBtn");

    temaBtn.onclick = () => {
      temaIndex = (temaIndex + 1) % temalar.length;
      localStorage.setItem("temaIndex", temaIndex);
      drawTema = temalar[temaIndex];
      canvasElement.style.backgroundColor = drawTema.arka;
      canvasElement.style.borderColor = drawTema.cerceve;
    };

    modBtn.onclick = () => {
      duvarGecilsinMi = !duvarGecilsinMi;
      localStorage.setItem("duvarModu", duvarGecilsinMi ? "gecisli" : "olumlu");
      modBtn.innerText = duvarGecilsinMi
        ? "🧱 Duvar Modu: Geçişli"
        : "🧱 Duvar Modu: Ölümlü";
    };

    modBtn.innerText = duvarGecilsinMi
      ? "🧱 Duvar Modu: Geçişli"
      : "🧱 Duvar Modu: Ölümlü";

    canvasElement.style.backgroundColor = drawTema.arka;
    canvasElement.style.borderColor = drawTema.cerceve;

    function drawYem(x, y, emoji = "🍎") {
      ctx.font = `${gridSize}px serif`;
      ctx.textAlign = "center";
      ctx.textBaseline = "middle";
      ctx.fillText(emoji, x + gridSize / 2, y + gridSize / 2);
    }

    function randomFood() {
      return {
        x: Math.floor(Math.random() * (canvas.width / gridSize)) * gridSize,
        y: Math.floor(Math.random() * (canvas.height / gridSize)) * gridSize,
      };
    }

    function maybeSpawnBonus() {
      if (bonusCounter % 5 === 0 && bonusCounter !== 0 && bonusFood === null) {
        bonusFood = randomFood();
      }
    }

    function maybeSpawnCherry() {
      if (cherryCounter % 7 === 0 && cherryCounter !== 0 && cherryFood === null) {
        cherryFood = randomFood();
      }
    }

    document.addEventListener("keydown", e => {
      if (e.key === "ArrowUp" && direction !== "DOWN") direction = "UP";
      else if (e.key === "ArrowDown" && direction !== "UP") direction = "DOWN";
      else if (e.key === "ArrowLeft" && direction !== "RIGHT") direction = "LEFT";
      else if (e.key === "ArrowRight" && direction !== "LEFT") direction = "RIGHT";
    });

    function gameLoop() {
      const head = { ...snake[0] };

      if (direction === "UP") head.y -= gridSize;
      if (direction === "DOWN") head.y += gridSize;
      if (direction === "LEFT") head.x -= gridSize;
      if (direction === "RIGHT") head.x += gridSize;

      if (!duvarGecilsinMi) {
        if (
          head.x < 0 || head.x >= canvas.width ||
          head.y < 0 || head.y >= canvas.height ||
          snake.some(segment => segment.x === head.x && segment.y === head.y)
        ) {
          restartBtn.style.display = "block";
          return;
        }
      } else {
        if (head.x < 0) head.x = canvas.width - gridSize;
        else if (head.x >= canvas.width) head.x = 0;

        if (head.y < 0) head.y = canvas.height - gridSize;
        else if (head.y >= canvas.height) head.y = 0;

        if (snake.some(segment => segment.x === head.x && segment.y === head.y)) {
          restartBtn.style.display = "block";
          return;
        }
      }

      snake.unshift(head);

      if (head.x === food.x && head.y === food.y) {
        skor++;
        skorSpan.innerText = skor;
        food = randomFood();
        bonusCounter++;
        cherryCounter++;
        maybeSpawnBonus();
        maybeSpawnCherry();

        if (skor % 5 === 0 && delay > 30) {
          delay -= 5;
        }

        if (skor > enYuksekSkor) {
          enYuksekSkor = skor;
          localStorage.setItem("enYuksekSkor", enYuksekSkor);
          enYuksekSkorSpan.innerText = enYuksekSkor;
        }

      } else if (bonusFood && head.x === bonusFood.x && head.y === bonusFood.y) {
        skor += 5;
        skorSpan.innerText = skor;
        bonusFood = null;

      } else if (cherryFood && head.x === cherryFood.x && head.y === cherryFood.y) {
        skor += 2;
        skorSpan.innerText = skor;
        cherryFood = null;
        if (delay > 30) delay -= 10;

      } else {
        snake.pop();
      }

      ctx.clearRect(0, 0, canvas.width, canvas.height);

      // 🌈 Skora göre yılan rengi
      let yilanRenk, govdeRenk;
      if (skor < 10) {
        yilanRenk = "#0f0";
        govdeRenk = "#090";
      } else if (skor < 20) {
        yilanRenk = "#00f";
        govdeRenk = "#009";
      } else if (skor < 30) {
        yilanRenk = "#f90";
        govdeRenk = "#a60";
      } else {
        yilanRenk = "#c0f";
        govdeRenk = "#808";
      }

      snake.forEach((segment, i) => {
        ctx.fillStyle = i === 0 ? yilanRenk : govdeRenk;
        ctx.fillRect(segment.x, segment.y, gridSize, gridSize);
      });

      drawYem(food.x, food.y, "🍎");
      if (bonusFood) drawYem(bonusFood.x, bonusFood.y, "💎");
      if (cherryFood) drawYem(cherryFood.x, cherryFood.y, "🍒");

      setTimeout(gameLoop, delay);
    }

    gameLoop();
    restartBtn.onclick = () => location.reload();
  </script>
</body>
</html>
