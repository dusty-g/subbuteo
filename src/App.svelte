<script>
  import { onMount } from 'svelte';
  import Matter from 'matter-js';

  let canvas;
  let width = 900;
  let height = 600;

  // Config
  const PITCH_MARGIN = 20;
  const WALL_THICK = 40;
  const DISC_R = 18;
  const BALL_R = 10;
  const MAX_FLICK = 24;
  // Higher damping slows players faster while keeping the ball lively
  const LINEAR_DAMPING = 0.05;
  const BALL_LINEAR_DAMPING = 0.025;
  const STOP_EPS = 0.03;
  const GOAL_WIDTH = 160;

  // Matter aliases
  const {
    Engine, Render, Runner, Bodies, Composite, Body, Events
  } = Matter;

  let engine, render, runner;
  let ball;
  let teamA = [];
  let teamB = [];

  // Aim/UI
  let aiming = false;
  let aimedBody = null;
  let mousePos = { x: 0, y: 0 };

  // Turns/score
  let currentSide = 'A';
  let waitingForSettle = false;
  let scoreA = 0, scoreB = 0;

  $: turnLabel = currentSide === 'A' ? 'A (Red)' : 'B (Blue)';
  $: turnColor = currentSide === 'A' ? '#e63946' : '#1d4ed8';

  const center = (x, y) => ({ x, y });

  function allStopped() {
    const bodies = [ball, ...teamA, ...teamB];
    return bodies.every(b => Math.hypot(b.velocity.x, b.velocity.y) < STOP_EPS);
  }

  function switchTurnIfSettle() {
    if (waitingForSettle && allStopped()) {
      waitingForSettle = false;
      currentSide = currentSide === 'A' ? 'B' : 'A';
    }
  }

  function isOwnPiece(b) {
    return currentSide === 'A' ? teamA.includes(b) : teamB.includes(b);
  }

  function clampFlick(vec) {
    const mag = Math.hypot(vec.x, vec.y);
    if (mag <= MAX_FLICK) return vec;
    const s = MAX_FLICK / (mag || 1);
    return { x: vec.x * s, y: vec.y * s };
  }

  function makeDisc(x, y, r, color) {
    return Bodies.circle(x, y, r, {
      restitution: 0.92,
      frictionAir: LINEAR_DAMPING,
      frictionStatic: 0,
      friction: 0,
      render: { fillStyle: color, strokeStyle: '#111', lineWidth: 1.5 }
    });
  }

  function addPitch(world) {
    const left   = Bodies.rectangle(PITCH_MARGIN - WALL_THICK/2, height/2, WALL_THICK, height, { isStatic: true, render: { fillStyle: '#2b2b2b' } });
    const right  = Bodies.rectangle(width - PITCH_MARGIN + WALL_THICK/2, height/2, WALL_THICK, height, { isStatic: true, render: { fillStyle: '#2b2b2b' } });
    const top    = Bodies.rectangle(width/2, PITCH_MARGIN - WALL_THICK/2, width, WALL_THICK, { isStatic: true, render: { fillStyle: '#2b2b2b' } });
    const bottom = Bodies.rectangle(width/2, height - PITCH_MARGIN + WALL_THICK/2, width, WALL_THICK, { isStatic: true, render: { fillStyle: '#2b2b2b' } });

    Composite.add(world, [left, right, top, bottom]);

    // Top/bottom goal gaps via slim bars
    const goalX1 = width/2 - GOAL_WIDTH/2;
    const goalX2 = width/2 + GOAL_WIDTH/2;

    const topL = Bodies.rectangle(goalX1/2, PITCH_MARGIN + 10, goalX1 - PITCH_MARGIN, 8, { isStatic: true, render: { fillStyle: '#444' } });
    const topR = Bodies.rectangle((goalX2 + width - PITCH_MARGIN)/2, PITCH_MARGIN + 10, (width - PITCH_MARGIN) - goalX2, 8, { isStatic: true, render: { fillStyle: '#444' } });
    const botL = Bodies.rectangle(goalX1/2, height - PITCH_MARGIN - 10, goalX1 - PITCH_MARGIN, 8, { isStatic: true, render: { fillStyle: '#444' } });
    const botR = Bodies.rectangle((goalX2 + width - PITCH_MARGIN)/2, height - PITCH_MARGIN - 10, (width - PITCH_MARGIN) - goalX2, 8, { isStatic: true, render: { fillStyle: '#444' } });

    Composite.add(world, [topL, topR, botL, botR]);
  }

  function resetPositions() {
    // Ball center
    Body.setPosition(ball, center(width/2, height/2));
    Body.setVelocity(ball, { x: 0, y: 0 });
    Body.setAngularVelocity(ball, 0);

    // Teams
    const yA = height - 160;
    const yB = 160;
    const spacing = 80;
    [ -1, 0, 1 ].forEach((i, idx) => {
      const x = width/2 + i*spacing;
      const a = teamA[idx];
      Body.setPosition(a, center(x, yA));
      Body.setVelocity(a, { x: 0, y: 0 });
      Body.setAngularVelocity(a, 0);

      const b = teamB[idx];
      Body.setPosition(b, center(x, yB));
      Body.setVelocity(b, { x: 0, y: 0 });
      Body.setAngularVelocity(b, 0);
    });

    currentSide = 'A';
    waitingForSettle = false;
  }

  function checkGoals() {
    const x = ball.position.x;
    const y = ball.position.y;
    const withinGap = x > (width/2 - GOAL_WIDTH/2) && x < (width/2 + GOAL_WIDTH/2);

    if (withinGap && y < PITCH_MARGIN + 6) {
      // A scores (top)
      scoreA += 1;
      resetPositions();
    } else if (withinGap && y > height - PITCH_MARGIN - 6) {
      // B scores (bottom)
      scoreB += 1;
      resetPositions();
    }
  }

  function worldFromPointer(e) {
    const rect = canvas.getBoundingClientRect();
    let clientX;
    let clientY;
    if (e.touches && e.touches.length) {
      clientX = e.touches[0].clientX;
      clientY = e.touches[0].clientY;
    } else {
      clientX = e.clientX;
      clientY = e.clientY;
    }
    return {
      x: (clientX - rect.left) * (canvas.width / rect.width),
      y: (clientY - rect.top) * (canvas.height / rect.height)
    };
  }

  function pickBodyAt(pos) {
    const bodies = Composite.allBodies(engine.world);
    for (const b of bodies) {
      if (b.circleRadius) {
        const d = Math.hypot(b.position.x - pos.x, b.position.y - pos.y);
        if (d <= b.circleRadius + 1) return b;
      }
    }
    return null;
  }

  function attachInputHandlers() {
    const down = e => {
      e.preventDefault();
      if (waitingForSettle) return;
      mousePos = worldFromPointer(e);
      const picked = pickBodyAt(mousePos);
      if (picked && picked !== ball && isOwnPiece(picked)) {
        aimedBody = picked;
        aiming = true;
      }
    };
    const move = e => {
      e.preventDefault();
      mousePos = worldFromPointer(e);
    };
    const up = e => {
      e.preventDefault();
      if (aiming && aimedBody) {
        const from = aimedBody.position;
        const raw = { x: from.x - mousePos.x, y: from.y - mousePos.y };
        const capped = clampFlick(raw);
        Body.setVelocity(aimedBody, capped);
        aiming = false;
        aimedBody = null;
        waitingForSettle = true;
      }
    };

    canvas.addEventListener('pointerdown', down, { passive: false });
    window.addEventListener('pointermove', move, { passive: false });
    window.addEventListener('pointerup', up, { passive: false });

    return () => {
      canvas.removeEventListener('pointerdown', down);
      window.removeEventListener('pointermove', move);
      window.removeEventListener('pointerup', up);
    };
  }

  function reset() {
    resetPositions();
  }

  onMount(() => {
    canvas.style.touchAction = 'none';
    engine = Engine.create({ gravity: { x: 0, y: 0 } });

    render = Render.create({
      canvas,
      engine,
      options: {
        width,
        height,
        wireframes: false,
        background: '#0b7d3a'
      }
    });

    // Pitch + walls
    addPitch(engine.world);

    // Ball
    ball = Bodies.circle(width/2, height/2, BALL_R, {
      restitution: 0.92,
      frictionAir: BALL_LINEAR_DAMPING,
      render: { fillStyle: '#fff', strokeStyle: '#111', lineWidth: 1.2 }
    });

    // Teams
    teamA = [
      makeDisc(width/2 - 80, height - 160, DISC_R, '#e63946'),
      makeDisc(width/2,       height - 160, DISC_R, '#e63946'),
      makeDisc(width/2 + 80,  height - 160, DISC_R, '#e63946')
    ];
    teamB = [
      makeDisc(width/2 - 80, 160, DISC_R, '#1d4ed8'),
      makeDisc(width/2,       160, DISC_R, '#1d4ed8'),
      makeDisc(width/2 + 80,  160, DISC_R, '#1d4ed8')
    ];

    Composite.add(engine.world, [ball, ...teamA, ...teamB]);

    // Physics & render
    runner = Runner.create({ isFixed: true, delta: 1000/120 });
    Runner.run(runner, engine);
    Render.run(render);

    // Tick hooks
    Events.on(engine, 'afterUpdate', () => {
      checkGoals();
      switchTurnIfSettle();
    });

    // Field under bodies
    Events.on(render, 'beforeRender', () => {
      const ctx = render.context;
      ctx.save();

      // field
      ctx.fillStyle = '#0b7d3a';
      ctx.fillRect(PITCH_MARGIN, PITCH_MARGIN, width - 2*PITCH_MARGIN, height - 2*PITCH_MARGIN);

      // midline + circle
      ctx.strokeStyle = 'rgba(255,255,255,0.7)';
      ctx.lineWidth = 2;
      ctx.beginPath();
      ctx.moveTo(PITCH_MARGIN, height/2);
      ctx.lineTo(width - PITCH_MARGIN, height/2);
      ctx.stroke();

      ctx.beginPath();
      ctx.arc(width/2, height/2, 70, 0, Math.PI*2);
      ctx.stroke();

      // goal lines
      ctx.beginPath();
      ctx.moveTo(width/2 - GOAL_WIDTH/2, PITCH_MARGIN);
      ctx.lineTo(width/2 + GOAL_WIDTH/2, PITCH_MARGIN);
      ctx.moveTo(width/2 - GOAL_WIDTH/2, height - PITCH_MARGIN);
      ctx.lineTo(width/2 + GOAL_WIDTH/2, height - PITCH_MARGIN);
      ctx.stroke();

      ctx.restore();
    });

    // Aim + HUD on top
    Events.on(render, 'afterRender', () => {
      const ctx = render.context;
      ctx.save();

      if (aiming && aimedBody) {
        const from = aimedBody.position;
        const dir = { x: from.x - mousePos.x, y: from.y - mousePos.y };
        const capped = clampFlick(dir);
        const preview = { x: from.x + capped.x * 6, y: from.y + capped.y * 6 };
        const power = Math.round((Math.hypot(capped.x, capped.y) / MAX_FLICK) * 100);

        // selection halo
        ctx.strokeStyle = 'rgba(255,255,255,0.6)';
        ctx.lineWidth = 2;
        ctx.beginPath();
        ctx.arc(from.x, from.y, (aimedBody.circleRadius || 18) + 6, 0, Math.PI*2);
        ctx.stroke();

        // aim line
        ctx.strokeStyle = 'rgba(255,255,255,0.85)';
        ctx.setLineDash([6, 6]);
        ctx.beginPath();
        ctx.moveTo(from.x, from.y);
        ctx.lineTo(preview.x, preview.y);
        ctx.stroke();
        ctx.setLineDash([]);

        // tip + power label
        ctx.beginPath();
        ctx.arc(preview.x, preview.y, 4, 0, Math.PI*2);
        ctx.fillStyle = 'white';
        ctx.fill();

        ctx.font = '600 14px system-ui, -apple-system, Segoe UI, Roboto';
        ctx.fillStyle = 'white';
        ctx.fillText(String(power), preview.x + 6, preview.y + 4);
      }

      // Score
      ctx.fillStyle = 'white';
      ctx.font = '600 16px system-ui, -apple-system, Segoe UI, Roboto';
      ctx.fillText(`A ${scoreA} - ${scoreB} B`, 16, 26);

      ctx.restore();
    });

    const detach = attachInputHandlers();

    // Cleanup
    return () => {
      detach();
      Render.stop(render);
      Runner.stop(runner);
      render.canvas = null;
      render.context = null;
      render.textures = {};
    };
  });
</script>

<style>
  .wrap {
    display: grid;
    gap: 12px;
    justify-items: start;
    padding: 12px;
    width: 100%;
  }
  canvas {
    image-rendering: crisp-edges;
    border-radius: 10px;
    box-shadow: 0 6px 24px rgba(0,0,0,.25);
    max-width: 100%;
    height: auto;
  }
  .hud {
    display: flex;
    gap: 8px;
    align-items: center;
  }
  .badge {
    font: 600 12px system-ui, -apple-system, Segoe UI, Roboto;
    padding: 4px 8px;
    border-radius: 999px;
    background: #111;
    color: #fff;
    opacity: .9;
  }
  button {
    font: 600 14px system-ui, -apple-system, Segoe UI, Roboto;
    padding: 8px 12px;
    border-radius: 8px;
    border: none;
    background: #1d4ed8;
    color: white;
    cursor: pointer;
  }
  button:active { transform: translateY(1px); }
</style>

<div class="wrap">
  <div class="hud">
    <span class="badge" style="background: {turnColor}">Turn: {turnLabel}</span>
    <button on:click={reset}>Reset</button>
  </div>
  <canvas bind:this={canvas} width={width} height={height}></canvas>
</div>
