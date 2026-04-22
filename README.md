# Neon-Rush-
Neon Rush a luxury game with extra effect and editions 
NEON RUSH
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
<title>NEON RUSH — Highway Runner</title>
<link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@400;700;900&display=swap" rel="stylesheet">
<style>
  *{margin:0;padding:0;box-sizing:border-box}
  html,body{width:100%;height:100%;overflow:hidden;background:#000}
  body{display:flex;justify-content:center;align-items:center}
  canvas{display:block;touch-action:none;max-height:100vh}
</style>
</head>
<body>
<canvas id="c"></canvas>
<script>
/* =============================================
   NEON RUSH — Highway Runner
   Luxury Traffic Runner Game
============================================= */

const canvas = document.getElementById('c');
const ctx    = canvas.getContext('2d');

// Polyfill roundRect
if (!CanvasRenderingContext2D.prototype.roundRect) {
  CanvasRenderingContext2D.prototype.roundRect = function(x,y,w,h,r){
    r = Math.min(r, Math.min(w,h)/2);
    this.beginPath();
    this.moveTo(x+r,y);
    this.lineTo(x+w-r,y);
    this.arcTo(x+w,y,x+w,y+r,r);
    this.lineTo(x+w,y+h-r);
    this.arcTo(x+w,y+h,x+w-r,y+h,r);
    this.lineTo(x+r,y+h);
    this.arcTo(x,y+h,x,y+h-r,r);
    this.lineTo(x,y+r);
    this.arcTo(x,y,x+r,y,r);
    this.closePath();
    return this;
  };
}

/* ── RESPONSIVE SIZING ── */
let W, H, SC;
function resize(){
  const pw = window.innerWidth, ph = window.innerHeight;
  const maxW = pw <= 900 ? pw : Math.min(pw * 0.45, 480);
  W = maxW; H = ph;
  canvas.width = W; canvas.height = H;
  SC = Math.max(Math.min(W/390, H/844), 0.55);
}
resize();
window.addEventListener('resize', ()=>{ resize(); initPositions(); });

/* ── CONSTANTS ── */
const LANES      = 5;
const PLAYER_YR  = 0.75; // player y as ratio
const DASH_H     = 70;
const DAY_CYCLE  = 1800;  // frames for full day/night cycle

/* ── DYNAMIC LAYOUT ── */
let RL, RR, RW, LW;
function initPositions(){
  RL = W * 0.05;  RR = W * 0.95;
  RW = RR - RL;   LW = RW / LANES;
}
initPositions();

function laneX(l){ return RL + LW * l + LW / 2; }
function lerp(a,b,t){ return a+(b-a)*t; }
function clamp(v,a,b){ return Math.max(a,Math.min(b,v)); }

/* ── STATE ── */
let state = 'menu'; // menu | playing | gameover
let score, hiScore=0, lives, dist, spd, frame=0;
let dayT = 0;    // 0-2PI
let roadOff = 0;
let tunnelT = 0; // 0 = open, 1 = in tunnel
let tunnelTarget = 0;
let nextTunnel = 3000;
let combo = 1, comboTimer = 0;
let screenShake = 0;

/* ── PLAYER ── */
const P = { lane:2, tLane:2, x:0, y:0, animF:0, animT:0, inv:0 };

/* ── VEHICLES ── */
let vehicles = [];
let vTimer   = 0;

/* ── PARTICLES ── */
let parts = [];

/* ── COINS ── */
let coins = [];
let coinTimer = 0;

/* ── HELPERS: DAY/NIGHT ── */
function dayVal(){ return (Math.sin(dayT)+1)/2; } // 0=night 1=day
function isNight(){ return dayVal() < 0.45; }

function skyCol(){
  const d = dayVal();
  if(d > 0.7){
    const t=(d-0.7)/0.3;
    return [`rgb(${lerp(70,120,t)|0},${lerp(130,200,t)|0},${lerp(200,255,t)|0})`,
            `rgb(${lerp(180,255,t)|0},${lerp(150,200,t)|0},${lerp(100,150,t)|0})`];
  } else if(d > 0.3){
    const t=(d-0.3)/0.4;
    return [`rgb(${lerp(5,70,t)|0},${lerp(5,130,t)|0},${lerp(30,200,t)|0})`,
            `rgb(${lerp(30,180,t)|0},${lerp(20,150,t)|0},${lerp(60,100,t)|0})`];
  } else {
    return ['rgb(2,2,18)','rgb(5,5,30)'];
  }
}

/* ── DRAW UTILS ── */
function glow(col, blur=20){
  ctx.shadowColor=col; ctx.shadowBlur=blur;
}
function noGlow(){ ctx.shadowBlur=0; }

/* ── VEHICLE COLORS ── */
const CAR_COLS   = ['#e74c3c','#3498db','#2ecc71','#f39c12','#9b59b6','#1abc9c','#e67e22','#e91e63','#00bcd4'];
const TRUCK_COLS = ['#607d8b','#e74c3c','#263238','#795548'];
const BIKE_COLS  = ['#f44336','#ff9800','#00e5ff','#76ff03','#ea80fc'];
const BUS_COLS   = ['#fdd835','#e53935','#1565c0'];

/* ══════════════════════════════�
