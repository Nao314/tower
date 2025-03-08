<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>タワーディフェンスゲーム 改良版</title>
  <style>
    body {
      background: #222;
      margin: 0;
      color: #eee;
      font-family: sans-serif;
    }
    h1 { text-align: center; margin-top: 20px; }
    #gameCanvas {
      background: #444;
      display: block;
      margin: 20px auto;
      border: 2px solid #888;
    }
    #ui {
      text-align: center;
      margin-bottom: 20px;
    }
    .button {
      background: #555;
      color: #eee;
      padding: 10px 20px;
      border: none;
      margin: 5px;
      cursor: pointer;
      font-size: 16px;
    }
    .button:hover {
      background: #777;
    }
    .stats {
      margin-top: 10px;
      font-size: 18px;
    }
  </style>
</head>
<body>
  <h1>タワーディフェンスゲーム 改良版</h1>
  <canvas id="gameCanvas" width="800" height="600"></canvas>
  <div id="ui">
    <button class="button" onclick="selectTower('cannon')">キャノンタワー ($100)</button>
    <button class="button" onclick="selectTower('arrow')">アロータワー ($80)</button>
    <button class="button" onclick="selectTower('magic')">マジックタワー ($120)</button>
    <div class="stats">
      所持金: $<span id="moneyDisplay">300</span>&nbsp;&nbsp;
      Time: <span id="timeDisplay">0</span> s&nbsp;&nbsp;
      Score: <span id="scoreDisplay">0</span>&nbsp;&nbsp;
      Lives: <span id="livesDisplay">10</span>
    </div>
  </div>

  <script>
    const canvas = document.getElementById("gameCanvas");
    const ctx = canvas.getContext("2d");

    let money = 300;
    let towers = [];
    let enemies = [];
    let projectiles = [];
    let selectedTowerType = null;
    let waveCount = 0;
    let enemySpawnTimer = 0;
    const enemySpawnInterval = 60; // フレーム数
    let gameFrame = 0;
    let score = 0;
    let lives = 10;
    let gameOver = false;

    // 敵が進むパス（ウェイポイント）
    const path = [
      {x: 0, y: 300},
      {x: 200, y: 300},
      {x: 200, y: 100},
      {x: 600, y: 100},
      {x: 600, y: 500},
      {x: 800, y: 500},
    ];

    // 各タワーの設定に projectileColor を追加
    const towerTypes = {
      cannon: { cost: 100, range: 120, fireRate: 60, damage: 30, color: 'orange', projectileSpeed: 3, splashRadius: 20, projectileColor: 'darkorange' },
      arrow: { cost: 80, range: 100, fireRate: 30, damage: 15, color: 'yellow', projectileSpeed: 6, projectileColor: 'gold' },
      magic: { cost: 120, range: 150, fireRate: 90, damage: 40, color: 'purple', projectileSpeed: 4, slowEffect: { multiplier: 0.5, duration: 60 }, projectileColor: 'violet' },
    };

    // 敵クラス
    class Enemy {
      constructor(type) {
        this.type = type;
        if (type === 'fast') {
          this.speed = 2.0;
          this.maxHealth = 40;
          this.color = 'cyan';
        } else if (type === 'tank') {
          this.speed = 1.0;
          this.maxHealth = 100;
          this.color = 'red';
        } else { // normal
          this.speed = 1.5;
          this.maxHealth = 60;
          this.color = 'green';
        }
        this.health = this.maxHealth;
        this.x = path[0].x;
        this.y = path[0].y;
        this.waypointIndex = 0;
        this.radius = 10;
        this.slowTimer = 0;
        this.slowMultiplier = 1;
        this.reachedEnd = false;
      }
      
      update() {
        let effectiveSpeed = this.speed;
        if(this.slowTimer > 0) {
          effectiveSpeed *= this.slowMultiplier;
          this.slowTimer--;
        }
        if (this.waypointIndex < path.length - 1) {
          const target = path[this.waypointIndex + 1];
          const dx = target.x - this.x;
          const dy = target.y - this.y;
          const dist = Math.sqrt(dx*dx + dy*dy);
          if (dist < effectiveSpeed) {
            this.x = target.x;
            this.y = target.y;
            this.waypointIndex++;
          } else {
            this.x += (dx / dist) * effectiveSpeed;
            this.y += (dy / dist) * effectiveSpeed;
          }
        } else {
          if (!this.reachedEnd) {
            lives--;
            this.reachedEnd = true;
            if(lives <= 0) { gameOver = true; }
          }
          this.health = 0;
        }
      }
      
      draw() {
        ctx.beginPath();
        ctx.fillStyle = this.color;
        ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
        ctx.fill();
        // 体力バー
        ctx.fillStyle = 'black';
        ctx.fillRect(this.x - this.radius, this.y - this.radius - 10, this.radius * 2, 5);
        ctx.fillStyle = 'lime';
        ctx.fillRect(this.x - this.radius, this.y - this.radius - 10, (this.health/this.maxHealth) * this.radius * 2, 5);
      }
    }

    // タワークラス（自動レベルアップを削除、手動アップグレードに対応）
    class Tower {
      constructor(x, y, type) {
        this.x = x;
        this.y = y;
        this.type = type;
        this.range = towerTypes[type].range;
        this.fireRate = towerTypes[type].fireRate;
        this.damage = towerTypes[type].damage;
        this.color = towerTypes[type].color;
        this.projectileSpeed = towerTypes[type].projectileSpeed;
        this.fireCooldown = 0;
        this.level = 1;
      }
      
      update() {
        if (this.fireCooldown > 0) {
          this.fireCooldown--;
        } else {
          for (let enemy of enemies) {
            const dx = enemy.x - this.x;
            const dy = enemy.y - this.y;
            const dist = Math.sqrt(dx*dx + dy*dy);
            if (dist < this.range) {
              projectiles.push(new Projectile(this.x, this.y, enemy, this.damage, this.projectileSpeed, this.type));
              this.fireCooldown = this.fireRate;
              break;
            }
          }
        }
      }
      
      draw() {
        ctx.beginPath();
        ctx.fillStyle = this.color;
        ctx.fillRect(this.x - 15, this.y - 15, 30, 30);
        ctx.fillStyle = 'white';
        ctx.font = "12px sans-serif";
        ctx.fillText("Lv" + this.level, this.x - 15, this.y - 20);
        // 射程範囲の描画（細い点線）
        ctx.beginPath();
        ctx.strokeStyle = this.color;
        ctx.lineWidth = 1;
        ctx.setLineDash([4, 2]);
        ctx.arc(this.x, this.y, this.range, 0, Math.PI*2);
        ctx.stroke();
        ctx.setLineDash([]);
      }
    }

    // 弾クラス（タワーごとの projectileColor を使用）
    class Projectile {
      constructor(x, y, target, damage, speed, towerType) {
        this.x = x;
        this.y = y;
        this.target = target;
        this.damage = damage;
        this.speed = speed;
        this.towerType = towerType;
        this.radius = 5;
        this.hit = false;
        this.color = towerTypes[towerType].projectileColor;
      }
      
      update() {
        const dx = this.target.x - this.x;
        const dy = this.target.y - this.y;
        const dist = Math.sqrt(dx*dx + dy*dy);
        if (dist < this.speed || this.target.health <= 0) {
          this.target.health -= this.damage;
          // タワーごとの特殊効果
          if(this.towerType === 'cannon') {
            const splashRadius = towerTypes.cannon.splashRadius;
            for(let enemy of enemies) {
              if(enemy !== this.target) {
                const dx2 = enemy.x - this.x;
                const dy2 = enemy.y - this.y;
                if(Math.sqrt(dx2*dx2 + dy2*dy2) <= splashRadius) {
                  enemy.health -= this.damage / 2;
                }
              }
            }
          } else if(this.towerType === 'magic') {
            if(this.target.health > 0) {
              this.target.slowTimer = towerTypes.magic.slowEffect.duration;
              this.target.slowMultiplier = towerTypes.magic.slowEffect.multiplier;
            }
          }
          this.hit = true;
        } else {
          this.x += (dx/dist) * this.speed;
          this.y += (dy/dist) * this.speed;
        }
      }
      
      draw() {
        ctx.beginPath();
        ctx.fillStyle = this.color;
        ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
        ctx.fill();
      }
    }

    // 直線セグメントと点の距離を求める関数
    function pointToSegmentDistance(px, py, x1, y1, x2, y2) {
      const A = px - x1;
      const B = py - y1;
      const C = x2 - x1;
      const D = y2 - y1;
      const dot = A * C + B * D;
      const len_sq = C * C + D * D;
      let param = -1;
      if (len_sq !== 0) { param = dot / len_sq; }
      let xx, yy;
      if (param < 0) {
        xx = x1; yy = y1;
      } else if (param > 1) {
        xx = x2; yy = y2;
      } else {
        xx = x1 + param * C;
        yy = y1 + param * D;
      }
      const dx = px - xx;
      const dy = py - yy;
      return Math.sqrt(dx*dx + dy*dy);
    }

    // クリック位置がパスに近いかを判定（30px以内なら近いとする）
    function isNearPath(x, y) {
      const threshold = 30;
      for(let i = 0; i < path.length - 1; i++) {
        const dist = pointToSegmentDistance(x, y, path[i].x, path[i].y, path[i+1].x, path[i+1].y);
        if(dist < threshold) { return true; }
      }
      return false;
    }

    // キャンバスクリック時の処理
    // ・既存のタワーがクリックされた場合はアップグレード処理を行う（手動アップグレード）
    // ・それ以外の場合、クリック位置がパス上でなければタワーを新規設置する
    canvas.addEventListener('click', function(event) {
      const rect = canvas.getBoundingClientRect();
      const x = event.clientX - rect.left;
      const y = event.clientY - rect.top;
      
      // 既存タワーの判定（中心から20px以内ならアップグレード対象）
      for(let tower of towers) {
        const dx = tower.x - x;
        const dy = tower.y - y;
        if(Math.sqrt(dx*dx + dy*dy) < 20) {
          const upgradeCost = tower.level * 50;
          if(confirm("このタワーをアップグレードしますか？ コスト: $" + upgradeCost)) {
            if(money >= upgradeCost) {
              money -= upgradeCost;
              tower.level++;
              tower.damage = Math.floor(tower.damage * 1.2);
              tower.range = Math.floor(tower.range * 1.1);
              tower.fireRate = Math.max(Math.floor(tower.fireRate * 0.9), 5);
              document.getElementById("moneyDisplay").innerText = money;
            } else {
              alert("資金が足りません！");
            }
          }
          return;
        }
      }
      
      // 新規タワー設置の場合、パス上は設置不可
      if(isNearPath(x, y)) {
        alert("タワーは道上に設置できません！");
        return;
      }
      
      // 選択中のタワー種類があれば設置
      if (selectedTowerType && money >= towerTypes[selectedTowerType].cost) {
        towers.push(new Tower(x, y, selectedTowerType));
        money -= towerTypes[selectedTowerType].cost;
        document.getElementById("moneyDisplay").innerText = money;
      }
    });

    function selectTower(type) {
      selectedTowerType = type;
    }

    function gameLoop() {
      if(gameOver) {
        ctx.fillStyle = "rgba(0,0,0,0.5)";
        ctx.fillRect(0, 0, canvas.width, canvas.height);
        ctx.fillStyle = "white";
        ctx.font = "48px sans-serif";
        ctx.fillText("GAME OVER", canvas.width/2 - 150, canvas.height/2);
        ctx.font = "24px sans-serif";
        ctx.fillText("Time: " + Math.floor(gameFrame/60) + " s  Score: " + score, canvas.width/2 - 150, canvas.height/2 + 40);
        return;
      }
      
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      
      // パスの描画
      ctx.beginPath();
      ctx.strokeStyle = 'white';
      ctx.lineWidth = 20;
      ctx.lineCap = 'round';
      ctx.moveTo(path[0].x, path[0].y);
      for (let i = 1; i < path.length; i++) {
        ctx.lineTo(path[i].x, path[i].y);
      }
      ctx.stroke();
      
      // タワーの更新と描画
      for (let tower of towers) {
        tower.update();
        tower.draw();
      }
      
      // 弾の更新と描画
      for (let i = projectiles.length - 1; i >= 0; i--) {
        let proj = projectiles[i];
        proj.update();
        if (proj.hit) {
          projectiles.splice(i, 1);
        } else {
          proj.draw();
        }
      }
      
      // 敵の生成（一定間隔）
      enemySpawnTimer--;
      if (enemySpawnTimer <= 0) {
        let enemyType = 'normal';
        if (waveCount % 3 === 0) enemyType = 'tank';
        if (waveCount % 5 === 0) enemyType = 'fast';
        enemies.push(new Enemy(enemyType));
        enemySpawnTimer = enemySpawnInterval;
        waveCount++;
      }
      
      // 敵の更新と描画、死亡時の処理
      for (let i = enemies.length - 1; i >= 0; i--) {
        let enemy = enemies[i];
        enemy.update();
        if (enemy.health <= 0) {
          if(!enemy.reachedEnd) {
            money += 20;
            if(enemy.type === 'tank') score += 20;
            else if(enemy.type === 'fast') score += 15;
            else score += 10;
            document.getElementById("moneyDisplay").innerText = money;
          }
          enemies.splice(i, 1);
        } else {
          enemy.draw();
        }
      }
      
      gameFrame++;
      document.getElementById("timeDisplay").innerText = Math.floor(gameFrame/60);
      document.getElementById("scoreDisplay").innerText = score;
      document.getElementById("livesDisplay").innerText = lives;
      
      requestAnimationFrame(gameLoop);
    }
    
    gameLoop();
  </script>
</body>
</html>
