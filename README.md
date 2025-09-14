<?php
/* ============================================================ */
/*          index.php – один файл (регистрация, вход, UI, JS)   */
/* ============================================================ */

session_start();

/* ---------- Путь к файлу‑базе ---------------------------------- */
$loginFile = __DIR__ . '/password.txt';

/* ---------- Создать файл, если его нет -------------------------- */
if (!file_exists($loginFile)) {
    file_put_contents($loginFile, '');
    chmod($loginFile, 0600);               // только веб‑приложение
}

/* ---------- Процесс выхода ------------------------------------- */
if (isset($_GET['logout'])) {
    session_destroy();
    header('Location: ' . $_SERVER['PHP_SELF']);
    exit;
}

/* ---------- Утилиты --------------------------------------------- */
$messages = [];
$action   = $_POST['action'] ?? '';

/* ---------- Регистрация ------------------------------------------ */
if ($action === 'register') {
    $user = trim($_POST['regLogin'] ?? '');
    $pwd  = $_POST['regPwd'] ?? '';
    if ($user === '' || $pwd === '') {
        $messages[] = 'Заполните оба поля';
    } else {
        $lines = file($loginFile, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES) ?: [];
        foreach ($lines as $ln) {
            [$old, ] = explode('|', $ln, 2);
            if ($old === $user) { $messages[] = 'Логин занят'; break; }
        }
        if (empty($messages)) {
            $hash = password_hash($pwd, PASSWORD_BCRYPT);
            file_put_contents($loginFile, "$user|$hash\n", FILE_APPEND | LOCK_EX);
            $_SESSION['login'] = $user;           // сразу войти
            header('Location: ' . $_SERVER['PHP_SELF']);
            exit;
        }
    }
}

/* ---------- Вход ----------------------------------------------- */
if ($action === 'login') {
    $user = trim($_POST['logLogin'] ?? '');
    $pwd  = $_POST['logPwd'] ?? '';
    if ($user === '' || $pwd === '') {
        $messages[] = 'Заполните оба поля';
    } else {
        $lines = file($loginFile, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES) ?: [];
        $found = false;
        foreach ($lines as $ln) {
            [$old, $hash] = explode('|', $ln, 2);
            if ($old === $user && password_verify($pwd, $hash)) { $found = true; break; }
        }
        if ($found) {
            $_SESSION['login'] = $user;
            header('Location: ' . $_SERVER['PHP_SELF']);
            exit;
        } else {
            $messages[] = 'Неверный логин / пароль';
        }
    }
}

/* ---------- Пользователь вошёл ли? ----------------------------- */
$userLogged = isset($_SESSION['login']);
$userName   = $userLogged ? $_SESSION['login'] : '';
?>
<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8"/>
<title>BedPulse – Регистрация и вход (через PHP)</title>
<style>
/* ---------- БАЗА ---------- */
*{box-sizing:border-box;}
html,body{margin:0;padding:0;font-family:'Segoe UI',Tahoma,Arial,Helvetica,sans-serif;
  background:#000;color:white;overflow:hidden;height:100vh;}
#header{width:100%;height:200px;background:#000;color:#fff;font-weight:800;font-size:5rem;
  line-height:200px;text-align:center;letter-spacing:4px;text-transform:uppercase;}
#background{position:fixed;top:0;left:0;width:110vw;height:110vh;background:
  radial-gradient(circle at center,#003a3a55 0%,transparent 70%),
  repeating-radial-gradient(circle,#00ffaa33 5%,transparent 15%);
  background-size:200% 200%;transition:transform .1s ease-out;pointer-events:none;z-index:-1;}
#main-content{position:relative;width:100%;height:100vh;display:flex;justify-content:center;
  align-items:center;flex-direction:column;}
#connectBtn{cursor:pointer;padding:15px 50px;font-size:1.5rem;font-weight:700;border-radius:10px;border:none;
  background:linear-gradient(135deg,#00ffff,#0080ff);color:#000;
  box-shadow:0 0 20px #00ffff,inset 0 0 10px #0080ff;transition:background .3s,color .3s;}
#connectBtn:hover{background:linear-gradient(135deg,#0080ff,#00ffff);color:white;
  box-shadow:0 0 35px #00ffff,inset 0 0 25px #0080ff;}
#connectBtn.clicked{transform:translateY(200px) scale(.8);opacity:0;
  transition:transform .4s ease,opacity .4s ease;pointer-events:none;}
#bottomRightPanel{position:fixed;bottom:15px;right:15px;display:flex;flex-direction:column;gap:10px;z-index:15;}
#bottomRightPanel button{background:#222;border:none;border-radius:8px;color:#00ffff;font-weight:600;
  padding:12px 18px;cursor:pointer;font-size:1rem;box-shadow:0 0 5px #00ffff;
  transition:background .25s,box-shadow .25s;min-width:140px;text-align:center;}
