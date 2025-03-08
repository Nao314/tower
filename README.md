<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>タワーディフェンスゲーム</title>
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
  </style>
</head>
<body>
  <h1>タワーディフェンスゲーム</h1>
  <canvas id="gameCanvas" width="800" height="600"></canvas>
  <div id="ui">
    <button class="button" onclick="selectTower('cannon')">キャノンタワー ($100)</button>
    <button class="button" onclick="selectTower('arrow')">アロータワー ($80)</button>
    <button class="button" onclick="selectTower('magic')">マジックタワー ($120)</button>
    <div style="margin-top:10px;">所持金: $<span id="moneyDisplay">300</span></div>
  </div>

  <script>
    // キャンバスとコンテキストの取得
    const canvas = document.getElementById("gameCanvas");
    const ctx = canvas.getContext("2d");

    // ゲームの基本変数
    let money = 300;
    let towers = [];
    let enemies = [];
    let projectiles = [];
    let selectedTowerType = null;
    let waveCount = 0;
    let enemySpawnTimer = 0;
    const enemySpawnInterval = 60; // フレーム数

    // 敵が進むパス（ウェイポイントの配列）
    const path = [
      {x: 0, y: 300},
      {x: 200, y: 300},
      {x: 200, y: 100},
      {x: 600, y: 100},
      {x: 600, y: 500},
      {x: 800, y: 500},
    ];

    // タワーの種類ごとの設定
    const towerTypes = {
      cannon: { cost: 100, range: 120, fireRate: 60, damage: 30, color: 'orange' },
      arrow: { cost: 80, range: 100, fireRate: 30, damage: 15, color: 'yellow' },
      magic: { cost: 120, range: 150, fireRate: 90, damage: 40, color: 'purple' },
    };

    // 敵クラス
    class Enemy {
      constructor(type) {
        this.type = type;
        // 敵の種類に応じたパラメータ設定
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
      }
      
      update() {
        if (this.waypointIndex < path.length - 1) {
          const target = path[this.waypointIndex + 1];
          const dx = target.x - this.x;
          const dy = target.y - this.y;
          const dist = Math.sqrt(dx*dx + dy*dy);
          if (dist < this.speed) {
            this.x = target.x;
            this.y = target.y;
            this.waypointIndex++;
          } else {
            this.x += (dx / dist) * this.speed;
            this.y += (dy / dist) * this.speed;
          }
        } else {
          // ゴール到達時（ここでは単に体力を0にして削除）
          this.health = 0;
          // ここでライフ減少処理などを実装可能
        }
      }
      
      draw() {
        ctx.beginPath();
        ctx.fillStyle = this.color;
        ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
        ctx.fill();
        // 体力バーの描画
        ctx.fillStyle = 'black';
        ctx.fillRect(this.x - this.radius, this.y - this.radius - 10, this.radius * 2, 5);
        ctx.fillStyle = 'lime';
        ctx.fillRect(this.x - this.radius, this.y - this.radius - 10, (this.health/this.maxHealth) * this.radius * 2, 5);
      }
    }

    // タワークラス
    class Tower {
      constructor(x, y, type) {
        this.x = x;
        this.y = y;
        this.type = type;
        this.range = towerTypes[type].range;
        this.fireRate = towerTypes[type].fireRate;
        this.damage = towerTypes[type].damage;
        this.color = towerTypes[type].color;
        this.fireCooldown = 0;
      }
      
      update() {
        if (this.fireCooldown > 0) {
          this.fireCooldown--;
        } else {
          // 射程内にいる敵を探索
          for (let enemy of enemies) {
            const dx = enemy.x - this.x;
            const dy = enemy.y - this.y;
            const dist = Math.sqrt(dx*dx + dy*dy);
            if (dist < this.range) {
              // 弾を発射
              projectiles.push(new Projectile(this.x, this.y, enemy, this.damage));
              this.fireCooldown = this.fireRate;
              break;
            }
          }
        }
      }
      
      draw() {
        // タワー本体の描画
        ctx.beginPath();
        ctx.fillStyle = this.color;
        ctx.fillRect(this.x - 15, this.y - 15, 30, 30);
        // 射程範囲の描画
        ctx.beginPath();
        ctx.strokeStyle = this.color;
        ctx.arc(this.x, this.y, this.range, 0, Math.PI*2);
        ctx.stroke();
      }
    }

    // 弾クラス
    class Projectile {
      constructor(x, y, target, damage) {
        this.x = x;
        this.y = y;
        this.target = target;
        this.speed = 4;
        this.damage = damage;
        this.radius = 5;
        this.hit = false;
      }
      
      update() {
        const dx = this.target.x - this.x;
        const dy = this.target.y - this.y;
        const dist = Math.sqrt(dx*dx + dy*dy);
        if (dist < this.speed || this.target.health <= 0) {
          // 敵に命中
          this.target.health -= this.damage;
          this.hit = true;
        } else {
          this.x += (dx/dist) * this.speed;
          this.y += (dy/dist) * this.speed;
        }
      }
      
      draw() {
        ctx.beginPath();
        ctx.fillStyle = 'white';
        ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
        ctx.fill();
      }
    }

    // キャンバスクリック時のタワー設置処理
    canvas.addEventListener('click', function(event) {
      const rect = canvas.getBoundingClientRect();
      const x = event.clientX - rect.left;
      const y = event.clientY - rect.top;
      if (selectedTowerType && money >= towerTypes[selectedTowerType].cost) {
        towers.push(new Tower(x, y, selectedTowerType));
        money -= towerTypes[selectedTowerType].cost;
        document.getElementById("moneyDisplay").innerText = money;
      }
    });

    // タワー選択用関数
    function selectTower(type) {
      selectedTowerType = type;
    }

    // ゲームループ
    function gameLoop() {
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

      // 一定間隔で敵を生成
      enemySpawnTimer--;
      if (enemySpawnTimer <= 0) {
        // 敵の種類を波ごとに変化させる例
        let enemyType = 'normal';
        if (waveCount % 3 === 0) enemyType = 'tank';
        if (waveCount % 5 === 0) enemyType = 'fast';
        enemies.push(new Enemy(enemyType));
        enemySpawnTimer = enemySpawnInterval;
        waveCount++;
      }

      // 敵の更新と描画
      for (let i = enemies.length - 1; i >= 0; i--) {
        let enemy = enemies[i];
        enemy.update();
        if (enemy.health <= 0) {
          enemies.splice(i, 1);
          money += 20;
          document.getElementById("moneyDisplay").innerText = money;
        } else {
          enemy.draw();
        }
      }

      requestAnimationFrame(gameLoop);
    }

    gameLoop();
  </script>
</body>
</html>
