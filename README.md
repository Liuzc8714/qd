<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>HandPose Particle System - AI李探长</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Microsoft YaHei', sans-serif; }
        canvas { display: block; }
        
        /* UI: 左上角状态 */
        #status-board {
            position: absolute; top: 10px; left: 10px;
            color: #0f0; font-family: monospace; font-size: 16px;
            text-shadow: 0 0 5px #0f0; pointer-events: none; z-index: 10;
        }
        
        /* UI: 右侧面板容器 (Lil-GUI 会自动挂载) */
        
        /* UI: 全屏按钮 */
        #fs-btn {
            position: absolute; top: 10px; right: 260px; /* 避开 GUI */
            background: rgba(0, 255, 255, 0.2); border: 1px solid cyan;
            color: cyan; padding: 5px 10px; cursor: pointer; z-index: 20;
            font-size: 12px; transition: 0.3s;
        }
        #fs-btn:hover { background: rgba(0, 255, 255, 0.5); }

        #loading {
            position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
            color: white; font-size: 24px; text-align: center;
        }
    </style>
    <script src="https://unpkg.com/three@0.150.0/build/three.min.js"></script>
    <script src="https://unpkg.com/@tensorflow/tfjs-core@3.21.0/dist/tf-core.min.js"></script>
    <script src="https://unpkg.com/@tensorflow/tfjs-converter@3.21.0/dist/tf-converter.min.js"></script>
    <script src="https://unpkg.com/@tensorflow/tfjs-backend-webgl@3.21.0/dist/tf-backend-webgl.min.js"></script>
    <script src="https://unpkg.com/@tensorflow-models/handpose@0.0.7/dist/handpose.min.js"></script>
    <script src="https://unpkg.com/lil-gui@0.17.0/dist/lil-gui.umd.min.js"></script>
</head>
<body>

<div id="loading">正在初始化 AI 模型...<br><span style="font-size:14px; color:#888;">首次加载可能需要几十秒</span></div>
<div id="status-board">
    <div>手型: <span id="st-gesture">未检测</span></div>
    <div>方向: <span id="st-direction">静止</span></div>
</div>
<button id="fs-btn">全屏模式</button>

<script>
/**
 * 配置与常量
 */
const CONFIG = {
    particleCount: 15000,
    text: "AI李探长",
    font: "bold 100px 'Microsoft YaHei'",
    colors: {
        cold: new THREE.Color(0x00ffff), // 180-240 H range approx
        warm: new THREE.Color(0xffaa00),
    }
};

// 全局状态
const state = {
    gesture: 'None',
    direction: 'None',
    magnitude: 0, // 0-1
    speed: 0,     // px per frame
    isFast: false,
    
    // UI 控制变量
    shapeMode: 'text', // sphere, torus, snow, spiral, text(star_dust initial)
    colorMode: 'speed', // single, hsl, speed
    motionMode: 'trail', // center, explode, trail
    baseColor: '#00ffff',
    
    // 调试值
    fps: 0
};

/**
 * 1. Three.js 初始化
 */
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 1, 3000);
camera.position.z = 1000;

const renderer = new THREE.WebGLRenderer({ antialias: false, powerPreference: "high-performance" });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
document.body.appendChild(renderer.domElement);

// 粒子系统
let particles;
let positions, targetPositions, colors, sizes;
const geometry = new THREE.BufferGeometry();

function initParticles() {
    positions = new Float32Array(CONFIG.particleCount * 3);
    targetPositions = new Float32Array(CONFIG.particleCount * 3); // 目标形状位置
    colors = new Float32Array(CONFIG.particleCount * 3);
    sizes = new Float32Array(CONFIG.particleCount);

    const color = new THREE.Color();
    
    for (let i = 0; i < CONFIG.particleCount; i++) {
        // 初始随机分布
        positions[i*3] = (Math.random() - 0.5) * 2000;
        positions[i*3+1] = (Math.random() - 0.5) * 2000;
        positions[i*3+2] = (Math.random() - 0.5) * 2000;
        
        // 颜色初始化
        color.setHSL(0.6, 1.0, 0.5);
        colors[i*3] = color.r;
        colors[i*3+1] = color.g;
        colors[i*3+2] = color.b;
        
        sizes[i] = 2.0;
    }

    geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
    geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));
    // 大小稍微简化，统一处理，或者使用 ShaderMaterial 支持大小
    
    // 使用点材质
    const material = new THREE.PointsMaterial({
        size: 3,
        vertexColors: true,
        blending: THREE.AdditiveBlending,
        depthWrite: false,
        transparent: true,
        opacity: 0.8
    });

    particles = new THREE.Points(geometry, material);
    scene.add(particles);
}

