<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Стиралка v2.0 — Очередь и Мониторинг</title>
  <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
  <style>
    :root {
      --bg: #0f172a;
      --card-bg: #1e293b;
      --text: #f8fafc;
      --accent: #38bdf8;
      --success: #22c55e;
      --danger: #ef4444;
      --warning: #f59e0b;
    }

    * { box-sizing: border-box; margin: 0; padding: 0; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
    body { background-color: var(--bg); color: var(--text); padding: 15px; max-width: 600px; margin: 0 auto; }

    h1, h2 { text-align: center; margin-bottom: 15px; color: var(--accent); }
    .card { background: var(--card-bg); border-radius: 12px; padding: 15px; margin-bottom: 15px; border: 1px solid #334155; }

    /* Табы (Панели) */
    .tabs { display: flex; gap: 10px; margin-bottom: 15px; }
    .tab-btn { flex: 1; padding: 10px; border: none; background: #334155; color: #fff; border-radius: 8px; font-weight: bold; cursor: pointer; }
    .tab-btn.active { background: var(--accent); color: #000; }

    /* Форма ввода */
    input, select { width: 100%; padding: 12px; margin-bottom: 10px; border-radius: 8px; border: 1px solid #475569; background: #0f172a; color: #fff; font-size: 16px; }
    button.btn-main { width: 100%; padding: 12px; background: var(--accent); border: none; border-radius: 8px; font-weight: bold; color: #0f172a; font-size: 16px; cursor: pointer; }

    /* Машинки */
    .washers-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; margin-bottom: 15px; }
    .washer-card { background: #0f172a; border-radius: 8px; padding: 10px; text-align: center; border: 2px solid #334155; }
    .washer-card.free { border-color: var(--success); }
    .washer-card.busy { border-color: var(--danger); }
    .washer-card.out_of_order { border-color: var(--warning); opacity: 0.6; }
    .status-badge { font-size: 12px; font-weight: bold; text-transform: uppercase; margin-top: 5px; }

    /* Очередь */
    .queue-list { list-style: none; }
    .queue-item { display: flex; justify-content: space-between; align-items: center; padding: 10px; background: #0f172a; border-radius: 6px; margin-bottom: 8px; border-left: 4px solid var(--accent); }
    .queue-item span { font-size: 14px; }
    .queue-item .pos { font-weight: bold; color: var(--accent); }

    /* Скрытые разделы */
    .tab-content { display: none; }
    .tab-content.active { display: block; }

    /* Админка */
    .admin-controls { display: flex; gap: 5px; margin-top: 5px; }
    .admin-controls button { flex: 1; padding: 5px; font-size: 12px; border: none; border-radius: 4px; cursor: pointer; }
    .btn-free { background: var(--success); color: #fff; }
    .btn-block { background: var(--warning); color: #000; }
  </style>
</head>
<body>

  <h1>🧺 Стиралка v2.0</h1>

  <!-- Переключатель видов -->
  <div class="tabs">
    <button class="tab-btn active" onclick="switchTab('user')">Студент</button>
    <button class="tab-btn" onclick="switchTab('admin')">Админ</button>
  </div>

  <!-- Статус 3 стиральных машин -->
  <div class="card">
    <h2>Состояние машинок</h2>
    <div class="washers-grid" id="washers-container">
      <!-- Машинки загружаются из БД -->
    </div>
  </div>

  <!-- РАЗДЕЛ СТУДЕНТА -->
  <div id="tab-user" class="tab-content active">
    <!-- Регистрация / Запись в очередь -->
    <div class="card">
      <h2>Записаться в очередь</h2>
      <form id="queue-form">
        <input type="text" id="student-name" placeholder="Ваше Имя и Фамилия" required>
        <input type="text" id="student-id" placeholder="Номер комнаты / студенческого" required>
        <button type="submit" class="btn-main">Встать в общую очередь</button>
      </form>
    </div>

    <!-- Список общей очереди -->
    <div class="card">
      <h2>Общая очередь</h2>
      <ul class="queue-list" id="queue-container">
        <!-- Список очереди загружается из БД -->
      </ul>
    </div>
  </div>

  <!-- РАЗДЕЛ АДМИНА -->
  <div id="tab-admin" class="tab-content">
    <div class="card">
      <h2>Управление машинами</h2>
      <div id="admin-washers-container"></div>
    </div>
  </div>

  <script>
    // --- 1. НАСТРОЙКА SUPABASE ---
    // ЗАМЕНИТЕ ЭТИ ЗНАЧЕНИЯ НА ВАШИ ИЗ SUPABASE (Settings -> API)
    const SUPABASE_URL = 'ВАШ_SUPABASE_URL';
    const SUPABASE_KEY = 'ВАШ_SUPABASE_ANON_KEY';
    const db = supabase.createClient(SUPABASE_URL, SUPABASE_KEY);

    // --- 2. ПЕРЕКЛЮЧЕНИЕ ТАБОВ ---
    function switchTab(tab) {
      document.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
      document.querySelectorAll('.tab-content').forEach(c => c.classList.remove('active'));
      
      if(tab === 'user') {
        document.querySelectorAll('.tab-btn')[0].classList.add('active');
        document.getElementById('tab-user').classList.add('active');
      } else {
        document.querySelectorAll('.tab-btn')[1].classList.add('active');
        document.getElementById('tab-admin').classList.add('active');
      }
    }

    // --- 3. ЗАГРУЗКА И ОТОБРАЖЕНИЕ СОСТОЯНИЯ МАШИНОК ---
    async function loadWashers() {
      const { data: washers, error } = await db.from('washers').select('*').order('id');
      if (error) return console.error(error);

      const userContainer = document.getElementById('washers-container');
      const adminContainer = document.getElementById('admin-washers-container');
      
      userContainer.innerHTML = '';
      adminContainer.innerHTML = '';

      washers.forEach(w => {
        const isBusy = w.status === 'busy';
        const isOff = w.status === 'out_of_order';
        const statusText = isBusy ? 'Занята' : (isOff ? 'Ремонт' : 'Свободна');

        // Карточка для студента
        userContainer.innerHTML += `
          <div class="washer-card ${w.status}">
            <strong>№${w.id}</strong>
            <div class="status-badge">${statusText}</div>${isBusy ? `<small>${w.current_student_name || ''}</small>` : ''}
          </div>
        `;

        // Управление для админа
        adminContainer.innerHTML += `
          <div style="margin-bottom: 10px; background: #0f172a; padding: 10px; border-radius: 6px;">
            <strong>Машинка №${w.id}</strong> —${statusText}
            <div class="admin-controls">
              <button class="btn-free" onclick="updateWasherStatus(${w.id}, 'free')">Освободить</button>
              <button class="btn-block" onclick="updateWasherStatus(${w.id}, 'out_of_order')">На ремонт</button>
            </div>
          </div>
        `;
      });
    }

    // --- 4. ЗАГРУЗКА И ОТОБРАЖЕНИЕ ОЧЕРЕДИ ---
    async function loadQueue() {
      const { data: queue, error } = await db.from('queue').select('*').order('created_at', { ascending: true });
      if (error) return console.error(error);

      const container = document.getElementById('queue-container');
      container.innerHTML = '';

      if (queue.length === 0) {
        container.innerHTML = '<li style="text-align:center; color:#94a3b8;">Очередь пуста</li>';
        return;
      }

      queue.forEach((item, index) => {
        container.innerHTML += `
          <li class="queue-item">
            <div>
              <span class="pos">#${index + 1}</span> 
              <strong>${item.student_name}</strong> 
              <small>(${item.student_id})</small>
            </div>
            <button onclick="removeFromQueue(${item.id})" style="background:none; border:none; color:var(--danger); cursor:pointer;">❌</button>
          </li>
        `;
      });

      // Проверка для уведомлений (Если человек стал #1 в очереди)
      checkMyNotification(queue);
    }

    // --- 5. ДОБАВЛЕНИЕ В ОЧЕРЕДЬ ---
    document.getElementById('queue-form').addEventListener('submit', async (e) => {
      e.preventDefault();
      const name = document.getElementById('student-name').value;
      const studentId = document.getElementById('student-id').value;

      const { error } = await db.from('queue').insert([{ student_name: name, student_id: studentId }]);
      if (error) alert('Ошибка записи: ' + error.message);
      else {
        localStorage.setItem('my_student_id', studentId); // Сохраняем ID локально для уведомлений
        document.getElementById('queue-form').reset();
        // Запрашиваем разрешение на Push-уведомления
        if (Notification.permission !== "granted") Notification.requestPermission();
      }
    });

    // --- 6. УДАЛЕНИЕ ИЗ ОЧЕРЕДИ / УПРАВЛЕНИЕ ---
    async function removeFromQueue(id) {
      await db.from('queue').delete().eq('id', id);
    }

    async function updateWasherStatus(id, status) {
      await db.from('washers').update({ status: status, current_student_name: null }).eq('id', id);
    }

    // --- 7. УВЕДОМЛЕНИЯ В БРАУЗЕРЕ ---
    function checkMyNotification(queue) {
      const myId = localStorage.getItem('my_student_id');
      if (!myId || queue.length === 0) return;

      if (queue[0].student_id === myId) {
        if (Notification.permission === "granted") {
          new Notification("🧺 Стиралка свободна!", {
            body: "Ваша очередь подошла! Подойдите к прачечной.",
          });
        }
      }
    }

    // --- 8. REALTIME ПОДПИСКА (Обновление без перезагрузки) ---
    db.channel('realtime_washers').on('postgres_changes', { event: '*', schema: 'public', table: 'washers' }, loadWashers).subscribe();
    db.channel('realtime_queue').on('postgres_changes', { event: '*', schema: 'public', table: 'queue' }, loadQueue).subscribe();

    // Первоначальная загрузка
    loadWashers();
    loadQueue();
  </script>
</body>
</html>