#bottomRightPanel button:hover{background:#333;box-shadow:0 0 12px #00ffff;}
#mobileToggle{position:fixed;bottom:15px;left:15px;width:40px;height:40px;
  background:#222;color:#fff;border:none;border-radius:6px;font-size:1rem;
  display:flex;align-items:center;justify-content:center;cursor:pointer;z-index:20;
  transition:background .2s;}
#mobileToggle:hover{background:#333;}
body.mobile #bottomRightPanel{bottom:15px;left:15px;right:auto;}
body.mobile #bottomRightPanel button{width:100%;border-radius:0;}
#popupMessage{position:fixed;bottom:50px;right:20px;background:#008080dd;color:white;
  padding:10px 20px;border-radius:8px;box-shadow:0 0 15px #00ffffdd;
  opacity:0;pointer-events:none;transition:opacity .3s ease;}
#popupMessage.show{opacity:1;pointer-events:auto;}
#connectInfo{position:fixed;top:50%;left:50%;transform:translate(-50%,-50%) scale(0);
  opacity:0;transition:transform .4s ease,opacity .4s ease;z-index:1200;
  background:rgba(0,0,30,.95);backdrop-filter:blur(10px);
  color:#00ffff;padding:20px 25px;border-radius:10px;text-align:center;width:320px;max-width:90vw;}
#connectInfo.open{transform:translate(-50%,-50%) scale(1);opacity:1;}
#connectInfo button.closeBtn{position:absolute;top:5px;right:5px;background:transparent;border:none;
  font-size:1.5rem;color:#00ffff;cursor:pointer;user-select:none;}
