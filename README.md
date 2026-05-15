<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Simulación: Llenado de cámaras de frío y despacho</title>
  <style>
    :root { --bg: #0f1b2a; --card: #16233a; --text: #e9f0f8; --accent: #4cc9f0; }
    * { box-sizing: border-box; }
    body { margin: 0; font-family: Arial, sans-serif; background: #f4f7fb; color: #111; }
    header { padding: 16px; background: #1e2a3a; color: white; display: flex; align-items: center; justify-content: space-between; }
    header h1 { margin: 0; font-size: 1.1rem; }
    main { padding: 16px; display: grid; grid-template-columns: 360px 1fr; gap: 16px; }
    section.card { background: white; border-radius: 8px; padding: 12px; box-shadow: 0 2px 6px rgba(0,0,0,.08); }
    h2 { font-size: 1rem; margin: 8px 0 12px; }
    label { display: block; margin: 8px 0 4px; font-size: 0.92em; color: #333; }
    input[type="number"] { padding: 6px 8px; width: 100%; border: 1px solid #ccc; border-radius: 4px; }
    button { padding: 8px 12px; border: none; border-radius: 6px; background: #4cc9f0; color: white; cursor: pointer; margin-right: 6px; }
    button.secondary { background: #6c757d; }
    .row { display:flex; gap:8px; align-items:center; }
    canvas { width: 100% !important; max-width: 720px; height: 240px !important; }
    #log { white-space: pre-wrap; background: #f7f7f7; border: 1px solid #eee; padding: 8px; height: 120px; overflow:auto; border-radius: 6px; margin-top: 8px; }
    .grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 8px; }
    @media (max-width: 900px) {
      main { grid-template-columns: 1fr; }
    }
  </style>
</head>
<body>
  <header>
    <div>
      <strong>Simulación: Llenado de cámaras de frío y despacho</strong>
      <div style="font-size:12px; color:#cbd5e1;">Página web estática • no requiere descarga</div>
    </div>
    <div class="row">
      <button id="runBtn" title="Ejecutar simulación">Ejecutar simulación</button>
      <button id="resetBtn" class="secondary" title="Reiniciar">Reiniciar</button>
    </div>
  </header>

  <main>
    <!-- Panel de configuración -->
    <section class="card" aria-label="Configuración de simulación">
      <h2>Configuración</h2>

      <div class="grid-2">
        <div>
          <label>Tiempo de simulación (horas):</label>
          <input type="number" id="simTime" value="24" step="1" min="1"/>
        </div>
        <div>
          <label>Número de montacargas:</label>
          <input type="number" id="nForklifts" value="2" min="1" max="6"/>
        </div>
      </div>

      <div class="grid-2" style="margin-top:8px;">
        <div>
          <label>Tasa de llegada (tambores por hora):</label>
          <input type="number" id="arrivalRate" value="2" step="0.1" min="0.1"/>
        </div>
        <div>
          <label>Tiempo medio de almacenamiento (horas):</label>
          <input type="number" id="storageMean" value="0.5" step="0.1" min="0.05"/>
        </div>
      </div>

      <div class="grid-2" style="margin-top:8px;">
        <div>
          <label>Tiempo medio de despacho (horas por tambor):</label>
          <input type="number" id="dispatchMean" value="0.25" step="0.05" min="0.05"/>
        </div>
        <div>
          <label>Capacidades de cámaras (una por cámara). </label>
          <div id="cameraInputs" class="row" style="flex-wrap:wrap;">
            <!-- Campos de capacidad generados dinámicamente -->
          </div>
        </div>
      </div>

      <div id="logTitle" style="margin-top:10px; font-size:12px; color:#555;">
        Nota: la simulación asume procesos en segundos a modo de horas (escala de tiempo ficticia para animación en el navegador).
      </div>
    </section>

    <!-- Visualización -->
    <section class="card" aria-label="Resultados">
      <h2>Resultados</h2>
      <canvas id="occChart" height="240"></canvas>
      <div id="summary" style="margin-top:12px;">
        <strong>Resumen:</strong>
        <ul id="summaryList" style="margin:0; padding-left:20px;">
          <li>Despachos realizados: <span id="despachados">0</span></li>
          <li>Tiempo promedio de almacenamiento: <span id="avgStorage">0</span> h</li>
        </ul>
      </div>
      <div id="log" aria-label="Registro de eventos" style="margin-top:10px;"></div>
    </section>
  </main>

  <!-- Bibliotecas (CDN) -->
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

  <script>
    // -----------------------------
    // Simulación en una página única
    // DES (discrete-event simulation) simplificado en JS
    // -----------------------------

    // Utiles
    function expo(mean) {
      // Media = mean. Muestra de distribución exponencial
      return -Math.log(1 - Math.random()) * mean;
    }

    // DOM helpers
    const runBtn = document.getElementById('runBtn');
    const resetBtn = document.getElementById('resetBtn');
    const simTimeInput = document.getElementById('simTime');
    const arrivalRateInput = document.getElementById('arrivalRate');
    const storageMeanInput = document.getElementById('storageMean');
    const dispatchMeanInput = document.getElementById('dispatchMean');
    const nForkliftsInput = document.getElementById('nForklifts');
    const cameraInputsEl = document.getElementById('cameraInputs');
    const occChartCtx = document.getElementById('occChart').getContext('2d');
    const despachadosEl = document.getElementById('despachados');
    const avgStorageEl = document.getElementById('avgStorage');
    const summaryListEl = document.getElementById('summaryList');
    const logEl = document.getElementById('log');
    const logTitleEl = document.getElementById('logTitle');

    // Estado de configuración
    // Capacidad por cámara (inicial 3 cámaras)
    const defaultCapacities = [10, 8, 12];
    let cameras = []; // [{id, capacity, occupied, lastTime, accumOccupancy}]
    let drumCounter = 0;
    let drums = {}; // drumId -> {cameraId, storageStart}
    let blocked = []; // tambores esperando espacio
    let readyQueue = []; // tambores listos para despacho
    let totalDespachados = 0;

    // Control de recursos
    let forkliftAvailable = 0;

    // Métricas de tiempo
    let currentTime = 0; // horas en simulación
    let events = []; // cola de eventos: {time, type, drumId}

    // Registro de almacenamiento
    let storageTimes = []; // tiempos de almacenamiento por tambor

    // Gráficos
    let occChart = null;

    // Inicializar cámara panel
    function renderCameraInputs() {
      cameraInputsEl.innerHTML = '';
      cameras.forEach((cam, idx) => {
        const div = document.createElement('div');
        div.style.display = 'flex';
        div.style.alignItems = 'center';
        div.style.gap = '8px';
        div.style.marginRight = '6px';
        div.innerHTML = `
          <label style="min-width: 220px;">Cámara C${cam.id} - capacidad:</label>
          <input type="number" value="${cam.capacity}" min="1" data-index="${idx}" style="width: 90px;" />
        `;
        cameraInputsEl.appendChild(div);
      });
      // Añade un listener para cambios de capacidad
      cameraInputsEl.querySelectorAll('input[type="number"]').forEach(inp => {
        inp.addEventListener('change', () => {
          const idx = parseInt(inp.getAttribute('data-index'));
          const val = Math.max(1, parseInt(inp.value || '1'));
          cameras[idx].capacity = val;
          // Update internal UI si se desea
        });
      });
    }

    // Construye configuración inicial
    function initConfiguration() {
      // Reset
      cameras = [];
      for (let i = 0; i < defaultCapacities.length; i++) {
        cameras.push({
          id: i + 1,
          capacity: defaultCapacities[i],
          occupied: 0,
          lastTime: 0,
          accumOccupancy: 0
        });
      }
      renderCameraInputs();
      // Detalles de UI
      logEl.textContent = '';
      logTitleEl.style.display = 'block';
    }

    // Helpers para eventos
    function scheduleEvent(time, type, drumId) {
      events.push({ time, type, drumId });
      events.sort((a, b) => a.time - b.time);
    }

    function now() { return currentTime; }

    function updateCameraOccupancy(cam, newOccupied, time) {
      const deltaTime = time - cam.lastTime;
      cam.accumOccupancy += cam.occupied * deltaTime;
      cam.lastTime = time;
      cam.occupied = newOccupied;
    }

    function log(msg) {
      logEl.textContent += msg + "\n";
      logEl.scrollTop = logEl.scrollHeight;
    }

    // Controladores de simulación
    function placeTamborInCamera(drumId, cam, time) {
      drumCounter;
      drums[drumId] = drums[drumId] || {};
      drums[drumId].cameraId = cam.id;
      drums[drumId].storageStart = time;
      updateCameraOccupancy(cam, cam.occupied + 1, time);
      // Programar almacenamiento -> despacho
      const storageMean = parseFloat(storageMeanInput.value);
      const tStorage = expo(storageMean); // en horas
      scheduleEvent(time + tStorage, 'storage_end', drumId);
      log(`Hora ${time.toFixed(2)}: Tambor ${drumId} colocado en C${cam.id} (capacidad=${cam.capacity}, ocupación=${cam.occupied})`);
    }

    function tryPlaceBlocked(time) {
      // Intentar colocar tambores que estaban bloqueados
      if (blocked.length === 0) return;
      // Buscar cámara con espacio
      let placedAny = false;
      for (let i = 0; i < blocked.length; i++) {
        const drumId = blocked[i];
        const cam = cameras.find(c => c.occupied < c.capacity);
        if (!cam) break;
        // quitar de blocked
        blocked.splice(i, 1);
        // colocar
        drums[drumId] = drums[drumId] || {};
        drums[drumId].cameraId = cam.id;
        drums[drumId].storageStart = time;
        updateCameraOccupancy(cam, cam.occupied + 1, time);
        const storageMean = parseFloat(storageMeanInput.value);
        const tStorage = expo(storageMean);
        scheduleEvent(time + tStorage, 'storage_end', drumId);
        log(`Hora ${time.toFixed(2)}: Tambor bloqueado ${drumId} colocado en C${cam.id} (nuevo ocupación=${cam.occupied})`);
        placedAny = true;
        // continuar buscando
        i--; // since we removed current item
      }
      if (placedAny) {
        // Después de colocar uno o más, intentar despacho si procede
        attemptDispatch(time);
      }
    }

    function attemptDispatch(time) {
      // Despacho mientras haya tambores listos y montacargas disponibles
      const nForks = parseInt(nForkliftsInput.value) || 1;
      forkliftAvailable = Math.max(0, Math.min(nForks, 999)); // inicial ajuste
      // Ajusta variable de disponibilidad a diario: contamos despachos en progreso con un reloj simplificado
      // Para simplicidad, mantenemos un contador de despachos en progreso separado
      // En este prototipo, creamos despachos explícitos en 'dispatch_end'
      // Este helper se invoca cuando puedas iniciar despachos
      while (readyQueue.length > 0 && forkliftAvailable > 0) {
        const drumId = readyQueue.shift();
        const drum = drums[drumId];
        const cam = cameras.find(c => c.id === drum.cameraId);
        if (!cam) {
          // error
          continue;
        }
        forkliftAvailable--;
        scheduleEvent(time, 'dispatch_start', drumId);
        // Despacho terminará en time + dispatchMean
        const dispatchMean = parseFloat(dispatchMeanInput.value);
        const tDispatch = expo(dispatchMean);
        scheduleEvent(time + tDispatch, 'dispatch_end', drumId);
        log(`Hora ${time.toFixed(2)}: Inicio despacho tambor ${drumId} desde C${cam.id} (duración ~${tDispatch.toFixed(2)} h)`);
      }
    }

    // Eventos principales
    function processEvent(e) {
      const type = e.type;
      const t = e.time;
      currentTime = t;
      switch (type) {
        case 'arrival': {
          // Llegó un tambor
          const drumId = e.drumId;
          // Buscar cámara con espacio
          const cam = cameras.find(c => c.occupied < c.capacity);
          if (cam) {
            placeTamborInCamera(drumId, cam, t);
            // Programar próxima llegada
            const arrivalRate = parseFloat(arrivalRateInput.value) || 1;
            const inter = expo(1 / arrivalRate);
            scheduleEvent(t + inter, 'arrival', drumId + 1); // el siguiente tambor se crea cuando llega este, ID incrementa
          } else {
            // bloqueo: sin espacio ahora
            blocked.push(drumId);
            log(`Hora ${t.toFixed(2)}: Tamor ${drumId} bloqueado (sin espacio)`);
            // intentos periódicos para desbloquear
            scheduleEvent(t + 0.25, 'check_blocked');
          }
          break;
        }
        case 'storage_end': {
          // Tambor termina almacenamiento y está listo para despacho
          const drumId = e.drumId;
          const drum = drums[drumId];
          if (!drum) {
            break;
          }
          // Añadir a despacho si hay espacio
          readyQueue.push(drumId);
          log(`Hora ${t.toFixed(2)}: Tambor ${drumId} listo para despacho (C${drum.cameraId})`);
          // Intentar despacho
          attemptDispatch(t);
          // Intenta colocar desde bloqueados si hay espacio
          tryPlaceBlocked(t);
          break;
        }
        case 'dispatch_end': {
          // Despacho finalizado
          const drumId = e.drumId;
          const drum = drums[drumId];
          if (!drum) break;
          const cam = cameras.find(c => c.id === drum.cameraId);
          if (cam) {
            // Liberar espacio de la cámara
            updateCameraOccupancy(cam, cam.occupied - 1, t);
            log(`Hora ${t.toFixed(2)}: Despachado tambor ${drumId} desde C${cam.id} (ocupación=${cam.occupied})`);
          }
          totalDespachados++;
          // Calcular almacenamiento real
          const storageTime = (typeof drum.storageStart === 'number') ? (t - drum.storageStart) : 0;
          storageTimes.push(storageTime);
          // Limpiar tambor
          delete drums[drumId];
          // Liberar montacargas
          forkliftAvailable++;
          // Intentar nuevos despachos
          attemptDispatch(t);
          // Intentar desbloquear bloques si existían
          tryPlaceBlocked(t);
          break;
        }
        case 'check_blocked': {
          // Intentar colocar tambores bloqueados
          tryPlaceBlocked(t);
          break;
        }
        default:
          break;
      }
    }

    // Ejecutar simulación
    function runSimulation() {
      // Reiniciar estado
      logEl.textContent = '';
      currentTime = 0;
      drumCounter = 0;
      drums = {};
      blocked = [];
      readyQueue = [];
      storageTimes = [];
      totalDespachados = 0;

      // Inicializar cámaras (en caso de cambios en UI)
      CamerasInitFromUI();

      // Preparar eventos
      events = [];
      // Primer tambor llega de forma aleatoria
      const simTime = parseFloat(simTimeInput.value) || 24;
      const arrivalRate = parseFloat(arrivalRateInput.value) || 1;
      // Primer arrival
      // Generamos el primer tambor
      drumCounter = 1;
      // Crear primer tambor en llegada
      // Utilizamos events con tipo 'arrival'
      scheduleEvent(0 + expo(1 / arrivalRate), 'arrival', drumCounter);

      // Ejecutar bucle de eventos
      while (events.length > 0 && currentTime <= simTime) {
        const ev = events.shift();
        processEvent(ev);
      }

      // Si quedan eventos al terminar el tiempo simulado, los podemos ignorar o acumular hasta terminar
      // Cálculos de métricas
      const simTimeActual = Math.max(0, Math.min(simTime, currentTime));
      // Ocupación promedio por cámara
      const occupancyAverages = cameras.map(cam => {
        // En caso de terminar, acumulOccupancy debe contener el tiempo ponderado
        const totalTime = simTimeActual;
        const avgOcc = (simTimeActual > 0) ? cam.accumOccupancy / simTimeActual : 0;
        return { id: cam.id, avgOcc };
      });

      // Desplegar resultados
      despachadosEl.textContent = totalDespachados.toString();
      const avgStorage = (storageTimes.length > 0) ? (storageTimes.reduce((a,b)=>a+b,0)/storageTimes.length) : 0;
      avgStorageEl.textContent = avgStorage.toFixed(2);

      // Gráfico de ocupación
      renderOccupancyChart(occupancyAverages);

      // Resumen rápido
      summaryListEl.innerHTML = `
        <li>Despachos realizados: <strong>${totalDespachados}</strong></li>
        <li>Tiempo promedio de almacenamiento: <strong>${avgStorage.toFixed(2)}</strong> h</li>
        <li>Ocupación media por cámara (horas): ${occupancyAverages.map(o => `C${o.id}: ${o.avgOcc.toFixed(2)}`).join(', ')}</li>
      `;
    }

    // Construye cámaras desde la UI (si el usuario cambió capacidades)
    function CamerasInitFromUI() {
      // Si hay inputs especiales para cámaras, reconstruimos la lista
      // En este ejemplo, mantenemos 3 cámaras por defecto si no hay cambios.
      // Aquí podrías leer valores de inputs dinámicos si añadieras más cámaras.
      // Por simplicidad, asumimos 3 cámaras ya definidas en initConfiguration.
      // Reconfigurar lastTime y accumOccupancy para cálculo de ocupación
      cameras.forEach(cam => {
        if (typeof cam.lastTime !== 'number') cam.lastTime = 0;
        if (typeof cam.accumOccupancy !== 'number') cam.accumOccupancy = 0;
      });
    }

    function renderOccupancyChart(data) {
      const labels = data.map(d => `C${d.id}`);
      const values = data.map(d => d.avgOcc);

      if (!occChart) {
        occChart = new Chart(occChartCtx, {
          type: 'bar',
          data: {
            labels,
            datasets: [{
              label: 'Ocupación media (horas)',
              data: values,
              backgroundColor: 'rgba(76,201,240,0.6)'
            }]
          },
          options: {
            scales: {
              y: { beginAtZero: true, title: { display: true, text: 'Horas / hora (media)' } },
              x: { title: { display: true, text: 'Cámaras' } }
            },
            plugins: {
              legend: { display: false },
              title: { display: true, text: 'Ocupación media por cámara' }
            }
          }
        });
      } else {
        occChart.data.labels = labels;
        occChart.data.datasets[0].data = values;
        occChart.update();
      }
    }

    // Inicializar configuración y evento de botones
    function resetUI() {
      initConfiguration();
      // Reset gráfico y métricas
      despachadosEl.textContent = '0';
      avgStorageEl.textContent = '0';
      logEl.textContent = '';
      if (occChart) occChart.destroy(), occChart = null;
    }

    // Eventos de UI
    runBtn.addEventListener('click', () => {
      runSimulation();
    });

    resetBtn.addEventListener('click', () => {
      resetUI();
    });

    // Inicialización al cargar la página
    (function bootstrap() {
      initConfiguration();
      // Configuración base de 3 cámaras
      // Cada cámara ya tiene su capacidad cargada en initConfiguration
    })();

  </script>
</body>
</html>