/**
 * 2. 形状生成逻辑 (文字 & 几何)
 */
function generateTextTargets() {
    const canvas = document.createElement('canvas');
    const ctx = canvas.getContext('2d');
    canvas.width = 1024;
    canvas.height = 512;
    
    ctx.fillStyle = '#000';
    ctx.fillRect(0,0, canvas.width, canvas.height);
    ctx.fillStyle = '#fff';
    ctx.font = CONFIG.font;
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(CONFIG.text, canvas.width/2, canvas.height/2);
    
    const imageData = ctx.getImageData(0,0, canvas.width, canvas.height);
    const data = imageData.data;
    const validPixels = [];
    
    for(let y=0; y<canvas.height; y+=2) {
        for(let x=0; x<canvas.width; x+=2) {
            const index = (y * canvas.width + x) * 4;
            if(data[index] > 128) { // 红色通道有值
                validPixels.push({
                    x: (x - canvas.width/2) * 2,
                    y: -(y - canvas.height/2) * 2,
                    z: 0
                });
            }
        }
    }
    
    // 填充 targetPositions
    for(let i=0; i<CONFIG.particleCount; i++) {
        const p = validPixels[i % validPixels.length];
        targetPositions[i*3] = p.x;
        targetPositions[i*3+1] = p.y;
        targetPositions[i*3+2] = p.z;
    }
}

function updateShapeTargets(type) {
    const time = Date.now() * 0.001;
    for (let i = 0; i < CONFIG.particleCount; i++) {
        let x, y, z;
        const idx = i * 3;
        
        switch (type) {
            case 'sphere':
                const theta = Math.random() * Math.PI * 2;
                const phi = Math.acos((Math.random() * 2) - 1);
                const r = 400;
                x = r * Math.sin(phi) * Math.cos(theta);
                y = r * Math.sin(phi) * Math.sin(theta);
                z = r * Math.cos(phi);
                break;
            case 'torus':
                const u = Math.random() * Math.PI * 2;
                const v = Math.random() * Math.PI * 2;
                const R = 400, r_tube = 100;
                x = (R + r_tube * Math.cos(v)) * Math.cos(u);
                y = (R + r_tube * Math.cos(v)) * Math.sin(u);
                z = r_tube * Math.sin(v);
                break;
            case 'spiral':
                const angle = i * 0.1;
                const rad = i * 0.05;
                x = Math.cos(angle) * rad * 5;
                y = (i - CONFIG.particleCount/2) * 0.1;
                z = Math.sin(angle) * rad * 5;
                break;
            case 'snow': // 六角对称
                // 简化版雪花：在几个轴上分布
                const arm = i % 6;
                const dist = Math.random() * 500;
                const armAngle = (arm / 6) * Math.PI * 2;
                x = Math.cos(armAngle) * dist + (Math.random()-0.5)*20;
                y = Math.sin(armAngle) * dist + (Math.random()-0.5)*20;
                z = (Math.random()-0.5) * 50;
                break;
            case 'text':
                // 文字形状是预计算的，这里只做微小抖动
                // 如果需要动态生成，应该在外面缓存，这里简单略过
                // 假设 text 已经是 targetPositions 的默认值
                // 如果要切回 text，需要重新 copy 一遍
                return; // 这里的逻辑由主循环的 lerp 控制，此函数仅用于计算几何体
            default: // random/stardust
                x = (Math.random() - 0.5) * 1000;
                y = (Math.random() - 0.5) * 1000;
                z = (Math.random() - 0.5) * 1000;
        }
        
        // 仅当非 Text 模式时覆盖
        if (type !== 'text') {
            targetPositions[idx] = x;
            targetPositions[idx+1] = y;
            targetPositions[idx+2] = z;
        }
    }
}

/**
 * 3. AI 手势识别与逻辑
 */
let model = null;
let video = null;
let lastHandPos = { x: 0, y: 0, z: 0, t: 0 };

async function setupCamera() {
    video = document.createElement('video');
    const stream = await navigator.mediaDevices.getUserMedia({ 
        video: { width: 640, height: 480, facingMode: 'user' } 
    });
    video.srcObject = stream;
    return new Promise((resolve) => {
        video.onloadedmetadata = () => { video.play(); resolve(video); };
    });
}