.modal{position:fixed;top:50%;left:50%;transform:translate(-50%,-50%);
  background:#111;border-radius:12px;box-shadow:0 0 20px #008080bb;
  width:320px;max-width:90vw;padding:25px 25px 50px 25px;
  z-index:1100;color:#ccc;display:none;user-select:text;}
.modal.open{display:block;}
.modal h3{margin-top:0;font-weight:600;font-size:1.3rem;text-align:center;margin-bottom:15px;color:#00cccc;}
.modal label{font-size:.85rem;color:#aaa;display:block;margin-bottom:6px;}
.modal input[type=email],
.modal input[type=password],
.modal input[type=text]{width:100%;padding:8px 10px;background:#222;border:1px solid #444;
  border-radius:6px;font-size:1rem;color:#ddd;margin-bottom:18px;outline-offset:2px;outline-color:#008080;}
.modal input:focus{border-color:#00cccc;outline:2px solid #00cccc55;}
.modal button.submitBtn{width:100%;padding:10px 0;background:linear-gradient(135deg,#00cccc,#006666);
  border:none;border-radius:8px;color:#001010;font-weight:700;font-size:1rem;
  box-shadow:0 0 12px #00ccccbb;transition:background .3s;}
.modal button.submitBtn:hover{background:linear-gradient(135deg,#00ffff,#009999);color:#000;}
.modal .bottom-right{position:absolute;bottom:12px;right:18px;font-size:.75rem;color:#555;user-select:none;}
.modal .bottom-right button{background:none;border:none;color:#555;font-weight:600;font-size:.75rem;
  padding:0 6px;cursor:pointer;text-decoration:underline;user-select:none;}
.modal .bottom-right button:hover{color:#00cccc;}
.modal .closeBtn{position:absolute;top:10px;right:14px;background:transparent;border:none;font-size:1.5rem;
  color:#008080;cursor:pointer;user-select:none;line-height:1;}
.modal .closeBtn:hover{color:#00cccc;}
#quickMenu{position:fixed;top:10px;left:10px;background:#222;color:#fff;
  padding:10px 15px;border-radius:4px;display:none;z-index:200;}
#quickMenu a{color:#00ffff;margin-left:12px;text-decoration:none;}
</style>
</head>
<body>
<!-- Марка – скрыто, пока пользователь не вошёл -->
<div id="quickMenu"><span id="greeting"><?php echo htmlspecialchars($userName); ?></span> <a href="?logout=1">Выход</a></div>

<header id="header">BedPulse</header>
<div id="background"></div>

<div id="main-content"><button id="connectBtn">Подключиться</button></div>

<div id="bottomRightPanel"><button id="registerBtn">Зарегистрироваться</button>
  <button id="supportBtn">Поддержка</button></div>

<button id="mobileToggle" title="Переключить режим">M</button>

<div id="popupMessage"></div>

<div id="connectInfo" role="dialog" aria-modal="true" aria-labelledby="connectInfoTitle" aria-hidden="true">
  <button class="closeBtn" aria-label="Закрыть">&times;</button>
  <h3 id="connectInfoTitle">Приятной игры</h3>
  <p id="ipAddress">IP: 192.168.1.100</p>
</div>

<div id="supportModal" class="modal" role="dialog" aria-modal="true" aria-labelledby="supportTitle" aria-hidden="true">
  <button class="closeBtn" aria-label="Закрыть">&times;</button>
  <h3 id="supportTitle">Контактная информация поддержки</h3>
  <div class="content" style="margin-top:20px;">
    <p>Электронная почта: <a href="mailto:support@bedpulse.com" style="color:#00cccc;text-decoration:none;">support@bedpulse.com</a></p>
    <p>Телефон: <a href="tel:+71234567890" style="color:#00cccc;text-decoration:none;">+7 123 456 78 90</a></p>
    <p>Telegram: <a href="https://t.me/bedpulse" target="_blank" style="color:#00cccc;text-decoration:none;">@bedpulse</a></p>
  </div>
</div>

<div id="loginModal" class="modal" role="dialog" aria-modal="true" aria-labelledby="loginTitle" aria-hidden="true">
  <button class="closeBtn" aria-label="Закрыть">&times;</button>
  <h3 id="loginTitle">Вход в аккаунт</h3>
  <form id="loginForm" method="post">
    <input type="hidden" name="action" value="login">
    <label for="logLogin">Логин</label>
    <input id="logLogin" name="logLogin" type="text" placeholder="example@mail.ru" required autocomplete="username"/>
    <label for="logPwd">Пароль</label>
    <input id="logPwd" name="logPwd" type="password" placeholder="******" required autocomplete="current-password"/>
    <button type="submit" class="submitBtn">Войти</button>
  </form>
  <div class="bottom-right">
    Если нет аккаунта <button id="toRegFromLogin">Зарегистрироваться</button>
  </div>
</div>

<div id="registerModal" class="modal" role="dialog" aria-modal="true" aria-labelledby="registerTitle" aria-hidden="true">
  <button class="closeBtn" aria-label="Закрыть">&times;</button>
  <h3 id="registerTitle">Регистрация</h3>
  <form id="registerForm" method="post">
    <input type="hidden" name="action" value="register">
    <label for="regLogin">Логин</label>
    <input id="regLogin" name="regLogin" type="text" placeholder="example@mail.ru" required autocomplete="username"/>
    <label for="regPwd">Пароль</label>
    <input id="regPwd" name="regPwd" type="password" placeholder="******" required autocomplete="new-password"/>
    <button type="submit" class="submitBtn">Зарегистрироваться</button>
  </form>
  <div class="bottom-right">
    Если есть аккаунт <button id="toLoginFromRegister">Войти</button>
  </div>
</div>

<script>
(() => {
  /* ---------- Переходы к модалам ---------- */
  const supportBtn   = document.getElementById('supportBtn');
  const registerBtn  = document.getElementById('registerBtn');
  const supportModal = document.getElementById('supportModal');
  const loginModal   = document.getElementById('loginModal');
  const registerModal= document.getElementById('registerModal');
  const modals = [supportModal, loginModal, registerModal];

  const open  = m => { if (m) { m.classList.add('open'); m.setAttribute('aria-hidden', 'false'); } };
  const close = m => { if (m) { m.classList.remove('open'); m.setAttribute('aria-hidden', 'true'); } };
  const closeAll = () => modals.forEach(close);

  supportBtn.addEventListener('click', () => open(supportModal));
  registerBtn.addEventListener('click', () => open(registerModal));
  document.getElementById('toLoginFromRegister').addEventListener('click', () => { closeAll(); open(loginModal); });
  document.getElementById('toRegFromLogin').addEventListener('click', () => { closeAll(); open(registerModal); });

  modals.forEach(m => {
    if (!m) return;
    const btn = m.querySelector('.closeBtn');
    if (btn) btn.addEventListener('click', closeAll);
    m.addEventListener('click', e => { if (e.target === m) closeAll(); });
  });

  /* ---------- Подключиться (IP‑панель) ---------- */
  const connectBtn  = document.getElementById('connectBtn');
  const connectInfo = document.getElementById('connectInfo');
  connectBtn.addEventListener('click', () => {
    connectBtn.classList.add('clicked');
    setTimeout(() => connectInfo.classList.add('open'), 300);
  });
  connectInfo.querySelector('.closeBtn').addEventListener('click', () => {
    connectInfo.classList.remove('open');
    connectBtn.classList.remove('clicked');
  });

  /* ---------- Копировать IP ---------- */
  document.getElementById('ipAddress').addEventListener('click', () => {
    const ip = document.getElementById('ipAddress').textContent.replace('IP: ', '');
    navigator.clipboard.writeText(ip).then(() => showPopup('IP скопирован!'));
  });

  /* ---------- Сообщения ---------- */
  const popup = document.getElementById('popupMessage');
  function showPopup(msg) {
    popup.textContent = msg;
    popup.classList.add('show');
    clearTimeout(popup.timeout);
    popup.timeout = setTimeout(() => popup.classList.remove('show'), 3000);
  }

  /* ---------- Медиа‑режим (дроби‑смещение) ---------- */
  const mobileToggle = document.getElementById('mobileToggle');
  mobileToggle.addEventListener('click', () => {
    document.body.classList.toggle('mobile');
    mobileToggle.textContent = document.body.classList.contains('mobile') ? '←' : 'M';
  });

  /* ---------- Показ/скрытие меню после входа ---------- */
  <?php if ($userLogged): ?> document.getElementById('quickMenu').style.display = 'block'; <?php endif; ?>
})();
</script>
</body>
</html>
