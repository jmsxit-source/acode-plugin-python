// enginebundle.js
import { SensitivityEngine } from './engine.js';

const engine = new SensitivityEngine();

// Estado do Input
let lastTouchX = 0;
let lastTouchY = 0;
let accumulatedDeltaX = 0;
let accumulatedDeltaY = 0;
let lastEventTime = 0;
let isTracking = false;

// Parâmetros do Filtro Passa-Baixa (LPF)
// RC = 1 / (2 * PI * f_cut). Para mobile gaming, uma frequência de corte entre 15Hz e 30Hz remove o jitter sem causar input lag perceptível.
const RC = 0.01; 
let filteredX = 0;
let filteredY = 0;

// Elemento alvo (ex: a área de trackpad do PWA)
const trackZone = document.getElementById('trackzone');

function init() {
    if (!trackZone) return;

    // Configuração dos Pointers para máxima performance no Chrome/Android
    trackZone.addEventListener('pointerdown', onPointerDown, { passive: false });
    trackZone.addEventListener('pointermove', onPointerMove, { passive: false });
    trackZone.addEventListener('pointerup', onPointerUp, { passive: false });
    trackZone.addEventListener('pointercancel', onPointerUp, { passive: false });

    // Inicia o Loop de Renderização casado com a tela
    requestAnimationFrame(gameLoop);
}

function onPointerDown(e) {
    e.preventDefault();
    isTracking = true;
    
    // Inicialização das variáveis com a primeira posição de toque
    lastTouchX = e.clientX;
    lastTouchY = e.clientY;
    filteredX = e.clientX;
    filteredY = e.clientY;
    lastEventTime = performance.now();
    accumulatedDeltaX = 0;
    accumulatedDeltaY = 0;
}

function onPointerMove(e) {
    if (!isTracking) return;
    e.preventDefault();

    const now = performance.now();
    const dt = (now - lastEventTime) / 1000; // em segundos
    lastEventTime = now;

    // Suporte a sub-frames (Dispositivos High-End com amostragem de toque > 60Hz)
    // O Chrome unifica eventos, mas o getCoalescedEvents nos dá acesso aos inputs intermediários reais do hardware
    const events = typeof e.getCoalescedEvents === 'function' ? e.getCoalescedEvents() : [e];
    
    for (let ev of events) {
        const rawX = ev.clientX;
        const rawY = ev.clientY;

        // Aplicação do Filtro Passa-Baixa para mitigar ruído (Jitter) do hardware
        // Formula: O_n = O_n-1 + alpha * (I_n - O_n-1)
        const alpha = dt / (RC + dt);
        filteredX = filteredX + alpha * (rawX - filteredX);
        filteredY = filteredY + alpha * (rawY - filteredY);

        // Calcula o delta em relação ao último frame filtrado
        const deltaX = filteredX - lastTouchX;
        const deltaY = filteredY - lastTouchY;

        // Acumula os deltas para processamento no requestAnimationFrame
        accumulatedDeltaX += deltaX;
        accumulatedDeltaY += deltaY;

        lastTouchX = filteredX;
        lastTouchY = filteredY;
    }
}

function onPointerUp(e) {
    isTracking = false;
    accumulatedDeltaX = 0;
    accumulatedDeltaY = 0;
}

// Loop principal sincronizado com o Refresh Rate da tela (Ex: 60Hz, 90Hz, 120Hz)
let lastFrameTime = performance.now();

function gameLoop(now) {
    const frameDeltaTime = now - lastFrameTime; // em milissegundos
    lastFrameTime = now;

    if (isTracking && (accumulatedDeltaX !== 0 || accumulatedDeltaY !== 0)) {
        
        // Passa os deltas acumulados e filtrados pela nossa engine de aceleração e clamping
        const finalMotion = engine.processMotion(accumulatedDeltaX, accumulatedDeltaY, frameDeltaTime);

        // Zera o acumulador do input consumido
        accumulatedDeltaX = 0;
        accumulatedDeltaY = 0;

        // DISPARAR MIRA (Aplique o valor final na sua Viewport / Canvas / Elemento de mira)
        updateCrosshair(finalMotion.x, finalMotion.y);
    }

    requestAnimationFrame(gameLoop);
}

function updateCrosshair(mx, my) {
    // Exemplo de aplicação: Custom Event para sua UI ou Engine de Renderização
    const event = new CustomEvent('crosshairMove', { detail: { x: mx, y: my } });
    window.dispatchEvent(event);
}

window.addEventListener('DOMContentLoaded', init);