// 几何计算辅助
function getAngle(A, B, C) {
    // 向量 BA 和 BC 的夹角
    const BA = { x: A[0]-B[0], y: A[1]-B[1], z: A[2]-B[2] };
    const BC = { x: C[0]-B[0], y: C[1]-B[1], z: C[2]-B[2] };
    const dot = BA.x*BC.x + BA.y*BC.y + BA.z*BC.z;
    const magBA = Math.sqrt(BA.x**2 + BA.y**2 + BA.z**2);
    const magBC = Math.sqrt(BC.x**2 + BC.y**2 + BC.z**2);
    return Math.acos(dot / (magBA * magBC)) * (180 / Math.PI);
}

function detectGesture(landmarks) {
    // 简单几何规则
    // 0:Wrist, 4:ThumbTip, 8:IndexTip, 12:MidTip, 16:RingTip, 20:PinkyTip
    // 关键点关节: 
    // Thumb: 1-2-3-4
    // Index: 5-6-7-8 ...
    
    const isFingerExtended = (baseIdx, tipIdx) => {
        // 判断指尖是否比第二关节离手腕更远 (简化版)
        // 更准确的是看角度
        const wrist = landmarks[0];
        const tip = landmarks[tipIdx];
        const pip = landmarks[baseIdx + 1]; // Proximal Interphalangeal joint
        
        // 使用距离手腕的距离来判断
        const dTip = Math.hypot(tip[0]-wrist[0], tip[1]-wrist[1]);
        const dPip = Math.hypot(pip[0]-wrist[0], pip[1]-wrist[1]);
        return dTip > dPip * 1.2; // 阈值
    };

    const thumbOpen = isFingerExtended(1, 4); // Thumb logic is tricky, simple heuristic
    const indexOpen = isFingerExtended(5, 8);
    const midOpen = isFingerExtended(9, 12);
    const ringOpen = isFingerExtended(13, 16);
    const pinkyOpen = isFingerExtended(17, 20);
    
    const count = (indexOpen?1:0) + (midOpen?1:0) + (ringOpen?1:0) + (pinkyOpen?1:0);
    
    // 1.1 静态手型判别
    if (count === 0 && !thumbOpen) return 'Fist'; // 握拳
    if (count === 4 && thumbOpen) return 'Palm'; // 手掌
    if (indexOpen && midOpen && !ringOpen && !pinkyOpen) return 'Victory'; // 剪刀手
    if (indexOpen && !midOpen && !ringOpen && !pinkyOpen) return 'Pointing'; // 竖食指
    
    // OK 手势 (拇指食指相触)
    const dThumbIndex = Math.hypot(landmarks[4][0]-landmarks[8][0], landmarks[4][1]-landmarks[8][1]);
    if (dThumbIndex < 30 && midOpen && ringOpen) return 'OK';

    // 鹰爪 (手指弯曲但不是握拳)
    // 检测指尖到掌心的距离 vs 手指长度，或者简单判断：不是握拳且 fingers bent
    // 这里简化：如果是 5 指张开但指尖弯曲 (这里用 count=4 但 geometry check 复杂，做简单映射)
    // 作为一个 demo，如果没有归类到上述，且手指比较聚集，可以认为是 Claw
    if (count > 0 && count < 4 && !indexOpen) return 'Claw'; 

    return 'Unknown';
}

async function startAI() {
    await setupCamera();
    model = await handpose.load();
    document.getElementById('loading').style.display = 'none';
    
    // 文字生成
    generateTextTargets();
    initParticles();
    
    // UI Init
    initUI();
    
    detectLoop();
    animate();
}

