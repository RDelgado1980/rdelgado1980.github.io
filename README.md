<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>3D Chaos Lab</title>
    <style>
        body { margin: 0; overflow: hidden; background-color: #000; font-family: 'Segoe UI', sans-serif; }
        canvas { display: block; }
        #ui { position: absolute; top: 20px; left: 20px; color: white; pointer-events: none; text-shadow: 0 0 15px rgba(255, 255, 255, 0.5); }
        h1 { margin: 0; font-size: 1.4rem; letter-spacing: 3px; text-transform: uppercase; font-weight: 300; }
        h2 { margin: 0; font-size: 2.4rem; letter-spacing: 3px; text-transform: uppercase; font-weight: 800; }
        h3 { margin: 0; font-size: 0.8rem; letter-spacing: 3px; font-weight: 200; }
        #seed-info { font-size: 0.8rem; opacity: 0.8; font-family: monospace; margin-top: 5px; color: #00f2ff; }
        #controls { position: absolute; bottom: 30px; left: 20px; display: flex; gap: 15px; align-items: flex-end; }
        #formula-container { position: absolute; bottom: 30px; right: 30px; color: rgba(255,255,255,0.9); background: rgba(0,0,0,0.6); padding: 15px; border-radius: 8px; border-right: 3px solid #00f2ff; font-family: 'Times New Roman', serif; pointer-events: none; backdrop-filter: blur(5px); transition: all 0.5s ease; }
        .formula-line { font-style: italic; font-size: 1.1rem; margin: 5px 0; }
        #selector-container { position: absolute; top: 20px; right: 20px; }
        
        /* Estilos para la autoría */
        #author-tag {
            position: absolute;
            bottom: 10px;
            right: 15px;
            color: rgba(255, 255, 255, 0.4);
            font-size: 0.65rem;
            letter-spacing: 1px;
            pointer-events: none;
            text-transform: uppercase;
        }

        select, button {
            background: rgba(0, 0, 0, 0.8); border: 1px solid #ffffff; color: #ffffff;
            padding: 12px 20px; font-size: 0.85rem; cursor: pointer; text-transform: uppercase;
            transition: all 0.3s ease; border-radius: 4px; outline: none; pointer-events: auto;
        }
        .slider-panel { display: flex; flex-direction: column; color: white; font-size: 0.7rem; gap: 10px; background: rgba(0,0,0,0.7); padding: 15px; border-radius: 4px; pointer-events: auto; border: 1px solid #ffffff; min-width: 180px; max-height: 40vh; overflow-y: auto; }
        .slider-group { display: flex; flex-direction: column; gap: 2px; }
        input[type=range] { cursor: pointer; width: 100%; accent-color: #00f2ff; }
        .theme-rayleigh { border-color: #39ff14 !important; color: #39ff14 !important; }
        .theme-lorenz { border-color: #ff3e00 !important; color: #ff3e00 !important; }
        .theme-aizawa { border-color: #00e5ff !important; color: #00e5ff !important; }
        .theme-roessler { border-color: #ff00ff !important; color: #ff00ff !important; }
        .theme-duffing { border-color: #ffcc00 !important; color: #ffcc00 !important; }
    </style>
</head>
<body>

    <div id="ui">
        <h2 id="title1">3D Chaos Lab</h2>
        <h1 id="title">Atractor de Rayleigh</h1>
        <h3 id="title2">Click Izquierdo = Rotar | Click Derecho = Desplazar | Scroll = Zoom</h3>
        <div id="seed-info">SISTEMA LISTO</div>
    </div>

    <div id="selector-container">
        <select id="attractorSelect" onchange="initUI(); resetSimulation();">
            <option value="lorenz">Lorenz</option>
            <option value="aizawa">Aizawa</option>
            <option value="roessler">Rössler</option>
            <option value="duffing">Duffing</option>
            <option value="thomas">Thomas</option>
            <option value="dadras">Dadras</option>
            <option value="chen">Chen</option>
            <option value="halvorsen">Halvorsen</option>
            <option value="rabinovich">Rabinovich-Fabrikant</option>
            <option value="fourwing">Four-Wing</option>
            <option value="rayleigh">Rayleigh</option>
        </select>
    </div>

    <div id="formula-container"></div>

    <div id="controls">
        <button id="resetBtn" onclick="resetSimulation()">Reiniciar Animación</button>
        <div id="dynamicControls" class="slider-panel"></div>
    </div>

    <div id="author-tag">Concepto elaborado por Ronald Delgado - Powered by Google Gemini</div>

    <script type="importmap">
        { "imports": { "three": "https://unpkg.com/three@0.160.0/build/three.module.js", "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/" } }
    </script>

    <script type="module">
        import * as THREE from 'three';
        import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

        const formulas = {
            lorenz: ["ẋ = σ(y - x)", "ẏ = x(ρ - z) - y", "ż = xy - βz"],
            rayleigh: ["ẋ = y", "ẏ = -x + a·y(1 - y²)", "ż = r·xy - b·z"],
            aizawa: ["ẋ = (z-b)x - dy", "ẏ = dx + (z-b)y", "ż = c + az - z³/3 - (x²+y²)(1+ez) + fzx³"],
            roessler: ["ẋ = -y - z", "ẏ = x + ay", "ż = b + z(x - c)"],
            duffing: ["ẋ = y", "ẏ = x - x³ - δy + γ cos(z)", "ż = ω"],
            thomas: ["ẋ = -bx + sin(y)", "ẏ = -by + sin(z)", "ż = -bz + sin(x)"],
            dadras: ["ẋ = y - ax + byz", "ẏ = cy - xz + z", "ż = dxy - ez"],
            chen: ["ẋ = a(y - x)", "ẏ = (c - a)x - xz + cy", "ż = xy - bz"],
            halvorsen: ["ẋ = -ax - 4y - 4z - y²", "ẏ = -ay - 4z - 4x - z²", "ż = -az - 4x - 4y - x²"],
            rabinovich: ["ẋ = y(z - 1 + x²) + ax", "ẏ = x(3z + 1 - x²) + ay", "ż = -2z(b + xy)"],
            fourwing: ["ẋ = ax + cy + yz", "ẏ = bx + cy - xz", "ż = -z - xy"]
        };

        const config = {
            lorenz: { σ: [0, 50, 10], ρ: [0, 100, 28], β: [0, 10, 8/3], dt: 0.008, scale: 1.2, offsetZ: -25 },
            aizawa: { a: [0, 2, 0.95], b: [0, 2, 0.7], c: [0, 2, 0.6], d: [0, 10, 3.5], e: [0, 1, 0.25], f: [0, 1, 0.1], dt: 0.015, scale: 15, offsetZ: 0 },
            roessler: { a: [0, 1, 0.2], b: [0, 1, 0.2], c: [0, 15, 5.7], dt: 0.04, scale: 1.5, offsetZ: -15 },
            rayleigh: { a: [1, 40, 12], r: [1, 50, 15], b: [0.1, 10, 2], dt: 0.005, scale: 2.0, offsetZ: 0 },
            duffing: { δ: [0, 1, 0.3], γ: [0, 1, 0.37], ω: [0, 3, 1.2], dt: 0.05, scale: 20, offsetZ: 0 },
            thomas: { b: [0, 1, 0.20815], dt: 0.1, scale: 10, offsetZ: 0 },
            dadras: { a: [0, 10, 3], b: [0, 10, 2.7], c: [0, 10, 1.7], d: [0, 10, 2], e: [0, 20, 9], dt: 0.01, scale: 3, offsetZ: 0 },
            chen: { a: [0, 100, 40], b: [0, 10, 3], c: [0, 100, 28], dt: 0.005, scale: 1, offsetZ: -25 },
            halvorsen: { a: [0, 5, 1.89], dt: 0.01, scale: 3, offsetZ: 0 },
            rabinovich: { a: [0, 2, 0.14], b: [0, 2, 0.1], dt: 0.01, scale: 15, offsetZ: 0 },
            fourwing: { a: [0, 2, 0.2], b: [0, 1, 0.01], c: [-1, 1, -0.4], dt: 0.05, scale: 15, offsetZ: 0 }
        };

        let x, y, z;
        let activeParams = {};
        let baseHue = 0;
        let autoRotationAxis = new THREE.Vector3(0, 1, 0);
        let rotationSpeed = 0.002;
        let isUserInteracting = false;
        const maxPoints = 1500000;
        let currentPoint = 0;
        
        const scene = new THREE.Scene();
        const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1500);
        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.body.appendChild(renderer.domElement);

        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;

        const geometry = new THREE.BufferGeometry();
        const positions = new Float32Array(maxPoints * 3);
        const colors = new Float32Array(maxPoints * 3);
        geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
        geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));

        const material = new THREE.LineBasicMaterial({ vertexColors: true, transparent: true, opacity: 0.8, blending: THREE.AdditiveBlending });
        const line = new THREE.Line(geometry, material);
        scene.add(line);

        window.updateDynamicParam = function(key, val) {
            activeParams[key] = parseFloat(val);
            document.getElementById('val-' + key).innerText = activeParams[key].toFixed(2);
        }

        window.initUI = function() {
            const type = document.getElementById('attractorSelect').value;
            const p = config[type];
            const panel = document.getElementById('dynamicControls');
            const formulaPanel = document.getElementById('formula-container');
            
            panel.innerHTML = '';
            activeParams = {};
            Object.keys(p).forEach(key => {
                if (Array.isArray(p[key])) {
                    activeParams[key] = p[key][2];
                    const group = document.createElement('div');
                    group.className = 'slider-group';
                    group.innerHTML = `
                        <label>${key}: <span id="val-${key}">${activeParams[key].toFixed(2)}</span></label>
                        <input type="range" min="${p[key][0]}" max="${p[key][1]}" step="0.01" value="${p[key][2]}" 
                               oninput="updateDynamicParam('${key}', this.value)">
                    `;
                    panel.appendChild(group);
                }
            });
            formulaPanel.innerHTML = formulas[type].map(f => `<div class="formula-line">${f}</div>`).join('');
            document.getElementById('title').innerText = "Atractor de " + type;
            document.getElementById('resetBtn').className = 'theme-' + type;
        }

        window.resetSimulation = function() {
            const type = document.getElementById('attractorSelect').value;
            const p = config[type];
            currentPoint = 0;
            geometry.setDrawRange(0, 0);
            
            baseHue = Math.random();
            x = (Math.random() - 0.5) * 2;
            y = (Math.random() - 0.5) * 2; 
            z = (Math.random() - 0.5) * 2;

            document.getElementById('seed-info').innerText = "EJECUTANDO SIMULACIÓN";
            camera.position.set(60, 60, 100);
            controls.target.set(0, 0, (type === 'lorenz' || type === 'chen' ? 25 : 0));
        }

        function animate() {
            requestAnimationFrame(animate);
            const type = document.getElementById('attractorSelect').value;
            const p = config[type];
            const v = activeParams;

            if (!isUserInteracting) line.rotateOnWorldAxis(autoRotationAxis, rotationSpeed);
            if (currentPoint < maxPoints) {
                for(let i = 0; i < 60; i++) {
                    if (currentPoint >= maxPoints) break;
                    let dx, dy, dz;
                    switch(type) {
                        case 'lorenz':
                            dx = (v.σ * (y - x)) * p.dt;
                            dy = (x * (v.ρ - z) - y) * p.dt;
                            dz = (x * y - v.β * z) * p.dt; break;
                        case 'rayleigh':
                            dx = y * p.dt;
                            dy = (-x + v.a * y * (1 - y**2)) * p.dt;
                            dz = (v.r * x * y - v.b * z + 0.1 * Math.sin(currentPoint * 0.01)) * p.dt; break;
                        case 'aizawa':
                            dx = ((z - v.b) * x - v.d * y) * p.dt;
                            dy = (v.d * x + (z - v.b) * y) * p.dt;
                            dz = (v.c + v.a * z - (z**3/3) - (x**2 + y**2) * (1 + v.e * z) + v.f * z * x**3) * p.dt;
                            break;
                        case 'roessler':
                            dx = (-y - z) * p.dt;
                            dy = (x + v.a * y) * p.dt; dz = (v.b + z * (x - v.c)) * p.dt;
                            break;
                        case 'duffing':
                            dx = y * p.dt;
                            dy = (x - x**3 - v.δ * y + v.γ * Math.cos(z)) * p.dt; dz = v.ω * p.dt;
                            break;
                        case 'thomas':
                            dx = (-v.b * x + Math.sin(y)) * p.dt;
                            dy = (-v.b * y + Math.sin(z)) * p.dt; dz = (-v.b * z + Math.sin(x)) * p.dt; break;
                        case 'dadras':
                            dx = (y - v.a * x + v.b * y * z) * p.dt;
                            dy = (v.c * y - x * z + z) * p.dt;
                            dz = (v.d * x * y - v.e * z) * p.dt; break;
                        case 'chen':
                            dx = (v.a * (y - x)) * p.dt;
                            dy = ((v.c - v.a) * x - x * z + v.c * y) * p.dt;
                            dz = (x * y - v.b * z) * p.dt; break;
                        case 'halvorsen':
                            dx = (-v.a * x - 4*y - 4*z - y**2) * p.dt;
                            dy = (-v.a * y - 4*z - 4*x - z**2) * p.dt;
                            dz = (-v.a * z - 4*x - 4*y - x**2) * p.dt; break;
                        case 'rabinovich':
                            dx = (y * (z - 1 + x**2) + v.a * x) * p.dt;
                            dy = (x * (3*z + 1 - x**2) + v.a * y) * p.dt;
                            dz = (-2*z * (v.b + x*y)) * p.dt; break;
                        case 'fourwing':
                            dx = (v.a * x + v.c * y + y * z) * p.dt;
                            dy = (v.b * x + v.c * y - x * z) * p.dt;
                            dz = (-z - x * y) * p.dt; break;
                    }
                    x += dx; y += dy; z += dz;
                    const idx = currentPoint * 3;
                    positions[idx] = x * p.scale;
                    positions[idx+1] = y * p.scale;
                    positions[idx+2] = (type === 'duffing' ? 0 : (z + (p.offsetZ || 0)) * p.scale);
                    
                    const progress = currentPoint / maxPoints;
                    const colorHue = (baseHue + Math.sin(progress * Math.PI * 4) * 0.5 + 0.5) % 1;
                    const color = new THREE.Color().setHSL(colorHue, 1.0, 0.5);
                    
                    colors[idx] = color.r; colors[idx+1] = color.g; colors[idx+2] = color.b;
                    currentPoint++;
                }
                geometry.attributes.position.needsUpdate = true;
                geometry.attributes.color.needsUpdate = true;
            }
            geometry.setDrawRange(0, currentPoint);
            controls.update();
            renderer.render(scene, camera);
        }

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight; camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });
        initUI();
        resetSimulation();
        animate();
    </script>
</body>
</html>
