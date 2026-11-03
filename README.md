# deva-html-
MY HTML Practice And Learning Codes
DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Road Rush: Kolkata ChaseA</title>
    <style>
        :root { --ink: #18232b; --cream: #fff4dc; --yellow: #f6b91a; --red: #d94635; }
        * { box-sizing: border-box; }
        html, body { margin: 0; min-height: 100%; background: #0d2930; font-family: Georgia, 'Times New Roman', serif; color: var(--cream); }
        body { display: grid; place-items: center; overflow: hidden; }
        #game-shell { position: relative; width: min(100vw, 900px); height: min(100vh, 760px); min-height: 520px; overflow: hidden; background: #78b4ad; box-shadow: 0 0 0 6px #e4bb55, 0 18px 50px #06171dcc; }
        canvas { display: block; width: 100%; height: 100%; }
        .hud { position: absolute; inset: 0 0 auto; display: flex; justify-content: space-between; align-items: flex-start; padding: 18px 22px; pointer-events: none; text-shadow: 2px 2px 0 #193339; }
        .brand { letter-spacing: .09em; font-size: clamp(16px, 2.5vw, 25px); font-weight: bold; color: #ffdb5e; }
        .brand span { display: block; color: var(--cream); font-size: .56em; letter-spacing: .22em; margin-top: 2px; }
        .stats { display: flex; gap: 18px; text-align: right; font-family: 'Trebuchet MS', sans-serif; font-size: clamp(12px, 2vw, 16px); font-weight: bold; }
        .stats b { color: #ffdb5e; display: block; font-size: 1.3em; }
        #message { position: absolute; inset: 0; display: grid; place-items: center; background: #0b2529aa; opacity: 0; pointer-events: none; transition: opacity .2s; }
        #message.visible { opacity: 1; pointer-events: auto; }
        .panel { width: min(86%, 470px); padding: 30px 28px 26px; text-align: center; background: #193b3e; border: 3px solid #f5c84c; box-shadow: 10px 10px 0 #101f25; }
        h1 { margin: 0 0 8px; color: #ffcf3f; font-size: clamp(27px, 6vw, 52px); line-height: .95; letter-spacing: -.03em; }
        .panel p { margin: 12px 0 22px; font: 14px/1.5 'Trebuchet MS', sans-serif; color: #fbeed5; }
        button { border: 0; cursor: pointer; font: bold 16px 'Trebuchet MS', sans-serif; color: #16262b; background: #f6bd27; padding: 13px 24px; box-shadow: 4px 4px 0 #9c552b; }
        button:active { transform: translate(2px, 2px); box-shadow: 2px 2px 0 #9c552b; }
        .touch { position: absolute; inset: auto 0 22px; display: flex; justify-content: space-between; padding: 0 24px; pointer-events: none; }
        .touch button { pointer-events: auto; width: 62px; height: 54px; padding: 0; opacity: .82; font-size: 25px; border-radius: 4px; }
        .touch .cluster { display: flex; gap: 10px; }
        @media (min-width: 720px) { .touch { display: none; } }
    </style>
</head>
<body>
    <main id="game-shell">
        <canvas id="game" aria-label="Road Rush: Kolkata Chase game"></canvas>
        <header class="hud">
            <div class="brand">ROAD RUSH <span>KOLKATA CHASE</span></div>
            <div class="stats"><div>SCORE<b id="score">00000</b></div><div>BEST<b id="best">00000</b></div></div>
        </header>
        <div id="message" class="visible">
            <section class="panel">
                <h1 id="message-title">Road Rush</h1>
                <p id="message-copy">Dodge the traffic, collect the sweets, and chase your high score through the city of joy.</p>
                <button id="start">START CHASE</button>
            </section>
        </div>
        <nav class="touch" aria-label="Touch controls">
            <div class="cluster"><button data-key="left" aria-label="Move left">&#8592;</button><button data-key="right" aria-label="Move right">&#8594;</button></div>
            <div class="cluster"><button data-key="brake" aria-label="Brake">&#9665;</button><button data-key="boost" aria-label="Boost">&#9655;</button></div>
        </nav>
    </main>
    <script>
        (() => {
            'use strict';
            const canvas = document.getElementById('game');
            const ctx = canvas.getContext('2d');
            const shell = document.getElementById('game-shell');
            const scoreEl = document.getElementById('score');
            const bestEl = document.getElementById('best');
            const message = document.getElementById('message');
            const title = document.getElementById('message-title');
            const copy = document.getElementById('message-copy');
            const start = document.getElementById('start');
            const keys = { left: false, right: false, boost: false, brake: false };
            const colors = ['#e4583f', '#3c88a4', '#efc94c', '#f28f3c', '#835a93'];
            let width = 900, height = 760, dpr = 1, raf = 0, last = 0;
            let state = 'menu', score = 0, best = Number(localStorage.getItem('kolkataRushBest') || 0);
            let speed = 260, roadOffset = 0, spawnTimer = 0, sweetTimer = 0, traffic = [], sweets = [], sparks = [];
            const player = { lane: 1, x: 0, y: 0, targetX: 0, w: 48, h: 78, color: '#f3c52c' };
            bestEl.textContent = String(best).padStart(5, '0');

            function resize() {
                const rect = shell.getBoundingClientRect();
                dpr = Math.min(window.devicePixelRatio || 1, 2); width = rect.width; height = rect.height;
                canvas.width = Math.floor(width * dpr); canvas.height = Math.floor(height * dpr); ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
                player.y = height - 132; player.targetX = laneX(player.lane); if (!player.x) player.x = player.targetX;
            }
            function laneX(lane) { return width / 2 + (lane - 1) * Math.min(width * .19, 142); }
            function roadBounds() { const road = Math.min(width * .76, 620); return { left: (width - road) / 2, right: (width + road) / 2 }; }
            function startGame() {
                state = 'playing'; score = 0; speed = 260; roadOffset = 0; spawnTimer = .3; sweetTimer = 1.3; traffic = []; sweets = []; sparks = [];
                player.lane = 1; player.targetX = laneX(1); player.x = player.targetX; message.classList.remove('visible'); last = performance.now(); cancelAnimationFrame(raf); raf = requestAnimationFrame(loop);
            }
            function endGame() {
                state = 'over'; best = Math.max(best, Math.floor(score)); localStorage.setItem('kolkataRushBest', best); bestEl.textContent = String(best).padStart(5, '0');
                title.textContent = 'Chase Over'; copy.textContent = `You scored ${Math.floor(score)} points. The city is waiting for another run.`; start.textContent = 'RACE AGAIN'; message.classList.add('visible');
            }
            function move(direction) { if (state !== 'playing') return; player.lane = Math.max(0, Math.min(2, player.lane + direction)); player.targetX = laneX(player.lane); }
            function spawnTraffic() { traffic.push({ lane: Math.floor(Math.random() * 3), y: -100, w: 45, h: 74, color: colors[Math.floor(Math.random() * colors.length)], wobble: Math.random() * 10 }); }
            function spawnSweet() { sweets.push({ lane: Math.floor(Math.random() * 3), y: -24, size: 13, spin: Math.random() * 6 }); }
            function update(dt) {
                const multiplier = keys.boost ? 1.3 : keys.brake ? .68 : 1;
                speed = Math.min(520, speed + dt * 3); roadOffset = (roadOffset + speed * multiplier * dt) % 80; score += dt * speed * .08 * multiplier;
                player.x += (player.targetX - player.x) * Math.min(1, dt * 12); spawnTimer -= dt; sweetTimer -= dt;
                if (spawnTimer <= 0) { spawnTraffic(); spawnTimer = Math.max(.43, 1.02 - speed / 850) + Math.random() * .3; }
                if (sweetTimer <= 0) { spawnSweet(); sweetTimer = 1.6 + Math.random() * 1.8; }
                traffic.forEach(car => car.y += speed * multiplier * dt); sweets.forEach(item => { item.y += speed * multiplier * dt; item.spin += dt * 5; });
                traffic = traffic.filter(car => car.y < height + 110); sweets = sweets.filter(item => item.y < height + 40);
                const playerRect = { x: player.x - 24, y: player.y - 39, w: 48, h: 78 };
                for (const car of traffic) { const carX = laneX(car.lane); if (hit(playerRect, { x: carX - 22, y: car.y - 37, w: 44, h: 74 })) { burst(player.x, player.y); endGame(); return; } }
                sweets = sweets.filter(item => { const collected = hit(playerRect, { x: laneX(item.lane) - 14, y: item.y - 14, w: 28, h: 28 }); if (collected) { score += 100; burst(laneX(item.lane), item.y, '#ffdb5e'); } return !collected; });
                sparks.forEach(s => { s.x += s.vx * dt; s.y += s.vy * dt; s.life -= dt; }); sparks = sparks.filter(s => s.life > 0);
                scoreEl.textContent = String(Math.floor(score)).padStart(5, '0');
            }
            function hit(a, b) { return a.x < b.x + b.w - 7 && a.x + a.w > b.x + 7 && a.y < b.y + b.h - 9 && a.y + a.h > b.y + 9; }
            function burst(x, y, color = '#ef6045') { for (let i = 0; i < 18; i++) sparks.push({ x, y, vx: (Math.random() - .5) * 220, vy: (Math.random() - .5) * 220, life: .65, color }); }
            function draw() { ctx.clearRect(0, 0, width, height); drawCity(); drawRoad(); sweets.forEach(drawSweet); traffic.forEach(drawCar); drawPlayer(); sparks.forEach(drawSpark); }
            function drawCity() {
                ctx.fillStyle = '#83b5a7'; ctx.fillRect(0, 0, width, height);
                ctx.fillStyle = '#5c968e'; for (let x = 0; x < width; x += 62) { const h = 40 + (x * 17 % 65); ctx.fillRect(x, 108 - h, 45, h); ctx.fillStyle = x % 124 ? '#6da39a' : '#4e8986'; }
                ctx.fillStyle = '#e3c16b'; ctx.fillRect(0, 109, width, 5); ctx.fillStyle = '#e7d49a'; ctx.fillRect(0, 114, width, 3);
                ctx.fillStyle = '#d3543a'; ctx.font = 'bold 13px Trebuchet MS'; ctx.fillText('কলকাতা', 24, 94); ctx.fillText('CITY OF JOY', width - 108, 94);
            }
            function drawRoad() { const b = roadBounds(); ctx.fillStyle = '#263b40'; ctx.fillRect(b.left, 0, b.right - b.left, height); ctx.fillStyle = '#d2aa58'; ctx.fillRect(b.left, 0, 8, height); ctx.fillRect(b.right - 8, 0, 8, height); ctx.fillStyle = '#e9d18a'; for (let lane = 1; lane < 3; lane++) { const x = b.left + (b.right - b.left) * lane / 3; for (let y = -80 + roadOffset; y < height; y += 80) ctx.fillRect(x - 2, y, 4, 42); } }
            function drawCar(car) { const x = laneX(car.lane), y = car.y; ctx.save(); ctx.translate(x, y); ctx.fillStyle = '#14262b'; ctx.fillRect(-25, -35, 50, 70); ctx.fillStyle = car.color; ctx.fillRect(-21, -37, 42, 74); ctx.fillStyle = '#bfe1d5'; ctx.fillRect(-15, -24, 30, 18); ctx.fillStyle = '#172b31'; ctx.fillRect(-17, 18, 8, 14); ctx.fillRect(9, 18, 8, 14); ctx.fillStyle = '#fff1a8'; ctx.fillRect(-17, -33, 9, 5); ctx.fillRect(8, -33, 9, 5); ctx.restore(); }
            function drawPlayer() { ctx.save(); ctx.translate(player.x, player.y); ctx.fillStyle = '#132c32'; ctx.fillRect(-27, -39, 54, 78); ctx.fillStyle = player.color; ctx.fillRect(-23, -39, 46, 78); ctx.fillStyle = '#1c484c'; ctx.fillRect(-16, -25, 32, 19); ctx.fillStyle = '#f7e8b2'; ctx.fillRect(-21, 0, 42, 5); ctx.fillStyle = '#141f23'; ctx.fillRect(-19, 24, 9, 14); ctx.fillRect(10, 24, 9, 14); ctx.fillStyle = '#fff2ad'; ctx.fillRect(-18, -35, 10, 5); ctx.fillRect(8, -35, 10, 5); ctx.fillStyle = '#dc4a32'; ctx.fillRect(-8, -42, 16, 5); ctx.restore(); }
            function drawSweet(item) { const x = laneX(item.lane); ctx.save(); ctx.translate(x, item.y); ctx.rotate(item.spin); ctx.fillStyle = '#d84736'; ctx.fillRect(-11, -11, 22, 22); ctx.fillStyle = '#ffd457'; ctx.fillRect(-7, -7, 14, 14); ctx.restore(); }
            function drawSpark(s) { ctx.globalAlpha = Math.max(0, s.life); ctx.fillStyle = s.color; ctx.fillRect(s.x, s.y, 5, 5); ctx.globalAlpha = 1; }
            function loop(now) { if (state !== 'playing') return; const dt = Math.min(.035, (now - last) / 1000); last = now; update(dt); draw(); raf = requestAnimationFrame(loop); }
            function press(name, value) { keys[name] = value; }
            window.addEventListener('resize', resize); window.addEventListener('keydown', event => { const key = event.key.toLowerCase(); if (['arrowleft', 'a'].includes(key)) { move(-1); event.preventDefault(); } if (['arrowright', 'd'].includes(key)) { move(1); event.preventDefault(); } if (key === 'shift') press('boost', true); if (key === ' ') { if (state !== 'playing') startGame(); event.preventDefault(); } });
            window.addEventListener('keyup', event => { if (event.key.toLowerCase() === 'shift') press('boost', false); });
            document.querySelectorAll('[data-key]').forEach(button => { const key = button.dataset.key; button.addEventListener('pointerdown', event => { event.preventDefault(); if (key === 'left') move(-1); else if (key === 'right') move(1); else press(key, true); }); button.addEventListener('pointerup', () => press(key, false)); button.addEventListener('pointercancel', () => press(key, false)); });
            start.addEventListener('click', startGame); resize(); draw();
        })();
    </script>
</body>
</html>