function detectLoop() {
    async function frame() {
        if (model && video) {
            const predictions = await model.estimateHands(video);
            if (predictions.length > 0) {
                const hand = predictions[0];
                const landmarks = hand.landmarks;
                
                // 3.1 手型识别
                const g = detectGesture(landmarks);
                if (g !== 'Unknown') {
                    state.gesture = g;
                    applyGestureRules(g);
                }

                // 运动计算
                // Hand center (approximate with palm base 0 and 9)
                const cx = (landmarks[0][0] + landmarks[9][0]) / 2;
                const cy = (landmarks[0][1] + landmarks[9][1]) / 2;
                // Z depth estimation (scale related)
                // 用手掌宽度估算 Z: 0到9的距离
                const palmSize = Math.hypot(landmarks[0][0]-landmarks[9][0], landmarks[0][1]-landmarks[9][1]);
                const cz = 1000 / (palmSize + 1); // rough z
                
                const now = Date.now();
                const dt = (now - lastHandPos.t) || 16;
                
                // 3.2 & 3.4 速度与方向
                const dx = cx - lastHandPos.x;
                const dy = cy - lastHandPos.y;
                const dz = cz - lastHandPos.z; // Push/Pull
                
                const dist = Math.sqrt(dx*dx + dy*dy);
                const velocity = dist / dt; // px/ms
                
                // 阈值判断
                state.speed = velocity * 100; // scale up for UI
                state.isFast = dt > 0 && state.speed > 50; // 调优阈值
                
                // 1.2 方向识别
                if (Math.abs(dz) > 5) { // Z 轴优先
                   state.direction = dz < 0 ? 'Push (前推)' : 'Pull (后拉)';
                } else if (Math.abs(dx) > Math.abs(dy)) {
                   state.direction = dx > 0 ? 'Left (左扫)' : 'Right (右扫)'; // 镜像
                } else {
                   state.direction = dy > 0 ? 'Down (下扫)' : 'Up (上扫)';
                }
                
                // 1.3 幅度 (normalize to 0-1)
                // 简单映射：屏幕宽度的占比
                const screenPercent = Math.min(dist / 300, 1);
                state.magnitude = screenPercent;

                lastHandPos = { x: cx, y: cy, z: cz, t: now };
                
                // 应用联动规则 3.2, 3.3, 3.4
                updatePhysicsParams();
            } else {
                state.gesture = 'None';
                state.direction = 'None';
            }
            
            // UI 更新
            document.getElementById('st-gesture').innerText = state.gesture;
            document.getElementById('st-direction').innerText = state.direction;
        }
        requestAnimationFrame(frame);
    }
    frame();
}

/**
 * 4. 联动规则实现 (The Logic Glue)
 */

function applyGestureRules(gesture) {
    // 3.1 静态手型 -> 形态
    // 避免每帧重置，只有变化时 trigger
    // 但为了平滑过渡，我们在 updateParticles 里 lerp
    let mode = state.shapeMode;
    switch(gesture) {
        case 'Fist': mode = 'sphere'; break;
        case 'Palm': mode = 'torus'; break;
        case 'Victory': mode = 'snow'; break;
        case 'Claw': mode = 'spiral'; break;
        case 'OK': mode = 'text'; break; // OK=星尘/Text
        case 'Pointing': 
             // 随机切换
             const modes = ['sphere','torus','snow','spiral','text'];
             if (Math.random() < 0.05) { // 降低随机频率
                 mode = modes[Math.floor(Math.random()*modes.length)];
             }
             break;
    }
    if (mode !== state.shapeMode) {
        state.shapeMode = mode;
        uiController.shape.setValue(mode); // Sync UI
        // 如果切回 text，需要重置 targetPositions
        if (mode === 'text') generateTextTargets(); 
        else updateShapeTargets(mode);
    }
}

function updatePhysicsParams() {
    // 3.2 运动方向 -> 运动模式
    const dir = state.direction;
    if (dir.includes('Left') || dir.includes('Right')) state.motionMode = 'trail';
    else if (dir.includes('Up') || dir.includes('Down')) state.motionMode = 'explode'; // 向心/离心交替 -> 统称 explode 逻辑处理
    else if (dir.includes('Push') || dir.includes('Pull')) state.motionMode = 'explode';

    // 3.3 幅度 -> 粒子半径 (影响 lerp 速度或扩散范围)
    // 小幅度 0-300px，大幅度 300-800px
    // 我们用一个 factor 控制扩散大小
    const mag = state.magnitude; // 0-1
    // 将 mag 映射到 influence radius
    // 这里我们用全局变量控制 updateParticles 中的行为
    
    // 3.4 速度 -> 颜色
    // 慢速冷色 (240H)，快速暖色 (60H -> 0H)
    // HSL: Blue=240/360=0.66, Yellow/Red=0-60
    if (state.colorMode === 'speed') {
        const speedFactor = Math.min(state.speed / 100, 1); // 0 slow, 1 fast
        // Lerp H value: Slow(0.66) -> Fast(0.1)
        const targetH = 0.66 - (speedFactor * 0.66); 
        const c = new THREE.Color().setHSL(targetH, 1.0, 0.5);
        state.baseColor = '#' + c.getHexString();
        // uiController.color.setValue(state.baseColor); // Update excessive UI will lag
    }
}


