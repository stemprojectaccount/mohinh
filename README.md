<!DOCTYPE html>
<html lang="vi">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">
<title>Ngôi nhà một tầng 3D</title>
<style>
*{box-sizing:border-box;margin:0;padding:0;-webkit-tap-highlight-color:transparent}
html,body{height:100%;overflow:hidden;font-family:'Segoe UI',system-ui,-apple-system,sans-serif;user-select:none}
#stage{position:fixed;inset:0;background:linear-gradient(180deg,#5fb0ec 0%,#b9e0f8 55%,#f3faff 100%);touch-action:none}
#title{position:fixed;top:max(14px,env(safe-area-inset-top));left:20px;color:#0f2744;line-height:1.25;pointer-events:none}
#title b{display:block;font-size:20px;font-weight:800;letter-spacing:.3px}
#title span{font-size:13px;font-weight:600;opacity:.75}
#bar{position:fixed;bottom:max(16px,env(safe-area-inset-bottom));left:50%;transform:translateX(-50%);display:flex;flex-wrap:wrap;gap:8px;justify-content:center;width:calc(100% - 24px);max-width:760px}
button{border:0;border-radius:999px;padding:11px 18px;font:700 14px 'Segoe UI',system-ui,sans-serif;color:#0f2744;background:rgba(255,255,255,.92);box-shadow:0 3px 12px rgba(15,39,68,.22);cursor:pointer;transition:transform .12s,background .2s}
button:hover{transform:translateY(-1px)}
button.on{background:#0f2744;color:#fff}
button.main{background:#f97316;color:#fff}
button.main.on{background:#c2410c}
.dim{background:#fff;color:#0f2744;border:1.5px solid #1d4ed8;font:700 13px 'Segoe UI',system-ui,sans-serif;padding:1px 8px;border-radius:10px;box-shadow:0 2px 6px rgba(0,0,0,.2);white-space:nowrap;pointer-events:none}
#toast{position:fixed;top:24px;left:50%;transform:translateX(-50%);background:rgba(15,39,68,.85);color:#fff;font:600 14px 'Segoe UI',sans-serif;padding:9px 18px;border-radius:999px;opacity:0;transition:opacity .5s;pointer-events:none}
#toast.show{opacity:1}
body.rec #title,body.rec #bar{display:none}
</style>
<script type="importmap">{"imports":{"three":"https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js","three/addons/":"https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/"}}</script>
</head>
<body>
<div id="stage"></div>
<div id="title"><b>Ngôi nhà một tầng</b><span>8000 × 6000 × 4500 mm</span></div>
<div id="bar">
  <button id="b-spin" class="main">▶ Xoay 360°</button>
  <button id="b-speed">⚡ Tốc độ: Vừa</button>
  <button id="b-roof">🏠 Mở mái</button>
  <button id="b-dim">📏 Kích thước</button>
  <button id="b-reset">⟲ Góc nhìn đầu</button>
  <button id="b-rec">🎬 Chế độ quay</button>
</div>
<div id="toast">Nhấn đúp hoặc Esc để thoát chế độ quay</div>
<script type="module">
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import { CSS2DRenderer, CSS2DObject } from 'three/addons/renderers/CSS2DRenderer.js';

const $ = id => document.getElementById(id), container = $('stage');
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(38, container.clientWidth / container.clientHeight, 1000, 90000);
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, powerPreference: 'high-performance' });
renderer.setSize(container.clientWidth, container.clientHeight);
renderer.setPixelRatio(Math.min(Math.max(devicePixelRatio, 2), 3));   // tối thiểu 2x để đường nét mảnh vẫn sắc
renderer.shadowMap.enabled = true; renderer.shadowMap.type = THREE.PCFSoftShadowMap;
container.appendChild(renderer.domElement);
const labelRenderer = new CSS2DRenderer();
labelRenderer.setSize(container.clientWidth, container.clientHeight);
Object.assign(labelRenderer.domElement.style, { position: 'absolute', top: '0', pointerEvents: 'none' });
container.appendChild(labelRenderer.domElement);

// Điều khiển dễ xoay: kéo = xoay, cuộn/chụm = phóng to, không cho chui xuống dưới đất
const controls = new OrbitControls(camera, renderer.domElement);
Object.assign(controls, { enableDamping: true, dampingFactor: 0.07, rotateSpeed: 0.8, zoomSpeed: 0.8, minDistance: 6000, maxDistance: 32000, maxPolarAngle: Math.PI * 0.485, autoRotateSpeed: 3 });
function resetView() { camera.position.set(-9500, 6200, 13500); controls.target.set(0, 1700, 0); controls.update(); }
resetView();

// Ánh sáng sáng, bóng đổ mềm
scene.add(new THREE.HemisphereLight(0xffffff, 0xa8c48a, 0.95));
const sun = new THREE.DirectionalLight(0xfff3dc, 1.55);
sun.position.set(-8000, 14000, 11000); sun.castShadow = true;
const SM = matchMedia('(pointer:fine)').matches ? Math.min(8192, renderer.capabilities.maxTextureSize) : 4096;
sun.shadow.mapSize.set(SM, SM); sun.shadow.bias = -0.0004; sun.shadow.normalBias = 25;
Object.assign(sun.shadow.camera, { left: -9500, right: 9500, top: 9500, bottom: -9500, near: 1000, far: 42000 });
scene.add(sun);

// Tâm nhà (4000, 3000) về gốc (0, 0)
const houseRoot = new THREE.Group();
houseRoot.position.set(-4000, 0, 3000);
scene.add(houseRoot);
const collections = { ROOF: new THREE.Group(), WALLS: new THREE.Group(), DOORS: new THREE.Group(), WINDOWS: new THREE.Group(), FURNITURE: new THREE.Group(), FLOOR: new THREE.Group(), DIMENSIONS: new THREE.Group() };
Object.values(collections).forEach(g => houseRoot.add(g));

const std = (color, roughness = 0.6, extra = {}) => new THREE.MeshStandardMaterial({ color, roughness, ...extra });
const mats = {
    ROOF_RED: std(0xb3261e, 0.6), ROOF_DARK: std(0x5b1d12, 0.55),
    WALL_CREAM: std(0xfff7e6, 0.7), WALL_PLINTH: std(0x78716c, 0.8),
    WOOD_FRAME: std(0x7a4a24, 0.45), WOOD_SASH: std(0xa8672f, 0.4), WOOD_LIGHT: std(0xc98a4b, 0.45),
    ALU_DARK: std(0x1e293b, 0.25, { metalness: 0.75 }),
    GLASS: new THREE.MeshPhysicalMaterial({ color: 0x9bd6f5, transparent: true, opacity: 0.5, roughness: 0.05 }),
    BRASS: std(0xd4a017, 0.2, { metalness: 0.9 }),
    FLOOR_CONC: std(0xd9dde3, 0.7), FLOOR_WOOD: std(0xb9824a, 0.45),
    BUSH_GREEN: std(0x2f9e44, 0.7), BED_WHITE: std(0xf8fafc, 0.7),
    DIM_LINE: new THREE.LineBasicMaterial({ color: 0x1d4ed8 })
};
const interactiveMeshes = [];
const edgeMat = new THREE.LineBasicMaterial({ color: 0xa3acba });   // viền siêu mảnh, rất nhạt
const SHOW_EDGES = false;   // false = không vẽ viền nào; đặt true nếu muốn bật lại
const edges = geo => SHOW_EDGES ? new THREE.LineSegments(new THREE.EdgesGeometry(geo), edgeMat) : new THREE.Object3D();
function addBox(name, category, matDesc, minPt, maxPt, mat, group, withOutline = true) {
    const [x0, y0, z0] = minPt, [x1, y1, z1] = maxPt, w = Math.abs(x1 - x0), d = Math.abs(y1 - y0), h = Math.abs(z1 - z0);
    if (w < 1 || d < 1 || h < 1) return null;
    const geo = new THREE.BoxGeometry(w, h, d), mesh = new THREE.Mesh(geo, mat);
    mesh.position.set((x0 + x1) / 2, (z0 + z1) / 2, -(y0 + y1) / 2);
    mesh.castShadow = mat !== mats.GLASS; mesh.receiveShadow = true;
    mesh.userData = { name, category, materialName: matDesc };
    if (withOutline) mesh.add(edges(geo, category === 'FURNITURE'));
    group.add(mesh); interactiveMeshes.push(mesh); return mesh;
}

        // =========================================================================
        // MÔ HÌNH DỰNG THEO BẢN VẼ — mặt bằng 8000×6000, tường cao 3000, mái +1500 (đỉnh 4500)
        // Quy ước: X = ngang nhà, Y = chiều sâu (Y=0 là mặt tiền), Z = cao từ mặt đất.
        // Sàn nhà cao 450 so với đất (3 bậc tam cấp × 150), móng trụ sâu 450 như mặt cắt 1-1.
        // =========================================================================
        const FL = 450, WH = 3000;
        Object.assign(mats, {
            GROUND: new THREE.MeshStandardMaterial({ color: 0x46543a, roughness: 0.95 }),
            TILE: new THREE.MeshStandardMaterial({ color: 0xd5dde5, roughness: 0.3 }),
            PLINTH: new THREE.MeshStandardMaterial({ color: 0xb8b2a6, roughness: 0.85 }),
            DARK: new THREE.MeshStandardMaterial({ color: 0x1f2937, roughness: 0.5 }),
            FABRIC: new THREE.MeshStandardMaterial({ color: 0x6b7f99, roughness: 0.9 }),
            FABRIC2: new THREE.MeshStandardMaterial({ color: 0xa8553a, roughness: 0.95 }),
            WHITE: new THREE.MeshStandardMaterial({ color: 0xf1f5f9, roughness: 0.3 }),
            STEEL: new THREE.MeshStandardMaterial({ color: 0xcbd5e1, metalness: 0.8, roughness: 0.25 })
        });
        mats.FLOOR_SMOOTH = new THREE.MeshStandardMaterial({ color: mats.WALL_CREAM.color, roughness: 1 });
[mats.PLINTH, mats.FLOOR_CONC].forEach(m => { m.color.copy(mats.WALL_CREAM.color); m.roughness = 1; });   // bệ móng, đế cột, tam cấp, nền: cùng màu sơn với tường
// Đẩy mặt phẳng lùi nhẹ ra sau để nét viền luôn nằm trên mặt, không đứt/răng cưa
Object.values(mats).forEach(m => { if (m.isMeshStandardMaterial) { m.polygonOffset = true; m.polygonOffsetFactor = 1; m.polygonOffsetUnits = 1; } });
[mats.WALL_CREAM, mats.PLINTH, mats.WALL_PLINTH].forEach(m => m.roughness = 1);   // tường lì, không loang sáng
Object.assign(mats.GLASS, { opacity: 0.3, depthWrite: false }); mats.GLASS.color.set(0xa8dcf7);
const mk = (cat, grp) => (n, d, a, b, m = mats.WALL_CREAM, o = true) => addBox(n, cat, d, a, b, m, grp, o);
        const W = mk('WALLS', collections.WALLS), D = mk('DOORS', collections.DOORS), N = mk('WINDOWS', collections.WINDOWS),
              R = mk('ROOF', collections.ROOF), L = mk('FLOOR', collections.FLOOR);
        const F = (n, d, a, b, m, o) => addBox(n, 'FURNITURE', d, [a[0], a[1], a[2] + FL], [b[0], b[1], b[2] + FL], m, collections.FURNITURE, o !== false);
        // dựng hộp theo trục tường: ax='x' (tường chạy dọc X, t = Y) hoặc 'y' (tường chạy dọc Y, t = X)
        const seg = (fn, n, d, ax, a0, a1, t0, t1, z0, z1, m, o) =>
            ax === 'x' ? fn(n, d, [a0, t0, z0], [a1, t1, z1], m, o) : fn(n, d, [t0, a0, z0], [t1, a1, z1], m, o);

        // ---------- 1. Cửa sổ & cửa đi (một nguồn dữ liệu: vừa khoét tường vừa dựng khung) ----------
        // [tên, trục, c(tâm tường), vị trí dọc tường, rộng, cao độ ngưỡng, cao, hướng ra ngoài, kiểu f=4 cánh + ô thoáng | s=2 cánh]
        const WINS = [
            ['Cửa sổ 4 cánh mặt tiền (Phòng khách)', 'x', 100, 1600, 1600, 1350, 1250, -1, 'f'],
            ['Cửa sổ 4 cánh mặt tiền (Phòng bếp)', 'x', 100, 6400, 1600, 1350, 1250, -1, 'f'],
            ['Cửa sổ hậu Phòng ngủ 1', 'x', 5900, 1600, 1100, 1100, 1100, 1, 's'],
            ['Cửa sổ hậu Nhà vệ sinh', 'x', 5900, 4000, 800, 1650, 600, 1, 's'],
            ['Cửa sổ hậu Phòng ngủ 2', 'x', 5900, 6400, 1100, 1100, 1100, 1, 's'],
            ['Cửa sổ hông trái Phòng khách', 'y', 100, 1550, 1100, 1100, 1100, -1, 's'],
            ['Cửa sổ hông trái Phòng ngủ 1', 'y', 100, 4500, 1100, 1100, 1100, -1, 's'],
            ['Cửa sổ hông phải Phòng bếp', 'y', 7900, 1500, 1000, 1100, 1100, 1, 's'],
            ['Cửa sổ hông phải Phòng ngủ 2', 'y', 7900, 4500, 1100, 1100, 1100, 1, 's']
        ];
        // [tên, trục, c, vị trí, rộng thông thủy, cao độ đỉnh khuôn, loại, nửa dày khuôn]
        const DOORS = [
            ['Cửa đi chính 2 cánh (D1)', 'x', 100, 4000, 1200, 2600, 'double', 112],
            ['Cửa phụ phòng bếp (D2)', 'y', 7900, 2500, 900, 2550, 'single', 112],
            ['Cửa Phòng ngủ 1', 'y', 3000, 3750, 800, 2550, 'single', 72],
            ['Cửa Phòng ngủ 2', 'y', 5000, 3750, 800, 2550, 'single', 72],
            ['Cửa Phòng bếp', 'y', 5000, 2400, 800, 2550, 'single', 72],
            ['Lối vào Phòng khách (không cánh)', 'y', 3000, 2400, 900, 2550, 'none', 72],
            ['Cửa Nhà vệ sinh', 'x', 4500, 4000, 800, 2450, 'single', 72]
        ];
        const OPS = {};
        const addOp = (ax, c, o) => { (OPS[ax + c] = OPS[ax + c] || []).push(o); };
        WINS.forEach(([n, ax, c, p, w, s, h]) => addOp(ax, c, [p, w, s, s + h]));
        DOORS.forEach(([n, ax, c, p, w, top]) => addOp(ax, c, [p, w, FL, top]));

        // ---------- 2. Tường (bao 200mm, ngăn 120mm) có khoét lỗ cửa ----------
        function wall(name, ax, c, a0, a1, th, ops) {
            ops = (ops || []).slice().sort((p, q) => p[0] - q[0]);
            const sh = new THREE.Shape(); sh.moveTo(a0, FL);
            ops.filter(o => o[2] <= FL).forEach(([oc, ow, , zt]) => { sh.lineTo(oc - ow / 2, FL); sh.lineTo(oc - ow / 2, zt); sh.lineTo(oc + ow / 2, zt); sh.lineTo(oc + ow / 2, FL); });   // cửa đi: khuyết xuống sàn
            sh.lineTo(a1, FL); sh.lineTo(a1, WH); sh.lineTo(a0, WH); sh.closePath();
            ops.filter(o => o[2] > FL).forEach(([oc, ow, zb, zt]) => { const h = new THREE.Path(); h.moveTo(oc - ow / 2, zb); h.lineTo(oc - ow / 2, zt); h.lineTo(oc + ow / 2, zt); h.lineTo(oc + ow / 2, zb); h.closePath(); sh.holes.push(h); });   // cửa sổ: lỗ
            const geo = new THREE.ExtrudeGeometry(sh, { depth: th, bevelEnabled: false }), m = new THREE.Mesh(geo, mats.WALL_CREAM);
            if (ax === 'x') m.position.z = -(c + th / 2); else { m.rotation.y = Math.PI / 2; m.position.x = c - th / 2; }
            m.castShadow = m.receiveShadow = true; m.add(edges(geo));
            m.userData = { name, category: 'WALLS', materialName: `Tường xây dày ${th}mm` };
            collections.WALLS.add(m); interactiveMeshes.push(m);
        }
        wall('Tường mặt tiền', 'x', 100, 0, 8000, 200, OPS.x100);
        wall('Tường mặt hậu', 'x', 5900, 0, 8000, 200, OPS.x5900);
        wall('Tường hông trái', 'y', 100, 0, 6000, 200, OPS.y100);
        wall('Tường hông phải', 'y', 7900, 0, 6000, 200, OPS.y7900);
        wall('Vách ngăn trục X=3000', 'y', 3000, 200, 5800, 120, OPS.y3000);
        wall('Vách ngăn trục X=5000', 'y', 5000, 200, 5800, 120, OPS.y5000);
        wall('Vách ngăn Phòng khách / Ngủ 1', 'x', 3000, 200, 2940, 120);
        wall('Vách ngăn Phòng bếp / Ngủ 2', 'x', 3000, 5060, 7800, 120);
        wall('Vách Nhà vệ sinh', 'x', 4500, 3060, 4940, 120, OPS.x4500);

        function win(name, ax, c, oc, ow, sill, wh, out, kind) {
            const a0 = oc - ow / 2, a1 = oc + ow / 2, z0 = sill, z1 = sill + wh, ft = 70, ia0 = a0 + ft, ia1 = a1 - ft, zb = z0 + ft, zt = z1 - ft;
            const P = (n, d, A0, A1, T0, T1, Z0, Z1, m, o) => seg(N, `${name} (${n})`, d, ax, A0, A1, c + T0, c + T1, Z0, Z1, m, o);
            P('khung dưới', 'Khung bao gỗ 70mm', a0, a1, -110, 110, z0, zb, mats.WOOD_FRAME);
            P('khung trên', 'Khung bao gỗ 70mm', a0, a1, -110, 110, zt, z1, mats.WOOD_FRAME);
            P('khung trái', 'Khung bao gỗ 70mm', a0, ia0, -110, 110, zb, zt, mats.WOOD_FRAME);
            P('khung phải', 'Khung bao gỗ 70mm', ia1, a1, -110, 110, zb, zt, mats.WOOD_FRAME);
            P('bệ cửa', 'Bệ cửa nhô ra ngoài', a0 - 70, a1 + 70, out > 0 ? -90 : -190, out > 0 ? 190 : 90, z0 - 50, z0, mats.WOOD_FRAME);
            P('kính', 'Kính trong 8mm', ia0, ia1, -8, 8, zb, zt, mats.GLASS, true);
            if (kind === 'f') {
                const zh = zt - 300;   // đố ngang tạo hàng ô thoáng phía trên như bản vẽ
                P('đố ngang ô thoáng', 'Đố gỗ', ia0, ia1, -50, 50, zh - 18, zh + 18, mats.WOOD_FRAME);
                for (let i = 1; i < 4; i++) { const x = ia0 + i * (ia1 - ia0) / 4; P(`đố đứng ${i}`, 'Đố gỗ', x - 18, x + 18, -50, 50, zb, zt, mats.WOOD_FRAME); }
            } else P('đố giữa', 'Đố cánh', oc - 25, oc + 25, -50, 50, zb, zt, mats.WOOD_FRAME);
        }
        function door(name, ax, c, oc, ow, top, leaf, ht) {
            const a0 = oc - ow / 2, a1 = oc + ow / 2, ft = 70, z0 = FL, zt = top - ft, ia0 = a0 + ft, ia1 = a1 - ft;
            const P = (n, d, A0, A1, T0, T1, Z0, Z1, m) => seg(D, `${name} (${n})`, d, ax, A0, A1, c + T0, c + T1, Z0, Z1, m);
            P('khuôn trái', 'Khuôn cửa gỗ 70mm', a0, ia0, -ht, ht, z0, top, mats.WOOD_FRAME);
            P('khuôn phải', 'Khuôn cửa gỗ 70mm', ia1, a1, -ht, ht, z0, top, mats.WOOD_FRAME);
            P('khuôn trên', 'Khuôn cửa gỗ 70mm', ia0, ia1, -ht, ht, zt, top, mats.WOOD_FRAME);
            if (leaf === 'none') return;
            if (leaf === 'double') {   // 2 cánh, mỗi cánh 1 pano dài như mặt đứng
                [[ia0, oc - 3], [oc + 3, ia1]].forEach(([s, e], i) => {
                    P(`cánh ${i + 1}`, 'Cánh gỗ dày 45mm', s, e, -22, 22, z0, zt, mats.WOOD_SASH);
                    P(`pano nổi ${i + 1}`, 'Pano gỗ nổi dài', s + 90, e - 90, -34, 34, z0 + 170, zt - 170, mats.WOOD_LIGHT);
                });
                [-40, 22].forEach(o => P('tay nắm', 'Tay nắm đồng', oc + o, oc + o + 18, -55, 55, z0 + 950, z0 + 1300, mats.BRASS));
            } else {
                P('cánh', 'Cánh gỗ dày 45mm', ia0, ia1, -22, 22, z0, zt, mats.WOOD_SASH);
                P('pano dưới', 'Pano gỗ nổi', ia0 + 80, ia1 - 80, -32, 32, z0 + 150, z0 + 900, mats.WOOD_LIGHT);
                P('pano trên', 'Pano gỗ nổi', ia0 + 80, ia1 - 80, -32, 32, z0 + 1050, zt - 150, mats.WOOD_LIGHT);
                P('tay nắm', 'Tay gạt đồng', ia1 - 130, ia1 - 70, -50, 50, z0 + 950, z0 + 1050, mats.BRASS);
            }
        }
        WINS.forEach(w => win(...w));
        DOORS.forEach(d => door(...d));

        // ---------- 3. Cột, đà viền, hiên, lan can, móng, sàn, tam cấp ----------
        [[-0.4, -0.4], [7700.4, -0.4], [-0.4, 5700.4], [7700.4, 5700.4]].forEach(([x, y], i) => {
            W(`Đế cột góc ${i + 1}`, 'Đế cột nhô dưới 1mm', [x - 0.4, y - 0.4, FL], [x + 300.4, y + 300.4, FL + 200], mats.PLINTH);
            W(`Cột góc ${i + 1}`, 'Cột 300×300', [x, y, FL + 200], [x + 300, y + 300, 2800], mats.WALL_CREAM);
        });
        function ringBox(name, cat, desc, x0, y0, x1, y1, ix0, iy0, ix1, iy1, z0, z1, mat, grp, line = true) {
            const sh = new THREE.Shape(); sh.moveTo(x0, y0); sh.lineTo(x1, y0); sh.lineTo(x1, y1); sh.lineTo(x0, y1); sh.closePath();
            const hl = new THREE.Path(); hl.moveTo(ix0, iy0); hl.lineTo(ix0, iy1); hl.lineTo(ix1, iy1); hl.lineTo(ix1, iy0); hl.closePath(); sh.holes.push(hl);
            const geo = new THREE.ExtrudeGeometry(sh, { depth: z1 - z0, bevelEnabled: false }), m = new THREE.Mesh(geo, mat);
            m.rotation.x = -Math.PI / 2; m.position.y = z0; m.castShadow = m.receiveShadow = true; if (line) m.add(edges(geo));
            m.userData = { name, category: cat, materialName: desc };
            grp.add(m); interactiveMeshes.push(m);
        }
        ringBox('Đà viền đỉnh tường', 'WALLS', 'Đà trát vữa, nhô 0.5mm', -0.5, -0.5, 8000.5, 6000.5, 200.5, 200.5, 7799.5, 5799.5, 2800, WH + 0.5, mats.WALL_CREAM, collections.WALLS);   // đà viền trát xi măng sơn cùng màu tường
        ringBox('Bệ móng', 'FLOOR', 'Bệ móng cao 450mm', -0.5, -0.5, 8000.5, 6000.5, 200, 200, 7800, 5800, 0, FL, mats.PLINTH, collections.FLOOR, false);
        [3000, 5000].forEach((x, i) => {   // 2 cột hiên trên trục 3000 & 5000 (nhô ra trước, theo mặt bằng)
            W(`Đế cột hiên ${i + 1}`, 'Đế cột 400×400', [x - 200, -530, FL - 150], [x + 200, -130, FL + 50], mats.PLINTH);
            W(`Cột hiên ${i + 1}`, 'Cột 300×300', [x - 150, -480, FL + 50], [x + 150, -180, 2800], mats.WALL_CREAM);
            W(`Dầm hiên nối tường ${i + 1}`, 'Dầm gỗ 180', [x - 90, -470, 2700], [x + 90, 0, WH], mats.WOOD_FRAME);
        });
        W('Dầm hiên cửa chính', 'Dầm gỗ 300×180', [2840, -490, 2640], [5160, -170, WH + 10], mats.WOOD_FRAME);
        [[270, 2850], [5150, 7730]].forEach(([x0, x1], k) => {   // lan can con tiện dưới cửa sổ mặt tiền
            W(`Tay vịn lan can ${k + 1}`, 'Gỗ tự nhiên', [x0, -90, 1090], [x1, 0, 1150], mats.WOOD_FRAME);
            W(`Thanh đáy lan can ${k + 1}`, 'Gỗ tự nhiên', [x0, -90, 500], [x1, 0, 560], mats.WOOD_FRAME);
            for (let x = x0 + 60; x < x1 - 30; x += 100) W('Con tiện lan can', 'Con tiện gỗ 50mm', [x - 25, -60, 560], [x + 25, -10, 1090], mats.WOOD_SASH, false);
        });
        [100, 3000, 5000, 7900].forEach((x, i) => L(`Trụ móng trục ${i + 1}`, 'Móng bê tông sâu 450mm', [x - 200, -100, -450], [x + 200, 6100, -10], mats.PLINTH));
        L('Sàn bê tông', 'Sàn BTCT dày 140mm', [0, 0, FL - 140], [8000, 6000, FL - 10], mats.FLOOR_CONC, false);
        [[0, 0, 3000, 3000, 'Phòng khách', 1], [0, 3000, 3000, 6000, 'Phòng ngủ 1', 1], [5000, 3000, 8000, 6000, 'Phòng ngủ 2', 1],
         [3000, 0, 5000, 4500, 'Hành lang', 0], [3000, 4500, 5000, 6000, 'Nhà vệ sinh', 0], [5000, 0, 8000, 3000, 'Phòng bếp + ăn', 0]]
            .forEach(([x0, y0, x1, y1, n, wood]) => L(`Lát sàn ${n}`, wood ? 'Sàn gỗ' : 'Gạch men', [Math.max(x0, 200), Math.max(y0, 200), FL - 10], [Math.min(x1, 7800), Math.min(y1, 5800), FL], mats.FLOOR_SMOOTH, false));   // một vật liệu, không viền: các phòng liền thành một mặt nền
        L('Tam cấp - sảnh', 'Bê tông ốp đá', [2800, -250, 0], [5200, -0.5, FL], mats.FLOOR_CONC);
        L('Tam cấp - bậc 2', 'Bê tông ốp đá', [2750, -480, 0], [5250, -250, 300], mats.FLOOR_CONC);
        L('Tam cấp - bậc 1', 'Bê tông ốp đá', [2700, -710, 0], [5300, -480, 150], mats.FLOOR_CONC);

        // ---------- 4. Mái dốc 2 mái, đầu hồi, vì kèo king-post (đỉnh mái = 4500) ----------
        const ZE = 2940, ZR = 4380, SL = (ZR - ZE) / 4450;      // tim mái: từ (-450, ZE) lên (4000, ZR)
        const TH = Math.atan(SL), RL = Math.hypot(4450, ZR - ZE) + 60, YA = -800, YB = 6400, DEP = YB - YA, YC = (YA + YB) / 2;
        const under = X => ZE + SL * (Math.min(X, 8000 - X) + 450) - 76;   // mặt dưới mái tại X
        function rbox(name, cat, desc, w, h, d, pos, rz, mat, grp) {   // hộp xoay quanh trục Y (pos = [x, z, y])
            const geo = new THREE.BoxGeometry(w, h, d), m = new THREE.Mesh(geo, mat);
            m.position.set(pos[0], pos[1], -pos[2]); m.rotation.z = rz;
            m.castShadow = m.receiveShadow = true;
            m.userData = { name, category: cat, materialName: desc, dims: `${Math.round(w)} × ${Math.round(d)} × ${Math.round(h)} mm`, pos: `(${Math.round(pos[0])}, ${Math.round(pos[2])}, ${Math.round(pos[1])})` };
            m.add(edges(geo));
            grp.add(m); interactiveMeshes.push(m); return m;
        }
        [-1, 1].forEach(s => {
            const m = rbox(s < 0 ? 'Mái ngói bên trái' : 'Mái ngói bên phải', 'ROOF', 'Mái ngói đỏ, vươn hiên 450mm', RL, 140, DEP, [s < 0 ? 1775 : 6225, (ZE + ZR) / 2, YC], s < 0 ? TH : -TH, mats.ROOF_RED, collections.ROOF);
            for (let i = 0; i < 17; i++) {   // hàng ngói / thanh mè
                const b = new THREE.Mesh(new THREE.BoxGeometry(26, 14, DEP - 60), mats.ROOF_DARK);
                b.position.set(-RL / 2 + 160 + i * (RL - 260) / 16, 76, 0); m.add(b);
            }
            [-1, 1].forEach(g => { const f = new THREE.Mesh(new THREE.BoxGeometry(RL, 200, 40), mats.WOOD_FRAME); f.position.set(0, -20, g * (DEP / 2 - 10)); m.add(f); });   // diềm đầu hồi
            const e = new THREE.Mesh(new THREE.BoxGeometry(40, 200, DEP + 20), mats.WOOD_FRAME); e.position.set(s < 0 ? -(RL / 2 - 10) : RL / 2 - 10, -20, 0); m.add(e);       // diềm mép mái
        });
        R('Ngói úp nóc', 'Ngói úp nóc', [3800, YA - 20, ZR + 10], [4200, YB + 20, ZR + 120], mats.ROOF_DARK);
        function gable(zPos, name) {   // tam giác đầu hồi xây kín dưới mái
            const sh = new THREE.Shape();
            sh.moveTo(0, WH); sh.lineTo(8000, WH); sh.lineTo(8000, under(8000)); sh.lineTo(4000, under(4000)); sh.lineTo(0, under(0)); sh.closePath();
            const geo = new THREE.ExtrudeGeometry(sh, { depth: 180, bevelEnabled: false }), m = new THREE.Mesh(geo, mats.WALL_CREAM);
            m.position.z = zPos; m.castShadow = m.receiveShadow = true; m.add(edges(geo));
            m.userData = { name, category: 'WALLS', materialName: 'Tường đầu hồi xây kín 180mm', dims: '8000 × 180 × 1500 mm', pos: '(0, ' + Math.round(-zPos) + ', 3000)' };
            collections.ROOF.add(m); interactiveMeshes.push(m);
        }
        gable(-200, 'Tường đầu hồi mặt tiền'); gable(-5980, 'Tường đầu hồi mặt hậu');
        [-40, 3000, 6040].forEach(ty => {   // 3 vì kèo: đầu hồi trước, giữa nhà, đầu hồi sau
            R(`Xà ngang vì kèo (Y=${ty})`, 'Gỗ Lim 120×120', [700, ty - 60, 3200], [7300, ty + 60, 3320], mats.WOOD_FRAME);
            R(`Cột trụ giữa vì kèo (Y=${ty})`, 'Gỗ Lim 120×120', [3940, ty - 60, 3320], [4060, ty + 60, 4340], mats.WOOD_FRAME);
            const len = Math.hypot(1100, under(2900) - 3320), ang = Math.atan2(under(2900) - 3320, 1100);
            [-1, 1].forEach(s => rbox(`Thanh chống xiên ${s < 0 ? 'trái' : 'phải'} (Y=${ty})`, 'ROOF', 'Gỗ Lim 110×100', len, 110, 100, [4000 + s * 550, (3320 + under(2900)) / 2, ty], s * ang, mats.WOOD_FRAME, collections.ROOF));
        });

        // ---------- 5. Không có nội thất: chỉ giữ phòng trống ----------

        // ===== KÍCH THƯỚC: chỉ giữ số dài – rộng – cao =====
        function dim(p1, p2, txt) {
            const v = p => new THREE.Vector3(p[0], p[2], -p[1]);
            collections.DIMENSIONS.add(new THREE.Line(new THREE.BufferGeometry().setFromPoints([v(p1), v(p2)]), mats.DIM_LINE));
            if (!txt) return;
            const d = document.createElement('div'); d.className = 'dim'; d.textContent = txt;
            const o = new CSS2DObject(d); o.position.set((p1[0] + p2[0]) / 2, (p1[2] + p2[2]) / 2, -(p1[1] + p2[1]) / 2);
            collections.DIMENSIONS.add(o);
        }
        const zG = 30;
        dim([0, -1100, zG], [3000, -1100, zG], '3000'); dim([3000, -1100, zG], [5000, -1100, zG], '2000'); dim([5000, -1100, zG], [8000, -1100, zG], '3000');
        dim([0, -1750, zG], [8000, -1750, zG], '8000');
        [0, 3000, 5000, 8000].forEach(x => dim([x, -760, zG], [x, -1850, zG]));
        dim([9100, 0, zG], [9100, 3000, zG], '3000'); dim([9100, 3000, zG], [9100, 6000, zG], '3000'); dim([9850, 0, zG], [9850, 6000, zG], '6000');
        [0, 3000, 6000].forEach(y => dim([8100, y, zG], [9950, y, zG]));
        dim([-950, 0, 0], [-950, 0, 3000], '3000'); dim([-950, 0, 3000], [-950, 0, 4500], '1500'); dim([-1700, 0, 0], [-1700, 0, 4500], '4500');
        [0, 3000, 4500].forEach(h => dim([-1750, 0, h || zG], [-100, 0, h || zG]));

        // ===== BỐI CẢNH: bãi cỏ tròn, lối đi, cây xanh để nhà nổi bật =====
        const put = (geo, mat, x, y, z, sy = 1) => { const m = new THREE.Mesh(geo, mat); m.position.set(x, y, z); m.scale.y = sy; m.castShadow = m.receiveShadow = true; scene.add(m); return m; };
        put(new THREE.CylinderGeometry(7500, 7500, 60, 96), std(0x62b24d, 0.95), 0, -30, 0);
        put(new THREE.CylinderGeometry(7480, 7480, 500, 96), std(0x7a5a3a, 0.95), 0, -310, 0);
        put(new THREE.BoxGeometry(1300, 24, 3600), std(0xe3ddd3, 0.8), 0, 12, 5510);

        // ===== ĐIỀU KHIỂN =====
        let spin = false, si = 1, roofY = 0, roofTarget = 0;
        const speeds = [[1.2, 'Chậm'], [3, 'Vừa'], [6, 'Nhanh']];
        const setSpin = v => { spin = v; controls.autoRotate = v; $('b-spin').classList.toggle('on', v); $('b-spin').textContent = v ? '⏸ Dừng xoay' : '▶ Xoay 360°'; };
        $('b-spin').onclick = () => setSpin(!spin);
        $('b-speed').onclick = () => { si = (si + 1) % 3; controls.autoRotateSpeed = speeds[si][0]; $('b-speed').textContent = '⚡ Tốc độ: ' + speeds[si][1]; };
        $('b-roof').onclick = e => { const open = roofTarget === 0; roofTarget = open ? 5000 : 0; e.currentTarget.classList.toggle('on', open); e.currentTarget.textContent = open ? '🏠 Đóng mái' : '🏠 Mở mái'; };
        let dimsOn = true;
        const setDims = v => { dimsOn = v; collections.DIMENSIONS.children.forEach(c => c.visible = v); $('b-dim').classList.toggle('on', v); };
        setDims(false);   // mặc định không hiện kích thước
        $('b-dim').onclick = () => setDims(!dimsOn);
        // Bấm (chạm nhẹ, không kéo) vào ngôi nhà để hiện/ẩn kích thước; kéo để xoay thì không bị nhầm
        let down = null;
        renderer.domElement.addEventListener('pointerdown', e => { down = [e.clientX, e.clientY, performance.now()]; });
        renderer.domElement.addEventListener('pointerup', e => {
            if (!down || document.body.classList.contains('rec')) return;
            const moved = Math.hypot(e.clientX - down[0], e.clientY - down[1]), t = performance.now() - down[2]; down = null;
            if (moved > 6 || t > 400) return;
            const r = renderer.domElement.getBoundingClientRect(), ray = new THREE.Raycaster();
            ray.setFromCamera(new THREE.Vector2(((e.clientX - r.left) / r.width) * 2 - 1, -((e.clientY - r.top) / r.height) * 2 + 1), camera);
            if (ray.intersectObjects(interactiveMeshes, false).length) setDims(!dimsOn);
        });
        $('b-reset').onclick = resetView;
        const toast = $('toast');
        $('b-rec').onclick = () => { document.body.classList.add('rec'); resetView(); setSpin(true); toast.classList.add('show'); setTimeout(() => toast.classList.remove('show'), 3000); };
        const exitRec = () => document.body.classList.remove('rec');
        addEventListener('dblclick', exitRec);
        addEventListener('keydown', e => { if (e.key === 'Escape') exitRec(); });
        addEventListener('resize', () => {
            camera.aspect = container.clientWidth / container.clientHeight; camera.updateProjectionMatrix();
            renderer.setSize(container.clientWidth, container.clientHeight); labelRenderer.setSize(container.clientWidth, container.clientHeight);
        });
        (function animate() {
            requestAnimationFrame(animate);
            roofY += (roofTarget - roofY) * 0.08; collections.ROOF.position.y = roofY;
            controls.update();
            renderer.render(scene, camera); labelRenderer.render(scene, camera);
        })();
</script>
</body>
</html>
