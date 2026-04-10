[script.js](https://github.com/user-attachments/files/26631391/script.js)
document.addEventListener("DOMContentLoaded", () => {
    let timerInterval = null;
    let isRunning = false;
    let isPaused = false;
    let audioContext = null;

    let timers = [];
    let nextId = 1;
    const colors = ["#00f2fe", "#fe0979", "#00ff87", "#f8ff00", "#9b00e8", "#ff007f", "#ff8c00"];

    const timersContainer = document.getElementById("timers-container");
    const setupSection = document.getElementById("setup-section");
    const inputName = document.getElementById("timer-name");
    const inputMins = document.getElementById("timer-mins");
    const inputSecs = document.getElementById("timer-secs");
    const btnAdd = document.getElementById("btn-add");

    const btnStart = document.getElementById("btn-start");
    const btnPause = document.getElementById("btn-pause");
    const btnReset = document.getElementById("btn-reset");

    // Add Timer Event
    btnAdd.addEventListener("click", () => {
        let name = inputName.value.trim() || `タイマー ${nextId}`;
        let m = parseInt(inputMins.value) || 0;
        let s = parseInt(inputSecs.value) || 0;

        let totalSecs = m * 60 + s;
        if (totalSecs <= 0) {
            alert("1秒以上の時間を設定してください。");
            return;
        }

        addTimer(name, totalSecs);
        
        inputName.value = "";
        inputMins.value = "";
        inputSecs.value = "";
    });

    function addTimer(name, durationSec) {
        const id = nextId++;
        const color = colors[(id - 1) % colors.length];

        const card = document.createElement("div");
        card.className = "timer-card";
        card.id = `timer-card-${id}`;
        card.style.opacity = "0"; // For entrance animation
        card.style.transform = "scale(0.8)";

        card.innerHTML = `
            <button class="btn-delete" data-id="${id}">×</button>
            <div class="timer-title">${name}</div>
            <svg class="progress-ring" width="200" height="200">
                <circle class="progress-ring__circle-bg" stroke="rgba(255, 255, 255, 0.1)" stroke-width="8" fill="transparent" r="90" cx="100" cy="100"/>
                <circle class="progress-ring__circle" stroke="${color}" stroke-width="8" fill="transparent" r="90" cx="100" cy="100" id="circle-${id}"/>
            </svg>
            <div class="time-display" id="display-${id}">00:00</div>
            <div class="status-text" id="status-${id}">Ready</div>
        `;

        timersContainer.appendChild(card);

        // Entrance animation
        requestAnimationFrame(() => {
            setTimeout(() => {
                card.style.opacity = "1";
                card.style.transform = "scale(1) translateY(0)";
            }, 10);
        });

        // Delete Event
        card.querySelector(".btn-delete").addEventListener("click", () => {
            // Exit animation
            card.style.transform = "scale(0.8)";
            card.style.opacity = "0";
            setTimeout(() => {
                if(timersContainer.contains(card)) {
                    timersContainer.removeChild(card);
                    timers = timers.filter(t => t.id !== id);
                    updateGlobalButtons();
                }
            }, 300);
        });

        const circle = document.getElementById(`circle-${id}`);
        const radius = circle.r.baseVal.value;
        const circumference = radius * 2 * Math.PI;

        circle.style.strokeDasharray = `${circumference} ${circumference}`;
        circle.style.strokeDashoffset = circumference;

        const timerObj = {
            id,
            name,
            durationSec,
            remainingSec: durationSec,
            color,
            finished: false,
            elements: {
                card,
                circle,
                display: document.getElementById(`display-${id}`),
                status: document.getElementById(`status-${id}`),
                btnDelete: card.querySelector(".btn-delete")
            },
            circumference,
            endTime: 0
        };

        timers.push(timerObj);
        updateTimerDisplay(timerObj);
        
        if (isRunning) {
            timerObj.endTime = Date.now() + timerObj.remainingSec * 1000;
            timerObj.elements.status.textContent = "Running";
        } else if (isPaused) {
            timerObj.elements.status.textContent = "Paused";
        }

        updateGlobalButtons();
    }

    function updateTimerDisplay(t) {
        t.elements.display.textContent = formatTime(t.remainingSec);
        
        let percent = t.remainingSec > 0 ? ((t.durationSec - t.remainingSec) / t.durationSec) * 100 : 100;
        const offset = t.circumference - (percent / 100) * t.circumference;
        t.elements.circle.style.strokeDashoffset = offset;
    }

    function updateGlobalButtons() {
        if (timers.length === 0) {
            btnStart.disabled = true;
            btnReset.disabled = true;
            btnPause.disabled = true;
            btnStart.textContent = "START ALL";
            
            if (isRunning || isPaused) {
                clearInterval(timerInterval);
                isRunning = false;
                isPaused = false;
            }
            return;
        }

        setupSection.classList.remove("disabled");
        timers.forEach(t => t.elements.btnDelete.style.display = "flex");
        
        if (!isRunning && !isPaused) {
            btnStart.disabled = false;
            btnStart.textContent = "START ALL";
            btnReset.disabled = true;
            btnPause.disabled = true;
        } else if (isRunning) {
            btnStart.disabled = true;
            btnPause.disabled = false;
            btnReset.disabled = false;
        } else if (isPaused) {
            btnStart.disabled = false;
            btnStart.textContent = "RESUME ALL";
            btnPause.disabled = true;
            btnReset.disabled = false;
        }
    }

    function formatTime(seconds) {
        if (seconds <= 0) return "00:00";
        if(seconds >= 3600) {
            const h = Math.floor(seconds / 3600);
            const m = Math.floor((seconds % 3600) / 60).toString().padStart(2, '0');
            const s = (seconds % 60).toString().padStart(2, '0');
            return `${h}:${m}:${s}`;
        }
        const m = Math.floor(seconds / 60).toString().padStart(2, '0');
        const s = (seconds % 60).toString().padStart(2, '0');
        return `${m}:${s}`;
    }

    function initAudio() {
        if (!audioContext) {
            audioContext = new (window.AudioContext || window.webkitAudioContext)();
        }
        if (audioContext.state === 'suspended') {
            audioContext.resume();
        }
    }

    function playAlarmSound() {
        initAudio();
        let startTime = audioContext.currentTime;
        for (let i = 0; i < 4; i++) {
            playBeep(startTime + i * 0.45);
        }
    }

    function playBeep(time) {
        const osc = audioContext.createOscillator();
        const gainNode = audioContext.createGain();
        
        osc.connect(gainNode);
        gainNode.connect(audioContext.destination);
        
        osc.type = 'triangle';
        osc.frequency.setValueAtTime(880, time);
        
        gainNode.gain.setValueAtTime(0, time);
        gainNode.gain.linearRampToValueAtTime(0.8, time + 0.05);
        gainNode.gain.exponentialRampToValueAtTime(0.01, time + 0.35);
        
        osc.start(time);
        osc.stop(time + 0.4);
    }

    function handleFinish(t) {
        t.elements.card.classList.add("finished", "animating");
        t.elements.status.textContent = "FINISHED!";
        setTimeout(() => t.elements.card.classList.remove("animating"), 500);
        
        // Match color for display and text completion
        t.elements.display.style.color = t.color;
        
        playAlarmSound();

        // バイブレーション (主にAndroid用)
        if ("vibrate" in navigator) {
            navigator.vibrate([500, 200, 500, 200, 500]);
        }

        // OS通知 (バナー通知、スマートウォッチ等の連携含む)
        if ("Notification" in window && Notification.permission === "granted") {
            new Notification(`タイマー終了: ${t.name}`, {
                body: "時間がゼロになりました！",
            });
        }
    }

    function tick() {
        let allFinished = true;
        const now = Date.now();

        timers.forEach(t => {
            if (t.remainingSec > 0 && !t.finished) {
                let msLeft = t.endTime - now;
                t.remainingSec = Math.max(0, Math.ceil(msLeft / 1000));
                
                if (t.remainingSec === 0 && !t.finished) {
                    t.finished = true;
                    handleFinish(t);
                }
                updateTimerDisplay(t);
            }
            if (!t.finished) allFinished = false;
        });

        const noRemaining = timers.every(t => t.remainingSec === 0);
        if (noRemaining && timers.length > 0) {
            clearInterval(timerInterval);
            isRunning = false;
            btnStart.disabled = true;
            btnPause.disabled = true;
            btnStart.textContent = "DONE";
        }
    }

    function startTimer() {
        initAudio();

        if ("Notification" in window && Notification.permission === "default") {
            Notification.requestPermission();
        }
        
        const noRemaining = timers.every(t => t.remainingSec === 0);
        if (noRemaining || timers.length === 0) return;
        
        if (!isRunning) {
            isRunning = true;
            isPaused = false;
            
            const now = Date.now();
            timers.forEach(t => {
                if (t.remainingSec > 0 && !t.finished) {
                    t.endTime = now + t.remainingSec * 1000;
                    t.elements.status.textContent = "Running";
                }
            });

            timerInterval = setInterval(tick, 500); // More frequent updates for smoothness
            
            updateGlobalButtons();
        }
    }

    function pauseTimer() {
        if (isRunning) {
            clearInterval(timerInterval);
            isRunning = false;
            isPaused = true;
            
            const now = Date.now();
            timers.forEach(t => {
                if (t.remainingSec > 0 && !t.finished) {
                    let msLeft = t.endTime - now;
                    t.remainingSec = Math.max(0, Math.ceil(msLeft / 1000));
                    t.elements.status.textContent = "Paused";
                }
            });
            updateGlobalButtons();
        }
    }

    function resetTimer() {
        clearInterval(timerInterval);
        isRunning = false;
        isPaused = false;
        
        timers.forEach(t => {
            t.remainingSec = t.durationSec;
            t.finished = false;
            t.elements.card.classList.remove("finished");
            t.elements.display.style.color = "";
            t.elements.status.textContent = "Ready";
            updateTimerDisplay(t);
        });
        
        updateGlobalButtons();
    }

    // Bind Controls
    btnStart.addEventListener("click", startTimer);
    btnPause.addEventListener("click", pauseTimer);
    btnReset.addEventListener("click", resetTimer);

    // Provide initial examples for user (40 min and 50 min)
    addTimer("作業 (40分)", 40 * 60);
    addTimer("休憩・予備 (50分)", 50 * 60);
});
