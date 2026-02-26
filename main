// ==UserScript==
// @name         GeoFS Precise Stats - Peak Impact
// @namespace    http://tampermonkey.net/
// @version      2.1
// @description  Captures PEAK descent rate before impact to ensure accuracy for hard landings.
// @author       User
// @match        https://www.geo-fs.com/geofs.php*
// @match        https://geo-fs.com/geofs.php*
// @grant        none
// ==/UserScript==

(function() {
    'use strict';

    const MENU_KEY = "l"; 

    let checkInit = setInterval(() => {
        if (window.geofs && window.geofs.animation && window.geofs.animation.values && geofs.aircraft && geofs.aircraft.instance) {
            clearInterval(checkInit);
            initLandingStats();
        }
    }, 2500);

    function initLandingStats() {
        const statsBox = document.createElement('div');
        statsBox.style = `position:fixed; top:20px; left:20px; background:#111; color:#fff; padding:15px; border-radius:4px; font-family:sans-serif; font-weight:bold; z-index:10000; display:none; border-left:5px solid #3498db; box-shadow:0px 10px 30px rgba(0,0,0,0.8); min-width:250px;`;
        document.body.appendChild(statsBox);

        const logbookMenu = document.createElement('div');
        logbookMenu.style = `position:fixed; top:50%; left:50%; transform:translate(-50%, -50%); background:#111; color:#fff; padding:20px; border-radius:8px; font-family:sans-serif; z-index:10001; display:none; border:1px solid #3498db; width:400px; max-height:500px; overflow-y:auto; box-shadow:0 0 50px rgba(0,0,0,0.9); text-align: left;`;
        document.body.appendChild(logbookMenu);

        let wasInAir = false;
        let bounceCount = 0;
        let landingLocked = false; 
        let firstFPM = 0;
        
        // --- NEW: HIGH FREQUENCY BUFFER ---
        let vsBuffer = []; 
        setInterval(() => {
            let currentVS = geofs.animation.values.verticalSpeed;
            vsBuffer.push(currentVS);
            if (vsBuffer.length > 10) vsBuffer.shift(); // Keep last 0.5s of flight data
        }, 50);

        document.addEventListener('keydown', (e) => {
            if (e.key.toLowerCase() === MENU_KEY && !geofs.isPaused()) {
                logbookMenu.style.display = logbookMenu.style.display === "none" ? "block" : "none";
                if (logbookMenu.style.display === "block") updateLogbookUI();
            }
        });

        async function getRealLocation(lla) {
            try {
                const response = await fetch(`https://nominatim.openstreetmap.org/reverse?format=json&lat=${lla[0]}&lon=${lla[1]}&zoom=12`);
                const data = await response.json();
                return data.address.city || data.address.town || data.address.state || "Global Region";
            } catch (e) { return "GPS COORDINATES"; }
        }

        function saveLanding(data) {
            let log = JSON.parse(localStorage.getItem('geofs_logbook') || "[]");
            log.unshift(data);
            if (log.length > 50) log.pop();
            localStorage.setItem('geofs_logbook', JSON.stringify(log));
        }

        function updateLogbookUI() {
            let log = JSON.parse(localStorage.getItem('geofs_logbook') || "[]");
            let html = `<h3 style="margin-top:0; color:#3498db; border-bottom:1px solid #333; padding-bottom:10px;">FLIGHT LOGBOOK</h3>`;
            log.forEach(entry => {
                html += `<div style="margin-bottom:10px; font-size:12px; border-bottom:1px solid #222; padding-bottom:5px;">
                    <span style="color:#3498db;">${entry.ac}</span> @ ${entry.loc}<br>
                    <span style="font-size:16px;">${entry.fpm} FPM</span> | ${entry.tas} KTS | Bounces: ${entry.bounces}
                </div>`;
            });
            html += `
                <button id="clearLog" style="width:100%; padding:10px; background:#e74c3c; border:none; color:white; font-weight:bold; cursor:pointer; border-radius:4px; margin-top:10px;">CLEAR ALL DATA</button>
                <button id="closeLog" style="width:100%; padding:10px; background:#3498db; border:none; color:white; font-weight:bold; cursor:pointer; border-radius:4px; margin-top:5px;">CLOSE</button>
            `;
            logbookMenu.innerHTML = html;
            document.getElementById('clearLog').onclick = () => { if(confirm("Clear log?")) { localStorage.setItem('geofs_logbook', "[]"); updateLogbookUI(); }};
            document.getElementById('closeLog').onclick = () => logbookMenu.style.display = "none";
        }

        setInterval(async () => {
            const isGrounded = geofs.animation.values.groundContact;
            
            if (isGrounded && wasInAir) {
                const tas = Math.round(geofs.aircraft.instance.trueAirSpeed * 1.94384);
                const aircraftName = geofs.aircraft.instance.aircraftRecord.name || geofs.aircraft.instance.setup.name || "AIRCRAFT";
                
                if (statsBox.style.display !== "block") {
                    bounceCount = 0;
                    
                    // --- CAPTURE PEAK FROM BUFFER ---
                    // We look for the MOST NEGATIVE value in our 0.5s buffer
                    firstFPM = Math.round(Math.min(...vsBuffer)); 
                    
                    const city = await getRealLocation(geofs.aircraft.instance.llaLocation);
                    statsBox.setAttribute('data-loc', city);
                    
                    if (!landingLocked) {
                        saveLanding({ ac: aircraftName, loc: city, fpm: firstFPM, tas: tas, bounces: 0 });
                        landingLocked = true;
                        setTimeout(() => { landingLocked = false; }, 20000);
                    }
                    statsBox.style.display = "block";
                } else {
                    bounceCount++;
                    let log = JSON.parse(localStorage.getItem('geofs_logbook'));
                    if (log && log.length > 0) {
                        log[0].bounces = bounceCount;
                        localStorage.setItem('geofs_logbook', JSON.stringify(log));
                    }
                }

                const location = statsBox.getAttribute('data-loc') || "Detecting...";
                statsBox.innerHTML = `
                    <div style="font-size:11px; color:#3498db; letter-spacing:1.5px; margin-bottom:2px;">${aircraftName.toUpperCase()}</div>
                    <div style="font-size:11px; color:#999; border-bottom:1px solid #333; padding-bottom:5px;">${location.toUpperCase()}</div>
                    <div style="font-size:32px; color:white; margin:5px 0;">${firstFPM} <span style="font-size:14px; opacity:0.6;">FPM</span></div>
                    <div style="font-size:14px;">TAS: <span style="color:#3498db;">${tas} KTS</span> | BOUNCES: <span style="color:#3498db;">${bounceCount}</span></div>
                `;

                clearTimeout(window.hideStats);
                window.hideStats = setTimeout(() => { statsBox.style.display = "none"; }, 12000);
            }
            wasInAir = !isGrounded;
        }, 100);
    }
})();
