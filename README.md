<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>BaldSquid Latency Calculator: DLSS 4.5 Expanded</title>
    <style>
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background-color: #121212; color: #e0e0e0; margin: 0; padding: 20px; }
        .container { max-width: 800px; margin: 0 auto; background: #1e1e1e; padding: 30px; border-radius: 8px; border: 1px solid #333; }
        h1 { color: #f39c12; margin-top: 0; text-transform: uppercase; font-size: 1.5rem; letter-spacing: 1px; border-bottom: 2px solid #333; padding-bottom: 10px; }
        .control-group { margin-bottom: 20px; }
        label { display: block; margin-bottom: 5px; font-weight: bold; color: #aaa; }
        input[type="range"] { width: 100%; margin-bottom: 10px; accent-color: #f39c12; }
        select { background: #333; color: #fff; border: 1px solid #555; padding: 8px; border-radius: 4px; width: 100%; }
        .dashboard { display: grid; grid-template-columns: repeat(2, 1fr); gap: 15px; margin: 30px 0; padding: 20px; background: #000; border-radius: 6px; }
        .metric { text-align: center; }
        .metric-title { font-size: 0.9rem; color: #888; text-transform: uppercase; }
        .metric-value { font-size: 2rem; font-weight: bold; color: #fff; }
        .highlight { color: #e74c3c; }
        .good { color: #2ecc71; }
        .warning { color: #f1c40f; }
        /* Chart Styles */
        .chart-container { margin-top: 30px; }
        .bar-row { margin-bottom: 15px; }
        .bar-label { margin-bottom: 5px; font-size: 0.9rem; font-weight: bold; }
        .bar-track { width: 100%; height: 30px; background: #222; border-radius: 4px; display: flex; overflow: hidden; }
        .segment { height: 100%; display: flex; align-items: center; justify-content: center; font-size: 0.8rem; font-weight: bold; color: #000; transition: width 0.3s ease; }
        .seg-queue { background-color: #95a5a6; }
        .seg-render { background-color: #3498db; }
        .seg-fg { background-color: #e74c3c; }
        .disclaimer { font-size: 0.85rem; color: #666; font-style: italic; margin-top: 20px; }
    </style>
</head>
<body>

<div class="container">
    <h1>Performance Audit: DLSS 4.5 Latency Tax</h1>
    
    <div class="control-group">
        <label for="baseFps">Native Target FPS: <span id="baseFpsVal">60</span></label>
        <input type="range" id="baseFps" min="30" max="360" value="60" oninput="updateUI()">
    </div>

    <div class="control-group" style="display: grid; grid-template-columns: 1fr 1fr; gap: 20px;">
        <div>
            <label for="reflexState">Nvidia Reflex State</label>
            <select id="reflexState" onchange="updateUI()">
                <option value="off">Off (Standard DX Pipeline)</option>
                <option value="on">Reflex On</option>
                <option value="boost" selected>Reflex + Boost</option>
            </select>
        </div>
        <div>
            <label for="fgToggle">Frame Generation</label>
            <select id="fgToggle" onchange="handleFGChange()">
                <option value="off" selected>Off (Native)</option>
                <option value="dlss3">DLSS 3 (2X Multiplier)</option>
                <option value="dlss45_2x">DLSS 4.5 (2X Multiplier)</option>
                <option value="dlss45_3x">DLSS 4.5 (3X Multiplier)</option>
                <option value="dlss45_4x">DLSS 4.5 (4X Multiplier)</option>
                <option value="dlss45_5x">DLSS 4.5 (5X Multiplier)</option>
                <option value="dlss45_6x">DLSS 4.5 (6X Multiplier)</option>
            </select>
        </div>
    </div>

    <div class="control-group" id="overheadGroup" style="opacity: 0.3; pointer-events: none;">
        <label for="overhead">AI Transformer Overhead Tax: <span id="overheadVal">12</span>%</label>
        <input type="range" id="overhead" min="0" max="25" value="12" oninput="updateUI()">
    </div>

    <div class="dashboard">
        <div class="metric">
            <div class="metric-title">Actual Base Render (FPS)</div>
            <div class="metric-value" id="actualFpsDisplay">60</div>
        </div>
        <div class="metric">
            <div class="metric-title">Final Output FPS</div>
            <div class="metric-value" id="outputFpsDisplay">60</div>
        </div>
        <div class="metric" style="grid-column: span 2;">
            <div class="metric-title">Total Input Latency</div>
            <div class="metric-value" id="latencyDisplay">14.7 ms</div>
        </div>
    </div>

    <div class="chart-container">
        <div class="bar-row">
            <div class="bar-label">Pipeline Latency Breakdown (Max 120ms scale)</div>
            <div class="bar-track" id="chartTrack">
                <div class="segment seg-queue" id="barQueue" style="width: 0%;"></div>
                <div class="segment seg-render" id="barRender" style="width: 0%;"></div>
                <div class="segment seg-fg" id="barFg" style="width: 0%;"></div>
            </div>
        </div>
    </div>

    <div class="disclaimer">
        *DLSS 4.5 generates up to 5 artificial frames per 1 physical frame. The 'FG Penalty' accounts for the structural wait time of the subsequent physical frame, plus a scaling compute time penalty (~3ms per generated frame) for the 2nd Gen Transformer model. 
    </div>
</div>

<script>
    function handleFGChange() {
        const fg = document.getElementById('fgToggle').value;
        const reflex = document.getElementById('reflexState');
        const overheadGroup = document.getElementById('overheadGroup');
        
        if (fg !== 'off') {
            overheadGroup.style.opacity = '1';
            overheadGroup.style.pointerEvents = 'auto';
            if (reflex.value === 'off') reflex.value = 'on'; 
        } else {
            overheadGroup.style.opacity = '0.3';
            overheadGroup.style.pointerEvents = 'none';
        }
        updateUI();
    }

    function updateUI() {
        const baseFps = parseInt(document.getElementById('baseFps').value);
        const reflexState = document.getElementById('reflexState').value;
        const fgMode = document.getElementById('fgToggle').value;
        const overhead = parseInt(document.getElementById('overhead').value);
        const scanout = 3; 

        document.getElementById('baseFpsVal').innerText = baseFps;
        document.getElementById('overheadVal').innerText = overhead;

        let actualFps = (fgMode !== 'off') ? baseFps * (1 - (overhead / 100)) : baseFps;
        
        let multiplier = 1;
        if (fgMode === 'dlss3') multiplier = 2;
        if (fgMode === 'dlss45_2x') multiplier = 2;
        if (fgMode === 'dlss45_3x') multiplier = 3;
        if (fgMode === 'dlss45_4x') multiplier = 4;
        if (fgMode === 'dlss45_5x') multiplier = 5;
        if (fgMode === 'dlss45_6x') multiplier = 6;

        let outputFps = actualFps * multiplier;
        
        let frameTime = 1000 / actualFps;
        let driverQueue = (reflexState === 'off') ? (frameTime * 2) : 0;
        let boostMod = (reflexState === 'boost') ? 0.9 : 1.0;
        
        let renderLatency = (frameTime * boostMod);
        
        let fgPenalty = 0;
        if (fgMode === 'dlss3') {
            fgPenalty = frameTime + 8; // Older OFA compute
        } else if (fgMode.startsWith('dlss45')) {
            let generatedFrames = multiplier - 1;
            // Transformer compute time scales with number of generated frames (~3ms per frame)
            fgPenalty = frameTime + (generatedFrames * 3); 
        }
        
        let totalLatency = driverQueue + renderLatency + fgPenalty + scanout;

        document.getElementById('actualFpsDisplay').innerText = Math.round(actualFps);
        document.getElementById('outputFpsDisplay').innerText = Math.round(outputFps);
        
        let latencyEl = document.getElementById('latencyDisplay');
        latencyEl.innerText = totalLatency.toFixed(1) + ' ms';
        
        if (totalLatency > 50) {
            latencyEl.className = 'metric-value highlight';
        } else if (totalLatency > 30) {
            latencyEl.className = 'metric-value warning';
        } else {
            latencyEl.className = 'metric-value good';
        }

        const maxScale = 120;
        document.getElementById('barQueue').style.width = Math.min((driverQueue / maxScale) * 100, 100) + '%';
        document.getElementById('barQueue').innerText = driverQueue > 5 ? 'DX Queue' : '';
        
        document.getElementById('barRender').style.width = Math.min((renderLatency / maxScale) * 100, 100) + '%';
        document.getElementById('barRender').innerText = renderLatency > 5 ? 'Render' : '';
        
        document.getElementById('barFg').style.width = Math.min((fgPenalty / maxScale) * 100, 100) + '%';
        document.getElementById('barFg').innerText = fgPenalty > 0 ? 'Hold + AI Gen' : '';
    }

    updateUI();
</script>

</body>
</html>