/**
 * 5. 粒子渲染循环 (Physics)
 */
function animate() {
    requestAnimationFrame(animate);

    if (!particles) return;

    const positionsAttr = geometry.attributes.position;
    const colorsAttr = geometry.attributes.color;
    const time = Date.now() * 0.001;

    // 性能优化：只在需要时计算颜色
    const baseC = new THREE.Color(state.baseColor);
    const hsl = {}; baseC.getHSL(hsl);

    // 3.3 幅度映射
    const effectRadius = state.magnitude < 0.4 ? 300 : 800;
    
    // 运动模式逻辑
    const isTrail = state.motionMode === 'trail';
    const isExplode = state.motionMode === 'explode'; // 包含向心离心
    const isPulse = state.direction.includes('Up') || state.direction.includes('Down');

    for (let i = 0; i < CONFIG.particleCount; i++) {
        const px = positions[i*3];
        const py = positions[i*3+1];
        const pz = positions[i*3+2];
        
        let tx = targetPositions[i*3];
        let ty = targetPositions[i*3+1];
        let tz = targetPositions[i*3+2];

        // 3.2 运动模式修改目标位置
        if (isExplode) {
            // 前后推：爆炸幅度
            const blast = state.direction.includes('Push') ? 1.5 : 0.5; // Push = Expand
            tx *= blast; ty *= blast; tz *= blast;
            
            if (isPulse) {
                // 上下扫：向心/离心交替 (sin wave)
                const pulse = 1 + Math.sin(time * 5) * 0.5;
                tx *= pulse; ty *= pulse; tz *= pulse;
            }
        }
        
        if (isTrail) {
            // 左右扫：拖尾 (延迟 lerp)
            // 简单实现：目标位置加上根据 ID 的相位延迟
            tx += Math.sin(time * 2 + i*0.01) * 50; 
        }

        // Physics Update: Lerp towards target
        // 速度越快，Lerp 越慢以产生拖尾？或者 Lerp 越快？
        // "快" = 200ms 连续，这里让粒子响应更灵敏
        const lerpFactor = state.isFast ? 0.1 : 0.05;
        
        positions[i*3]   += (tx - px) * lerpFactor;
        positions[i*3+1] += (ty - py) * lerpFactor;
        positions[i*3+2] += (tz - pz) * lerpFactor;

        // 颜色更新
        if (state.colorMode === 'hsl') {
            const h = (time * 0.1 + i * 0.0001) % 1;
            const c = new THREE.Color().setHSL(h, 1.0, 0.5);
            colors[i*3] = c.r; colors[i*3+1] = c.g; colors[i*3+2] = c.b;
        } else {
            // Single or Speed (Speed updates state.baseColor)
            colors[i*3] = baseC.r;
            colors[i*3+1] = baseC.g;
            colors[i*3+2] = baseC.b;
        }
    }

    positionsAttr.needsUpdate = true;
    colorsAttr.needsUpdate = true;

    // 旋转场景一点点
    particles.rotation.y += 0.001;
    
    renderer.render(scene, camera);
}


/**
 * 6. UI 初始化
 */
let uiController = {};
function initUI() {
    const gui = new lil.GUI({ width: 250, title: '控制面板' });
    
    // 5.1 右侧面板
    uiController.shape = gui.add(state, 'shapeMode', ['sphere', 'torus', 'snow', 'spiral', 'text']).name('粒子形态').onChange(val => {
        if(val==='text') generateTextTargets();
        else updateShapeTargets(val);
    });
    
    uiController.colorMode = gui.add(state, 'colorMode', ['single', 'hsl', 'speed']).name('颜色模式');
    
    // 实时数值条 (readonly)
    gui.add(state, 'magnitude', 0, 1).name('幅度').listen().disable();
    gui.add(state, 'speed', 0, 200).name('速度').listen().disable();
    
    // 5.3 全屏按钮
    document.getElementById('fs-btn').addEventListener('click', () => {
        if (!document.fullscreenElement) document.documentElement.requestFullscreen();
        else document.exitFullscreen();
    });
}

// Window Resize
window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});

// Start
startAI();

</script>
</body>
</html>
