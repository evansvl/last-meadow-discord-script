# Last Meadow Online - Automation Bot

This is an automation script for the Discord Activity game **"Last Meadow Online"**. It automates various in-game mechanics, solves puzzles, and perfectly plays minigames for you. 

---

## 🛠️ Step 1: How to Enable Discord Developer Console

Discord disables the developer console by default in its desktop app to protect users from self-XSS attacks. Here is how you enable it depending on what you use:

### Option A: Using Discord in a Web Browser (Easiest)
If you play the game via Chrome, Edge, or Firefox, you don't need to change any settings:
1. Press `F12` or `Ctrl + Shift + I` (Windows) / `Cmd + Option + I` (Mac).
2. Go to the **"Console"** tab.

### Option B: Using the Discord Desktop App
To enable the console in the downloaded Discord app:
1. Close Discord completely (make sure it's not running in the system tray).
2. Open your file explorer and go to your Discord settings folder:
   * **Windows:** Press `Win + R`, type `%appdata%\discord` and hit Enter.
   * **Mac:** Open Terminal and type `open ~/Library/Application\ Support/discord`
3. Open the `settings.json` file with any text editor (like Notepad).
4. Add the following line inside the brackets `{ }`, right above the last existing setting. Make sure to add a comma `,` to the previous line if needed:
   ```json
   "DANGEROUS_ENABLE_DEVTOOLS_ONLY_ENABLE_IF_YOU_KNOW_WHAT_YOURE_DOING": true
   ```
5. Save the file and start Discord.
6. Now you can press `Ctrl + Shift + I` (Windows) or `Cmd + Option + I` (Mac) to open the Developer Tools. Go to the **"Console"** tab.

---

## 🚀 Step 2: Running the Script

Copy the entire code block below, paste it into the console text box, and press **Enter**.

```javascript
const Y_OFFSET = 200; 

let isBotEnabled = true;
let isPaladinGameActive = false;
let paladinRafId = null;
let globalBotState = "INITIALIZING...";

let lock = null; 
let projIdCounter = 0; 

let debugUI = document.getElementById('lmo-debug-ui');
let debugStatus, btnStopBot;
if (!debugUI) {
    debugUI = document.createElement('div');
    debugUI.id = 'lmo-debug-ui';
    debugUI.style.cssText = [
        'position:fixed', 'top:80px', 'left:10px',
        'background:rgba(0,0,0,0.88)', 'color:white',
        'padding:12px', 'font-family:monospace', 'font-size:13px',
        'z-index:10000', 'pointer-events:none',
        'border:2px solid #555', 'border-radius:8px',
        'min-width:300px', 'box-shadow:0 4px 15px rgba(0,0,0,0.5)'
    ].join(';');
    debugUI.innerHTML = `<b style='color:#0f0;font-size:14px'>[ Last Meadow Online Bot ]</b><br><br>` +
        `Status: <span id='debugStatus' style='color:cyan'>${globalBotState}</span>` +
        `<div style="margin-top:12px;padding-top:12px;border-top:1px solid #444;display:flex;align-items:center;">
        <button id="btnStopBot" style="pointer-events:auto;background:#e74c3c;color:#fff;border:none;padding:6px 12px;border-radius:4px;cursor:pointer;font-weight:bold;font-family:monospace;">Turn Off</button>
        <span style="color:#aaa;font-size:11px;margin-left:10px;">or press Alt + S</span>
        </div>`;
    document.body.appendChild(debugUI);
    debugStatus = debugUI.querySelector("#debugStatus");
    btnStopBot = debugUI.querySelector("#btnStopBot");
}

function updateDebugUI(html = null) {
    if (debugStatus && html) { debugStatus.innerHTML = globalBotState + `<br><br><span style="color:white">${html}</span>` }
    else if (debugStatus && debugStatus.innerHTML !== globalBotState && !isPaladinGameActive) { debugStatus.innerHTML = globalBotState };
}

function simulateRealClick(el) {
    if (!el) return;['pointerdown','mousedown','mouseup','pointerup','click'].forEach(t =>
        el.dispatchEvent(new MouseEvent(t, { bubbles:true, cancelable:true, view:window, buttons:1 }))
    );
}

function simulateKeyPress(keyName) {
    const codes = { ArrowLeft:37, ArrowUp:38, ArrowRight:39, ArrowDown:40 };
    const cfg = { key:keyName, code:keyName, keyCode:codes[keyName]||0,
                  which:codes[keyName]||0, bubbles:true, cancelable:true, composed:true };
    document.dispatchEvent(new KeyboardEvent('keydown', cfg));
    setTimeout(() => document.dispatchEvent(new KeyboardEvent('keyup', cfg)), 25);
}

function simulatePointerMove(x, y, area) {
    const cfg = { clientX:x, clientY:y, bubbles:true, cancelable:true, view:window };
    area.dispatchEvent(new PointerEvent('pointermove', cfg));
    area.dispatchEvent(new MouseEvent('mousemove', cfg));
}

function blockRealMouse(e) {
    if (isBotEnabled && isPaladinGameActive && e.isTrusted) e.stopPropagation();
}

window.addEventListener('mousemove',   blockRealMouse, true);
window.addEventListener('pointermove', blockRealMouse, true);

const pData = new Map(); 

function runPaladinEngine() {
    if (!isBotEnabled || !isPaladinGameActive) return;

    const shield   = document.querySelector('[class*="shield_"]');
    const gameArea = document.querySelector('[class*="game__"]');
    const allP     = document.querySelectorAll('[class*="projectile_"]');

    if (!shield || !gameArea || allP.length === 0) {
        isPaladinGameActive = false;
        lock = null;
        pData.clear();
        globalBotState = "FINISHED SHIELD MINIGAME";
        updateDebugUI();
        return;
    }

    const shieldR  = shield.getBoundingClientRect();
    const catchY   = shieldR.top;           
    const now      = performance.now();

    globalBotState = "SHIELD MINIGAME";

    allP.forEach(el => {
        const r = el.getBoundingClientRect();
        const isVisible = r.width > 0 && r.height > 0 && window.getComputedStyle(el).opacity !== '0';

        if (!pData.has(el)) {
            if (!isVisible) return; 
            pData.set(el, {
                id:          ++projIdCounter,
                stableWidth: r.width, 
                prevY:       r.top,
                prevT:       now,
                speed:       0.2, 
                lastR:       r, 
                cx:          r.left + r.width / 2,
                isDead:      false
            });
        }

        const d = pData.get(el);
        if (d.isDead) return;

        if (isVisible) {
            if (r.top < d.prevY - 50) {
                d.id = ++projIdCounter; 
                d.stableWidth = r.width;
                d.speed = 0.2;
                d.isDead = false;
            }

            d.cx = r.left + r.width / 2;
            d.lastR = r;

            const dt = now - d.prevT;
            if (dt > 10) {
                const dy = r.top - d.prevY;
                d.speed = d.speed * 0.7 + (Math.max(0, dy) / dt) * 0.3; 
                d.prevY = r.top;
                d.prevT = now;
            }

            const virtualBottom = r.bottom - Y_OFFSET;
            
            if (virtualBottom >= catchY - 50 && r.width > d.stableWidth * 1.25) {
                d.isDead = true; 
            } else {
                if (r.width < d.stableWidth) {
                    d.stableWidth = r.width;
                } else {
                    d.stableWidth = d.stableWidth * 0.9 + r.width * 0.1;
                }
            }
        }
    });

    const valid = [];
    
    for (const [el, d] of pData) {
        const virtualTop = d.lastR.top - Y_OFFSET;
        
        if (!document.body.contains(el) || virtualTop > shieldR.bottom + 50) {
            pData.delete(el); 
            continue;
        }

        if (d.isDead) continue;

        const virtualBottom = d.lastR.bottom - Y_OFFSET;
        const distToShield = catchY - virtualBottom;
        const safeSpeed = Math.max(0.04, d.speed);
        
        const eta = distToShield <= 0 ? 0 : distToShield / safeSpeed;

        valid.push({ el, d, distToShield, eta, cx: d.cx });
    }

    if (lock) {
        const current = valid.find(v => v.d.id === lock.id);

        if (!current) {
            lock = null; 
        } else {
            valid.sort((a, b) => a.eta - b.eta);
            const best = valid[0];
            
            if (best && best.d.id !== current.d.id && best.eta < current.eta - 50) {
                lock = { id: best.d.id };
            }
        }
    }

    if (!lock && valid.length > 0) {
        valid.sort((a, b) => a.eta - b.eta);
        lock = { id: valid[0].d.id };
    }

    const target = lock ? valid.find(v => v.d.id === lock.id) : null;

    allP.forEach(el => { if (document.body.contains(el)) el.style.border = ''; });

    let ui = `Y-Offset: <b style='color:orange'>+${Y_OFFSET}px</b><br>` +
             `Memory: <b>${pData.size}</b> | Active: <b>${valid.length}</b><br><br>`;

    if (target) {
        if (document.body.contains(target.el)) {
            target.el.style.border = '3px solid #00ff00';
            target.el.style.boxSizing = 'border-box';
        }
        ui += `<span style='color:#0f0'>🚀 INTERCEPTING</span> | ETA: ${Math.round(target.eta)}ms<br><br>`;
    } else {
        ui += `<span style='color:gray'>STANDBY</span><br><br>`;
    }

    valid.forEach((v) => {
        const isT = target && v.d.id === target.d.id;
        ui += `<div style='color:${isT ? "#0f0" : "#777"}'>`;
        ui += `[ID:${v.d.id}] ETA:${Math.round(v.eta)}ms W:${Math.round(v.d.lastR.width)} Norm:${Math.round(v.d.stableWidth)}</div>`;
    });

    updateDebugUI(ui);

    if (target) {
        const shieldCY = shieldR.top + shieldR.height / 2;
        simulatePointerMove(Math.round(target.cx), Math.round(shieldCY), gameArea);
    }

    paladinRafId = requestAnimationFrame(runPaladinEngine);
}

const speedMs = 50;
let lastCraftKeyTime   = 0;
let lastPriestClickTime = 0;

const mainLoop = setInterval(() => {
    if (!isBotEnabled) return;

    const tryClick = (btn, state) => {
        if (btn && !btn.className.toLowerCase().includes('disabled')) {
            globalBotState = state;
            simulateRealClick(btn);
            updateDebugUI();
            return true;
        }
        return false;
    };

    let btnContinue, btnGoBack;
    const cw = document.querySelector('[class*="continueButtonWrapper_"]');
    if (cw) {
        btnContinue = cw.querySelector('[role="button"]');
        if (tryClick(btnContinue,  "CLICKING CONTINUE"))  return;
    }

    const gb = document.querySelector('[class*="fullScreenOnMobile_"] [class*="footer_"]');
    if (gb) {
        btnGoBack = gb.querySelectorAll('[role="button"]');
        if (btnGoBack.length === 1 && tryClick(btnGoBack[0],  "CLICKING GO BACK")) return;
    }

    const allP = document.querySelectorAll('[class*="projectile_"]');
    if (allP.length > 0) {
        if (!isPaladinGameActive) {
            isPaladinGameActive = true;
            lock = null;
            pData.clear();
            paladinRafId = requestAnimationFrame(runPaladinEngine);
        }
        return;
    } else if (isPaladinGameActive) {
        isPaladinGameActive = false;
        lock = null;
        pData.clear();
    }

    const targets = document.querySelectorAll('[class*="targetContainer_"]');
    if (targets.length > 0) {
        globalBotState = "SHOOTING TARGETS";
        targets.forEach(t => {
            if (!t._botShot) {
                simulateRealClick(t);
                if (t.firstElementChild) simulateRealClick(t.firstElementChild);
                t._botShot = true;
            }
        });
        updateDebugUI(); return;
    }

    const arrowImgs = document.querySelectorAll(
        '[class*="sequences_"] img[alt^="Arrow"],[class*="character_"] img[alt^="Arrow"]'
    );
    if (arrowImgs.length > 0) {
        let active = null;
        for (const img of arrowImgs) {
            if (parseFloat(window.getComputedStyle(img.parentElement).opacity) > 0.8 &&
                parseFloat(window.getComputedStyle(img).opacity) > 0.8) {
                active = img; break;
            }
        }
        if (active && Date.now() - lastCraftKeyTime > 150) {
            globalBotState = "CRAFTING ARROWS";
            simulateKeyPress(active.getAttribute('alt'));
            lastCraftKeyTime = Date.now();
            updateDebugUI(); return;
        }
    }

    const priestTiles = document.querySelectorAll('[class*="gridItem_"][role="button"]');
    if (priestTiles.length > 0) {
        globalBotState = "SOLVING PRIEST PUZZLE";
        if (Date.now() - lastPriestClickTime > 1500) {
            const groups = {};
            priestTiles.forEach(tile => {
                const svg = tile.querySelector('svg');
                if (!svg) return;
                const sig = svg.innerHTML;
                if (tile._botSig !== sig) { tile._botSolved = false; tile._botSig = sig; }
                if (tile._botSolved || parseFloat(window.getComputedStyle(tile).opacity) < 0.5) return;
                if (!groups[sig]) groups[sig] = [];
                groups[sig].push(tile);
            });
            for (const sig in groups) {
                if (groups[sig].length === 3) {
                    groups[sig].forEach((tile, i) => {
                        tile._botSolved = true;
                        setTimeout(() => simulateRealClick(tile), i * 150);
                    });
                    lastPriestClickTime = Date.now(); break;
                }
            }
        }
        updateDebugUI(); return;
    }

    let btnBattle, btnCraft, btnAdventure;

    document.querySelectorAll('img[class*="asset_"], img[class*="activityButtonAsset_"]').forEach(img => {
        const btn = img.closest('[role="button"]');
        if (!btn) return;
        const src = img.src || '';
        if (src.includes('0492e39') || src.includes('19393b5') || src.includes('16fb255')) btnBattle    = btn;
        if (src.includes('23aba2a') || src.includes('b7febb5') || src.includes('b603820')) btnCraft     = btn;
        if (src.includes('282df26'))                                                       btnAdventure = btn;
    });

    if (tryClick(btnBattle,    "STARTING BATTLE"))     return;
    if (tryClick(btnCraft,     "STARTING CRAFT"))      return;
    if (tryClick(btnAdventure, "STARTING ADVENTURE"))  return;

    globalBotState = "SCANNING / IDLE...";
    updateDebugUI();

}, speedMs);

window.stopBot = function() {
    isBotEnabled = false;
    isPaladinGameActive = false;
    clearInterval(mainLoop);
    if (paladinRafId) cancelAnimationFrame(paladinRafId);
    window.removeEventListener('mousemove',   blockRealMouse, true);
    window.removeEventListener('pointermove', blockRealMouse, true);
    document.removeEventListener('keydown', stopBotHotkey);
    btnStopBot.removeEventListener('click', window.stopBot);
    if (debugUI) debugUI.remove();
    console.log('🛑 BOT STOPPED');
};

function stopBotHotkey(e) { if (e.altKey && e.code === 'KeyS') window.stopBot(); }
btnStopBot.addEventListener('click', window.stopBot);
document.addEventListener('keydown', stopBotHotkey);

console.clear();
console.log('✈️ Last Meadow Online Bot STARTED | Alt+S to stop');
```

---

## 🛑 How to Stop the Bot
*   Click the red **"Turn Off"** button in the bot's UI overlay.
*   OR press `Alt + S` on your keyboard.
*   OR simply refresh Discord (`Ctrl + R`).

*** 

### ⚠️ Security Disclaimer
*Never paste code into your console from people you do not trust. Discord warns you about this because malicious scripts can steal your Discord login token. Always verify what the code does before hitting enter!*
