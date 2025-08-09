<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import Matter from 'matter-js';

  let canvas: HTMLCanvasElement;
  let width = 900;
  let height = 600;

  // Game config
  const PITCH_MARGIN = 20;
  const WALL_THICK = 40;
  const DISC_R = 18;
  const BALL_R = 10;
  const MAX_FLICK = 12;      // cap initial velocity “power”
  const LINEAR_DAMPING = 0.025; // air-like friction
  const STOP_EPS = 0.03;     // velocities below this → considered stopped
  const GOAL_WIDTH = 160;

  // Matter aliases
  const {
    Engine, Render, Runner, Bodies, Composite, Body, Events, Vector, Constraint, Mouse, MouseConstraint
  } = Matter;

  let engine: Matter.Engine;
  let render: Matter.Render;
  let runner: Matter.Runner;

  // Bodies
  let ball: Matter.Body;
  let teamA: Matter.Body[] = [];
  let teamB: Matter.Body[] = [];

  // Aim UI state
  let aiming = false;
  let aimedBody: Matter.Body | null = null;
  let mousePos: Matter.Vector = { x: 0, y: 0 };

  // Turn logic
  let currentSide: 'A' | 'B' = 'A';
  let waitingForSettle = false;
  let scoreA = 0, scoreB = 0;

  const center = (x: number, y: number) => ({ x, y });

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

  function isOwnPiece(b: Matter.Body) {
    return currentSide === 'A' ? teamA.includes(b) : teamB.includes(b);
  }

  function clampFlick(vec: Matter.Vector) {
    const mag = Math.hypot(vec.x, vec.y);
    if (mag <= MAX_FLICK) return vec;
    const s = MAX_FLICK / (mag || 1);
    return { x: vec.x * s, y: vec.y * s };
  }

  function makeDisc(x: number, y: number, r: number, color: string) {
    const d = Bodies.circle(x, y, r, {
      restitution: 0.92,
      frictionAir: LINEAR_DAMPING,
      frictionStatic: 0.0,
      friction: 0.0,
      render: { fillStyle: color, strokeStyle: '#111', lineWidth: 1.5 }
    });
    return d;
  }

  function addPitch(world: Matter.World) {
    // Walls around pitch (offscreen thick to keep inside)
    const left   = Bodies.rectangle(PITCH_MARGIN - WALL_THICK/2, height/2, WALL_THICK, height, { isStatic: true, render: { fillStyle: '#2b2b2b' } });
    const right  = Bodies.rectangle(width - PITCH_MARGIN + WALL_THICK/2, height/2, WALL_THICK, height, { isStatic: true, render: { fillStyle: '#2b2b2b' } });
    const top    = Bodies.rectangle(width/2, PITCH_MARGIN - WALL_THICK/2, width, WALL_THICK, { isStatic: true, render: { fillStyle: '#2b2b2b' } });
    const bottom = Bodies.rectangle(width/2, height - PITCH_MARGIN + WALL_THICK/2, width, WALL_THICK, { isStatic: true, render: { fillStyle: '#2b2b2b' } });

    Composite.add(world, [left, right, top, bottom]);

    // Goals (just gaps in top/bottom walls via slim posts)
    const postWidth = 12;
    const goalX1 = width/2 - GOAL_WIDTH/2;
    const goalX2 = width/2 + GOAL_WIDTH/2;

    const topL = Bodies.rectangle(goalX1/2, PITCH_MARGIN + 10, goalX1 - PITCH_MARGIN, 8, { isStatic: true, render: { fillStyle: '#444' } });
    const topR = Bodies.rectangle((goalX2 + width - PITCH_MARGIN)/2, PITCH_MARGIN + 10, (width - PITCH_MARGIN) - goalX2, 8, { isStatic: true, render: { fillStyle: '#444' } });

    const botL = Bodies.rectangle(goalX1/2, height - PITCH_MARGIN - 10, goalX1 - PITCH_MARGIN, 8, { isStatic: true, render: { fillStyle: '#444' } });
    const botR = Bodies.rectangle((goalX2 + width - PITCH_MARGIN)/2, height - PITCH_MARGIN - 10, (width - PITCH_MARGIN) - goalX2, 8, { isStatic: true, render: { fillStyle: '#444' } });

    Composite.add(world, [topL, topR, botL, botR]);
  }

  function resetPositions() {
    // center ball
    Body.setPosition(ball, center(width/2, height/2));
    Body.setVelocity(ball, { x: 0, y: 0 });
    Body.setAngularVelocity(ball, 0);

    // line up teams
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
    // Ball crosses top or bottom gap area
    const x = ball.position.x;
    const y = ball.position.y;
    const withinGap = x > (width/2 - GOAL_WIDTH/2) && x < (width/2 + GOAL_WIDTH/2);

    if (withinGap && y < PITCH_MARGIN + 6) {
      // Team A scores into top goal
      scoreA += 1;
      resetPositions();
    } else if (withinGap && y > height - PITCH_MARGIN - 6) {
      // Team B scores into bottom goal
      scoreB += 1;
      resetPositions();
    }
  }

  function drawPitchOverlay(ctx: CanvasRenderingContext2D) {
    // Field
    ctx.save();
    ctx.fillStyle = '#0b7d3a';
    ctx.fillRect(PITCH_MARGIN, PITCH_MARGIN, width - 2*PITCH_MARGIN, height - 2*PITCH_MARGIN);

    // Midline + circle
    ctx.strokeStyle = 'rgba(255,255,255,0.7)';
    ctx.lineWidth = 2;
    ctx.beginPath();
    ctx.moveTo(PITCH_MARGIN, height/2);
    ctx.lineTo(width - PITCH_MARGIN, height/2);
    ctx.stroke();

    ctx.beginPath();
    ctx.arc(width/2, height/2, 70, 0, Math.PI*2);
    ctx.stroke();

    // Goals
    ctx.beginPath();
    ctx.moveTo(width/2 - GOAL_WIDTH/2, PITCH_MARGIN);
    ctx.lineTo(width/2 + GOAL_WIDTH/2, PITCH_MARGIN);
    ctx.moveTo(width/2 - GOAL_WIDTH/2, height - PITCH_MARGIN);
    ctx.lineTo(width/2 + GOAL_WIDTH/2, height - PITCH_MARGIN);
    ctx.stroke();

    // Aim line
    if (aiming && aimedBody) {
      const from = aimedBody.position;
      const dir = { x: from.x - mousePos.x, y: from.y - mousePos.y };
      const capped = clampFlick(dir);
      const preview = { x: from.x + capped.x * 6, y: from.y + capped.y * 6 };

      ctx.strokeStyle = 'rgba(255,255,255,0.85)';
      ctx.lineWidth = 2;
      ctx.setLineDash([6, 6]);
      ctx.beginPath();
      ctx.moveTo(from.x, from.y);
      ctx.lineTo(preview.x, preview.y);
      ctx.stroke();
      ctx.setLineDash([]);

      // power dot
      ctx.beginPath();
      ctx.arc(preview.x, preview.y, 4, 0, Math.PI*2);
      ctx.fillStyle = 'white';
      ctx.fill();
    }

    // Score + turn badge
    ctx.fillStyle = 'white';
    ctx.font = '600 16px system-ui, -apple-system, Segoe UI, Roboto';
    ctx.fillText(`A ${scoreA} - ${scoreB} B`, 16, 26);
    ctx.fillText(`Turn: ${currentSide}`, 16, 48);
    ctx.restore();
  }

  function worldFromMouse(e: MouseEvent): Matter.Vector {
    const rect = canvas.getBoundingClientRect();
    return {
      x: (e.clientX - rect.left) * (canvas.width / rect.width),
      y: (e.clientY - rect.top) * (canvas.height / rect.height)
    };
  }

  function pickBodyAt(pos: Matter.Vector): Matter.Body | null {
    const bodies = Composite.allBodies(engine.world);
    for (const b of bodies) {
      if (b.circleRadius) {
        const d = Math.hypot(b.position.x - pos.x, b.position.y - pos.y);
        if (d <= b.circleRadius + 1) return b;
      }
    }
    return null;
  }

  onMount(() => {
    engine = Engine.create({ gravity: { x: 0, y: 0 } });

    render = Render.create({
      canvas,
      engine,
      options: {
        width,
        height,
        wireframes: false,
        background: '#0b7d3a' // we’ll paint the field overlay each frame
      }
    });

    // Pitch + walls
    addPitch(engine.world);

    // Ball
    ball = Bodies.circle(width/2, height/2, BALL_R, {
      restitution: 0.92,
      frictionAir: LINEAR_DAMPING,
      render: { fillStyle: '#fff', strokeStyle: '#111', lineWidth: 1.2 }
    });

    // Teams
    teamA = [
      makeDisc(width/2 - 80, height - 160, DISC_R, '#e63946'),
      makeDisc(width/2,       height - 160, DISC_R, '#e63946'),
      makeDisc(width/2 + 80, height - 160, DISC_R, '#e63946')
    ];
    teamB = [
      makeDisc(width/2 - 80, 160, DISC_R, '#1d4ed8'),
      makeDisc(width/2,       160, DISC_R, '#1d4ed8'),
      makeDisc(width/2 + 80, 160, DISC_R, '#1d4ed8')
    ];

    Composite.add(engine.world, [ball, ...teamA, ...teamB]);

    // Step & render
    runner = Runner.create({ isFixed: true, delta: 1000/120 }); // 120 Hz physics
    Runner.run(runner, engine);
    Render.run(render);

    // Per-tick hooks
    Events.on(engine, 'afterUpdate', () => {
      // Check goal after movement
      checkGoals();

      // draw overlay on top
      const ctx = render.context as CanvasRenderingContext2D;
      drawPitchOverlay(ctx);

      switchTurnIfSettle();
    });

    // Input: click/drag/release to flick
    const down = (e: MouseEvent) => {
      if (waitingForSettle) return;
      mousePos = worldFromMouse(e);
      const picked = pickBodyAt(mousePos);
      if (picked && picked !== ball && isOwnPiece(picked)) {
        aimedBody = picked;
        aiming = true;
      }
    };
    const move = (e: MouseEvent) => {
      mousePos = worldFromMouse(e);
    };
    const up = (e: MouseEvent) => {
      if (aiming && aimedBody) {
        const from = aimedBody.position;
        const raw = { x: from.x - mousePos.x, y: from.y - mousePos.y };
        const capped = clampFlick(raw);

        // Apply impulse as setting velocity directly for arcade feel
        Body.setVelocity(aimedBody, capped);

        aiming = false;
        aimedBody = null;
        waitingForSettle = true;
      }
    };

    canvas.addEventListener('mousedown', down);
    window.addEventListener('mousemove', move);
    window.addEventListener('mouseup', up);

    onDestroy(() => {
      canvas.removeEventListener('mousedown', down);
      window.removeEventListener('mousemove', move);
      window.removeEventListener('mouseup', up);
      Render.stop(render);
      Runner.stop(runner);
      // @ts-ignore
      render.canvas = null;
      // @ts-ignore
      render.context = null;
      // @ts-ignore
      render.textures = {};
    });
  });

  function reset() {
    resetPositions();
  }
</script>

<style>
  .wrap {
    display: grid;
    gap: 12px;
    justify-items: start;
  }
  canvas {
    image-rendering: crisp-edges;
    border-radius: 10px;
    box-shadow: 0 6px 24px rgba(0,0,0,.25);
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
    <span class="badge">Single-player MVP</span>
    <button on:click={reset}>Reset</button>
  </div>
  <canvas bind:this={canvas} width={width} height={height}></canvas>
</div>
