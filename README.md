const STORAGE_KEY = 'last_patron_match_v1';
const SETTINGS_KEY = 'last_patron_settings_v1';

const INITIAL_SETTINGS = {
  sound: true,
  difficulty: 'medium',
  botCount: 1,
  viewMode: 'first'
};

const ITEM_LABELS = {
  scope: 'Лупа',
  medkit: 'Аптечка',
  cuffs: 'Наручники',
  knife: 'Ножівка',
  discard: 'Викидання набою'
};

const state = {
  settings: loadSettings(),
  match: null,
  audioContext: null,
  modalStack: []
};

const ui = {
  menuScreen: document.getElementById('menuScreen'),
  gameScreen: document.getElementById('gameScreen'),
  firstPersonView: document.getElementById('firstPersonView'),
  tacticalView: document.getElementById('tacticalView'),
  opponentsBank: document.getElementById('opponentsBank'),
  playersPanel: document.getElementById('playersPanel'),
  ammoCount: document.getElementById('ammoCount'),
  liveCount: document.getElementById('liveCount'),
  blankCount: document.getElementById('blankCount'),
  activePlayerCard: document.getElementById('activePlayerCard'),
  inventoryPanel: document.getElementById('inventoryPanel'),
  eventLog: document.getElementById('eventLog'),
  targetBar: document.getElementById('targetBar'),
  reloadBtn: document.getElementById('reloadBtn'),
  shootBtn: document.getElementById('shootBtn'),
  useItemBtn: document.getElementById('useItemBtn'),
  rulesModal: document.getElementById('rulesModal'),
  settingsModal: document.getElementById('settingsModal'),
  matchTitle: document.getElementById('matchTitle'),
  viewToggleBtn: document.getElementById('viewToggleBtn'),
  continueBtn: document.getElementById('continueBtn'),
  soundToggle: document.getElementById('soundToggle'),
  difficultySelect: document.getElementById('difficultySelect'),
  botCountSelect: document.getElementById('botCountSelect'),
  viewModeSelect: document.getElementById('viewModeSelect')
};

const MODAL_OPENERS = {
  rules: ui.rulesModal,
  settings: ui.settingsModal
};

function loadSettings() {
  try {
    const raw = JSON.parse(localStorage.getItem(SETTINGS_KEY));
    return { ...INITIAL_SETTINGS, ...(raw || {}) };
  } catch (error) {
    return { ...INITIAL_SETTINGS };
  }
}

function saveSettings() {
  localStorage.setItem(SETTINGS_KEY, JSON.stringify(state.settings));
}

function ensureAudio() {
  if (!state.settings.sound) return;
  if (!state.audioContext) {
    const AudioCtx = window.AudioContext || window.webkitAudioContext;
    if (!AudioCtx) return;
    state.audioContext = new AudioCtx();
  }
  if (state.audioContext.state === 'suspended') {
    state.audioContext.resume();
  }
}

function playTone(frequency, duration, type = 'square', volume = 0.04) {
  if (!state.settings.sound || !state.audioContext) return;
  const oscillator = state.audioContext.createOscillator();
  const gain = state.audioContext.createGain();
  oscillator.type = type;
  oscillator.frequency.value = frequency;
  gain.gain.value = volume;
  oscillator.connect(gain);
  gain.connect(state.audioContext.destination);
  oscillator.start();
  oscillator.stop(state.audioContext.currentTime + duration);
  gain.gain.exponentialRampToValueAtTime(0.0001, state.audioContext.currentTime + duration);
}

function emitSound(name) {
  if (!state.settings.sound) return;
  switch (name) {
    case 'shot':
      playTone(180, 0.09, 'sawtooth', 0.045);
      setTimeout(() => playTone(110, 0.12, 'triangle', 0.03), 50);
      break;
    case 'reload':
      playTone(420, 0.13, 'square', 0.03);
      break;
    case 'button':
      playTone(620, 0.06, 'triangle', 0.02);
      break;
    default:
      break;
  }
}

function safePushLog(message) {
  if (!state.match) return;
  if (!state.match.log.length) {
    state.match.log.unshift({ message, timestamp: Date.now() });
    return;
  }
  state.match.log.unshift({ message, timestamp: Date.now() });
  if (state.match.log.length > 18) {
    state.match.log = state.match.log.slice(0, 18);
  }
}

function buildDeck() {
  const deck = [];
  const liveCount = 7;
  const blankCount = 7;
  for (let i = 0; i < liveCount; i += 1) deck.push('live');
  for (let i = 0; i < blankCount; i += 1) deck.push('blank');
  for (let i = deck.length - 1; i > 0; i -= 1) {
    const j = Math.floor(Math.random() * (i + 1));
    [deck[i], deck[j]] = [deck[j], deck[i]];
  }
  return deck;
}

function createPlayer(id, name, isHuman = false) {
  return {
    id,
    name,
    isHuman,
    maxHealth: 5,
    health: 5,
    isAlive: true,
    items: {
      scope: 1,
      medkit: 1,
      cuffs: 1,
      knife: 1,
      discard: 1
    },
    effects: {
      skipNextTurn: 0,
      bonusDamage: 0,
      knownNextCard: null
    }
  };
}

function getPlayers() {
  return state.match ? state.match.players : [];
}

function getCurrentPlayer() {
  if (!state.match || !state.match.players.length) return null;
  return state.match.players[state.match.currentIndex];
}

function getAlivePlayers() {
  return getPlayers().filter((player) => player.isAlive);
}

function getLivingOpponents(playerId) {
  return getPlayers().filter((player) => player.id !== playerId && player.isAlive);
}

function chooseDefaultTarget(playerId) {
  const options = getLivingOpponents(playerId);
  if (!options.length) return null;
  return options[0].id;
}

function calculateDeckStats() {
  const deck = state.match.deck || [];
  return {
    live: deck.filter((card) => card === 'live').length,
    blank: deck.filter((card) => card === 'blank').length,
    total: deck.length
  };
}

function canUseItem(player, itemKey) {
  if (!player || !player.isAlive) return false;
  if (itemKey === 'medkit') {
    return player.items.medkit > 0 && player.health < player.maxHealth;
  }
  if (itemKey === 'scope') {
    return player.items.scope > 0 && state.match.deck.length > 0;
  }
  if (itemKey === 'cuffs') {
    return player.items.cuffs > 0 && getLivingOpponents(player.id).length > 0;
  }
  if (itemKey === 'knife') {
    return player.items.knife > 0;
  }
  if (itemKey === 'discard') {
    return player.items.discard > 0 && state.match.deck.length > 0;
  }
  return false;
}

function getTargetLabel(player) {
  return player ? `${player.name} (${player.health}/${player.maxHealth})` : '—';
}

function setViewMode(mode) {
  state.settings.viewMode = mode;
  saveSettings();
  state.match.viewMode = mode;
  ui.firstPersonView.classList.toggle('active', mode === 'first');
  ui.tacticalView.classList.toggle('active', mode === 'tactical');
  ui.firstPersonView.classList.toggle('hidden', mode !== 'first');
  ui.tacticalView.classList.toggle('hidden', mode !== 'tactical');
  ui.viewToggleBtn.textContent = mode === 'first' ? 'Тактичний вид' : 'Вид від першої особи';
}

function refreshSettingsControls() {
  ui.soundToggle.checked = state.settings.sound;
  ui.difficultySelect.value = state.settings.difficulty;
  ui.botCountSelect.value = String(state.settings.botCount);
  ui.viewModeSelect.value = state.settings.viewMode;
}

function openModal(name) {
  const modal = MODAL_OPENERS[name];
  if (!modal) return;
  modal.classList.remove('hidden');
  modal.setAttribute('aria-hidden', 'false');
  state.modalStack.push(name);
}

function closeModal(name) {
  const modal = MODAL_OPENERS[name];
  if (!modal) return;
  modal.classList.add('hidden');
  modal.setAttribute('aria-hidden', 'true');
  state.modalStack = state.modalStack.filter((item) => item !== name);
}

function closeAllModals() {
  Object.keys(MODAL_OPENERS).forEach((name) => closeModal(name));
}

function loadSavedMatch() {
  try {
    const raw = JSON.parse(localStorage.getItem(STORAGE_KEY));
    if (!raw || !raw.players || !raw.players.length) return null;
    raw.players = raw.players.map((player) => ({
      ...player,
      items: { ...player.items },
      effects: { ...player.effects }
    }));
    raw.log = Array.isArray(raw.log) ? raw.log : [];
    raw.viewMode = raw.viewMode || state.settings.viewMode;
    raw.difficulty = raw.difficulty || state.settings.difficulty;
    state.match = raw;
    return raw;
  } catch (error) {
    return null;
  }
}

function saveMatch() {
  if (!state.match) return;
  localStorage.setItem(STORAGE_KEY, JSON.stringify(state.match));
}

function generateMatch() {
  const human = createPlayer('human', 'Гравець', true);
  const bots = Array.from({ length: state.settings.botCount }, (_, index) => createPlayer(`bot-${index + 1}`, `Бот ${index + 1}`));
  const players = [human, ...bots];
  const deck = buildDeck();

  state.match = {
    players,
    currentIndex: 0,
    deck,
    log: [],
    difficulty: state.settings.difficulty,
    viewMode: state.settings.viewMode,
    winner: null,
    gameOver: false,
    turnLocked: false,
    selectedTargetId: chooseDefaultTarget('human'),
    repeatShotAllowed: false,
    autoReloadNeeded: false
  };

  safePushLog('Новий матч розпочато. Складність: ' + (state.settings.difficulty === 'easy' ? 'Легка' : state.settings.difficulty === 'medium' ? 'Середня' : 'Важка'));
  safePushLog('Колода сформована. Кількість набоїв у серії: ' + deck.length);
  if (state.match.players.length > 1) {
    state.match.selectedTargetId = chooseDefaultTarget('human');
  }
  startTurn();
  render();
  saveMatch();
}

function restartMatch() {
  if (!state.match) {
    generateMatch();
    return;
  }
  if (state.match.gameOver) {
    generateMatch();
    return;
  }
  const oldSettings = { ...state.settings };
  generateMatch();
  state.settings = oldSettings;
  saveSettings();
}

function battleEnd(winnerName) {
  state.match.gameOver = true;
  state.match.winner = winnerName;
  safePushLog(`Матч завершено. Переможець: ${winnerName}`);
  render();
  saveMatch();
}

function advanceTurn() {
  if (!state.match || state.match.gameOver) return;

  state.match.turnLocked = false;
  state.match.repeatShotAllowed = false;

  let nextIndex = state.match.currentIndex;
  let attempts = 0;

  do {
    nextIndex = (nextIndex + 1) % state.match.players.length;
    attempts += 1;
    if (attempts > state.match.players.length * 2) break;
  } while (!state.match.players[nextIndex].isAlive);

  state.match.currentIndex = nextIndex;
  startTurn();
}

function startTurn() {
  if (!state.match || state.match.gameOver) return;
  const active = getCurrentPlayer();
  if (!active || !active.isAlive) {
    advanceTurn();
    return;
  }

  if (active.effects.skipNextTurn > 0) {
    const skipped = active.effects.skipNextTurn;
    active.effects.skipNextTurn = 0;
    safePushLog(`${active.name} пропускає хід (${skipped}).`);
    state.match.turnLocked = false;
    state.match.repeatShotAllowed = false;
    setTimeout(() => advanceTurn(), 600);
    render();
    saveMatch();
    return;
  }

  state.match.selectedTargetId = chooseDefaultTarget(active.id) || active.id;
  state.match.turnLocked = false;
  state.match.repeatShotAllowed = false;

  if (!active.isHuman) {
    setTimeout(() => botTurn(), 700);
  }

  render();
  saveMatch();
}

function getWeaponTypeLabel(card) {
  if (card === 'live') return 'Бойовий';
  if (card === 'blank') return 'Холостий';
  return 'Невідомо';
}

function isHumanTurn() {
  return !!(state.match && getCurrentPlayer() && getCurrentPlayer().isHuman);
}

function handleAction(actionType, payload = {}) {
  if (!state.match || state.match.gameOver || state.match.turnLocked) return;
  const current = getCurrentPlayer();
  if (!current || !current.isAlive) return;

  if (actionType === 'reload') {
    reloadDeck(current);
    return;
  }

  if (actionType === 'selectTarget') {
    state.match.selectedTargetId = payload.targetId;
    render();
    saveMatch();
    return;
  }

  if (actionType === 'fire') {
    fireAtTarget(payload.targetId || state.match.selectedTargetId);
    return;
  }

  if (actionType === 'item') {
    useItem(current, payload.itemKey, payload.targetId || null);
    return;
  }
}

function handleKeydown(event) {
  if (event.target && ['INPUT', 'TEXTAREA', 'SELECT'].includes(event.target.tagName)) {
    return;
  }

  if (event.key === 'Escape') {
    closeAllModals();
    return;
  }

  if (event.key.toLowerCase() === 'm') {
    toggleSound();
    return;
  }

  if (event.key.toLowerCase() === 'r') {
    if (isHumanTurn()) {
      const active = getCurrentPlayer();
      if (state.match.deck.length === 0) {
        reloadDeck(active);
      }
    }
    return;
  }

  if (event.key === '1') {
    const current = getCurrentPlayer();
    if (!current || !current.isHuman) return;
    const targets = getLivingOpponents(current.id);
    if (targets.length) {
      state.match.selectedTargetId = targets[0].id;
      render();
    }
  }
}

function toggleSound() {
  state.settings.sound = !state.settings.sound;
  saveSettings();
  refreshSettingsControls();
  if (state.settings.sound) {
    ensureAudio();
    emitSound('button');
  }
}

function reloadDeck(player) {
  if (!state.match || state.match.gameOver || !player || !player.isAlive) return;
  if (state.match.deck.length > 0 && !state.match.autoReloadNeeded) {
    return;
  }
  const newDeck = buildDeck();
  state.match.deck = newDeck;
  state.match.autoReloadNeeded = false;
  safePushLog(`${player.name} перезаряджає та формує нову серію.`);
  emitSound('reload');
  render();
  saveMatch();
}

function useItem(player, itemKey, targetId = null) {
  if (!state.match || state.match.gameOver || !player || !player.isAlive) return;
  if (state.match.turnLocked) return;
  if (!canUseItem(player, itemKey)) return;

  if (itemKey === 'scope') {
    if (!state.match.deck.length) return;
    const nextCard = state.match.deck[0];
    player.effects.knownNextCard = nextCard;
    player.items.scope -= 1;
    safePushLog(`${player.name} використовує Лупу: наступний набій — ${getWeaponTypeLabel(nextCard).toLowerCase()}.`);
    emitSound('button');
    render();
    saveMatch();
    return;
  }

  if (itemKey === 'medkit') {
    player.items.medkit -= 1;
    const restored = Math.min(player.maxHealth, player.health + 1);
    const delta = restored - player.health;
    player.health = restored;
    safePushLog(`${player.name} використовує Аптечку і відновлює ${delta} здоров'я.`);
    emitSound('button');
    render();
    saveMatch();
    return;
  }

  if (itemKey === 'cuffs') {
    const target = getPlayers().find((candidate) => candidate.id === targetId && candidate.isAlive && candidate.id !== player.id);
    if (!target) return;
    player.items.cuffs -= 1;
    target.effects.skipNextTurn += 1;
    safePushLog(`${player.name} використовує Наручники на ${target.name}. Наступний хід ${target.name} буде пропущений.`);
    emitSound('button');
    render();
    saveMatch();
    return;
  }

  if (itemKey === 'knife') {
    if (player.effects.bonusDamage > 0) return;
    player.items.knife -= 1;
    player.effects.bonusDamage = 1;
    safePushLog(`${player.name} підготовлює Ножівку. Наступний бойовий постріл завдасть +1 шкоди.`);
    emitSound('button');
    render();
    saveMatch();
    return;
  }

  if (itemKey === 'discard') {
    if (!state.match.deck.length) return;
    player.items.discard -= 1;
    state.match.deck.shift();
    safePushLog(`${player.name} викидає наступний набій без показу типу.`);
    emitSound('button');
    render();
    saveMatch();
    return;
  }
}

function fireAtTarget(targetId) {
  if (!state.match || state.match.gameOver || state.match.turnLocked) return;
  const shooter = getCurrentPlayer();
  if (!shooter || !shooter.isAlive) return;
  if (!targetId) {
    state.match.selectedTargetId = chooseDefaultTarget(shooter.id);
    targetId = state.match.selectedTargetId;
  }
  const target = getPlayers().find((player) => player.id === targetId && player.isAlive);
  if (!target || target.id === shooter.id && getLivingOpponents(shooter.id).length === 0) {
    return;
  }

  if (state.match.deck.length === 0) {
    if (shooter.isHuman) {
      safePushLog('Колода порожня. Треба перезарядити перед пострілом.');
    } else {
      reloadDeck(shooter);
    }
    render();
    return;
  }

  state.match.turnLocked = true;
  const drawn = state.match.deck.shift();
  const selfShot = target.id === shooter.id;
  const isLive = drawn === 'live';
  const damageBonus = shooter.effects.bonusDamage > 0 ? 1 : 0;
  const baseDamage = isLive ? 1 + damageBonus : 0;

  if (shooter.effects.bonusDamage > 0) {
    shooter.effects.bonusDamage = 0;
  }

  if (isLive) {
    if (selfShot) {
      shooter.health = Math.max(0, shooter.health - baseDamage);
      shooter.isAlive = shooter.health > 0;
      safePushLog(`${shooter.name} пострілює себе і отримує ${baseDamage} шкоди.`);
      if (!shooter.isAlive) {
        safePushLog(`${shooter.name} вибув.`);
      }
    } else {
      target.health = Math.max(0, target.health - baseDamage);
      target.isAlive = target.health > 0;
      safePushLog(`${shooter.name} вистрілює в ${target.name} і завдає ${baseDamage} шкоди.`);
      if (!target.isAlive) {
        safePushLog(`${target.name} вибув.`);
      }
    }
    emitSound('shot');

    if (checkWinCondition()) {
      state.match.turnLocked = false;
      render();
      return;
    }

    if (selfShot) {
      state.match.repeatShotAllowed = true;
      state.match.turnLocked = false;
      safePushLog('Холостий постріл в себе дає можливість вистрілити ще раз у поточному ході.');
      render();
      saveMatch();
      return;
    }

    state.match.turnLocked = false;
    advanceTurn();
    render();
    saveMatch();
    return;
  }

  if (selfShot) {
    safePushLog(`${shooter.name} робить холостий постріл у себе і отримує другу можливість.`);
    emitSound('button');
    state.match.repeatShotAllowed = true;
    state.match.turnLocked = false;
    render();
    saveMatch();
    return;
  }

  safePushLog(`${shooter.name} робить холостий постріл у ${target.name}. Хід завершується.`);
  state.match.turnLocked = false;
  advanceTurn();
  render();
  saveMatch();
}

function checkWinCondition() {
  const alive = getAlivePlayers();
  if (alive.length <= 1) {
    const winner = alive[0] || getPlayers().find((player) => player.isAlive);
    if (winner) {
      battleEnd(winner.name);
      return true;
    }
  }
  return false;
}

function render() {
  if (!state.match) return;
  ui.matchTitle.textContent = 'ОСТАННІЙ ПАТРОН';
  const deck = calculateDeckStats();
  ui.ammoCount.textContent = String(deck.total);
  ui.liveCount.textContent = String(deck.live);
  ui.blankCount.textContent = String(deck.blank);
  ui.viewToggleBtn.textContent = state.match.viewMode === 'first' ? 'Тактичний вид' : 'Вид від першої особи';

  renderTargets();
  renderInventory();
  renderLog();
  renderPlayers();
  renderActiveCard();
  renderFirstPerson();
  setViewMode(state.match.viewMode || state.settings.viewMode);

  const current = getCurrentPlayer();
  const canReload = !!(current && current.isAlive && state.match.deck.length === 0 && !state.match.gameOver);
  ui.reloadBtn.disabled = !(canReload || (current && current.isAlive && state.match.deck.length === 0));
  ui.shootBtn.disabled = !current || !current.isAlive || !state.match || state.match.gameOver || (current.isHuman ? false : true);
  ui.useItemBtn.disabled = !current || !current.isAlive || !state.match || state.match.gameOver || (current.isHuman ? false : true);

  const continueHidden = !localStorage.getItem(STORAGE_KEY);
  ui.continueBtn.style.display = continueHidden ? 'none' : 'inline-flex';
}

function renderTargets() {
  if (!state.match) return;
  const current = getCurrentPlayer();
  if (!current) return;

  const options = getLivingOpponents(current.id);
  ui.targetBar.innerHTML = '';

  options.forEach((player) => {
    const button = document.createElement('button');
    button.type = 'button';
    button.className = 'target-option';
    if (state.match.selectedTargetId === player.id) button.classList.add('selected');
    button.innerHTML = `<span>${player.name}</span><span>${player.health}/${player.maxHealth}</span>`;
    button.disabled = !current.isHuman || !state.match || state.match.gameOver;
    button.addEventListener('click', () => {
      handleAction('selectTarget', { targetId: player.id });
    });
    ui.targetBar.appendChild(button);
  });

  if (!options.length) {
    const empty = document.createElement('div');
    empty.className = 'target-option inactive';
    empty.textContent = 'Немає живих суперників';
    ui.targetBar.appendChild(empty);
  }
}

function renderInventory() {
  if (!state.match) return;
  const current = getCurrentPlayer();
  if (!current) return;

  const items = Object.entries(ITEM_LABELS);
  ui.inventoryPanel.innerHTML = '';

  items.forEach(([key, label]) => {
    const info = document.createElement('div');
    info.className = 'inventory-item';
    const stock = current.items[key] || 0;
    if (stock <= 0) info.classList.add('used');
    info.innerHTML = `
      <div class="name">${label}</div>
      <div class="count">${stock}</div>
    `;
    info.addEventListener('click', () => {
      if (current.isHuman && canUseItem(current, key)) {
        if (key === 'cuffs') {
          const target = getLivingOpponents(current.id)[0];
          if (target) {
            useItem(current, key, target.id);
          } else {
            return;
          }
        } else {
          useItem(current, key, state.match.selectedTargetId);
        }
      }
    });
    ui.inventoryPanel.appendChild(info);
  });
}

function renderLog() {
  if (!state.match) return;
  ui.eventLog.innerHTML = '';
  state.match.log.forEach((entry) => {
    const item = document.createElement('div');
    item.className = 'log-entry';
    item.textContent = entry.message;
    ui.eventLog.appendChild(item);
  });
}

function renderPlayers() {
  if (!state.match) return;
  ui.playersPanel.innerHTML = '';

  state.match.players.forEach((player, index) => {
    const card = document.createElement('div');
    card.className = 'player-card';
    if (state.match.currentIndex === index) card.classList.add('active');
    if (!player.isAlive) card.classList.add('eliminated');

    const effects = Object.entries(player.effects)
      .filter(([key, value]) => key !== 'knownNextCard' && value > 0)
      .map(([key, value]) => {
        if (key === 'skipNextTurn') return `Пропуск: ${value}`;
        if (key === 'bonusDamage') return 'Ножівка';
        return `${key}: ${value}`;
      });

    card.innerHTML = `
      <div class="player-top">
        <div>
          <div class="badge">${player.isHuman ? 'Гравець' : 'Бот'}</div>
          <strong>${player.name}</strong>
        </div>
        <div class="status"><span class="dot ${player.isAlive ? '' : 'dead'}"></span></div>
      </div>
      <div>Здоров'я: ${player.health}/${player.maxHealth}</div>
      <div class="health-bar"><span style="width:${(player.health / player.maxHealth) * 100}%"></span></div>
      <div class="effect-list">
        ${effects.length ? effects.map((effect) => `<span class="effect-pill">${effect}</span>`).join('') : '<span class="effect-pill">Немає</span>'}
      </div>
    `;

    ui.playersPanel.appendChild(card);
  });
}

function renderActiveCard() {
  if (!state.match) return;
  const active = getCurrentPlayer();
  if (!active) return;

  ui.activePlayerCard.innerHTML = `
    <div class="player-chip">
      <div class="meta">
        <strong>${active.name}</strong>
        <span>${active.isHuman ? 'Ви' : 'Бот'}</span>
      </div>
      <span class="dot ${active.isAlive ? '' : 'dead'}"></span>
    </div>
    <div>Здоров'я: ${active.health}/${active.maxHealth}</div>
    <div>Відомий наступний набій: ${active.effects.knownNextCard ? getWeaponTypeLabel(active.effects.knownNextCard) : 'не виявлено'}</div>
    <div>Ефекти: ${Object.values(active.effects).filter((value) => value > 0).length ? 'активні' : 'немає'}</div>
  `;
}

function renderFirstPerson() {
  if (!state.match) return;

  const current = getCurrentPlayer();
  const players = getPlayers();
  ui.opponentsBank.innerHTML = '';

  players.forEach((player) => {
    if (player.id === current.id) return;
    const card = document.createElement('button');
    card.type = 'button';
    card.className = 'opponent-card';
    if (player.isAlive && state.match.selectedTargetId === player.id) card.classList.add('selected');
    if (state.match.currentIndex === players.findIndex((entry) => entry.id === player.id)) card.classList.add('active');
    if (!player.isAlive) {
      card.style.opacity = '0.45';
    }
    card.innerHTML = `
      <div class="silhouette"></div>
      <div>${player.name}</div>
      <div class="health-bar"><span style="width:${(player.health / player.maxHealth) * 100}%"></span></div>
      <div>${player.health}/${player.maxHealth}</div>
    `;
    card.disabled = !player.isAlive || !current.isHuman || current.id === player.id || state.match.gameOver;
    card.addEventListener('click', () => {
      if (current.isHuman) {
        handleAction('selectTarget', { targetId: player.id });
      }
    });
    ui.opponentsBank.appendChild(card);
  });
}

function botTurn() {
  if (!state.match || state.match.gameOver) return;
  const bot = getCurrentPlayer();
  if (!bot || bot.isHuman || !bot.isAlive || state.match.turnLocked) return;

  if (state.match.deck.length === 0) {
    reloadDeck(bot);
    setTimeout(() => botTurn(), 500);
    return;
  }

  const current = getCurrentPlayer();
  if (!current || !current.isAlive) return;

  const targets = getLivingOpponents(current.id);
  if (!targets.length) {
    battleEnd(current.name);
    return;
  }

  let chosenTarget = null;
  const difficulty = state.match.difficulty || state.settings.difficulty;

  if (difficulty === 'easy') {
    chosenTarget = targets[Math.floor(Math.random() * targets.length)];
  } else if (difficulty === 'medium') {
    const target = [...targets].sort((a, b) => a.health - b.health)[0];
    chosenTarget = target;
  } else {
    let best = null;
    for (const candidate of targets) {
      const risk = Math.max(0, 4 - candidate.health);
      const favorable = candidate.health <= current.health ? 1 : 0;
      const itemBonus = current.items.knife > 0 ? 1 : 0;
      const score = favorable + risk + itemBonus;
      best = !best || score > best.score ? { candidate, score } : best;
    }
    chosenTarget = best ? best.candidate : targets[0];
  }

  if (current.effects.knownNextCard === 'live' && current.items.scope > 0) {
    state.match.selectedTargetId = chosenTarget.id;
  }

  if (current.items.medkit > 0 && current.health <= 1 && Math.random() < (difficulty === 'hard' ? 0.8 : difficulty === 'medium' ? 0.5 : 0.25)) {
    useItem(current, 'medkit');
    setTimeout(() => botTurn(), 300);
    return;
  }

  if (current.items.cuffs > 0 && difficulty === 'hard' && Math.random() < 0.6) {
    const victim = targets[Math.floor(Math.random() * targets.length)];
    useItem(current, 'cuffs', victim.id);
    setTimeout(() => botTurn(), 300);
    return;
  }

  if (current.items.knife > 0 && difficulty !== 'easy' && current.health > 1 && Math.random() < (difficulty === 'hard' ? 0.7 : 0.45)) {
    useItem(current, 'knife');
    setTimeout(() => botTurn(), 300);
    return;
  }

  if (current.items.scope > 0 && Math.random() < (difficulty === 'hard' ? 0.5 : difficulty === 'medium' ? 0.3 : 0.15)) {
    useItem(current, 'scope', chosenTarget.id);
    setTimeout(() => botTurn(), 300);
    return;
  }

  state.match.selectedTargetId = chosenTarget.id;
  fireAtTarget(chosenTarget.id);
}

function toggleViewMode() {
  const nextMode = state.match.viewMode === 'first' ? 'tactical' : 'first';
  state.match.viewMode = nextMode;
  state.settings.viewMode = nextMode;
  saveSettings();
  render();
  saveMatch();
}

function setupUI() {
  document.querySelectorAll('[data-action]').forEach((button) => {
    button.addEventListener('click', () => {
      const action = button.dataset.action;
      emitSound('button');
      if (action === 'quick-game' || action === 'bot-game') {
        state.settings.botCount = Number(state.settings.botCount || 1);
        generateMatch();
        ui.menuScreen.classList.remove('active');
        ui.gameScreen.classList.add('active');
        return;
      }
      if (action === 'rules') {
        openModal('rules');
        return;
      }
      if (action === 'settings') {
        openModal('settings');
        return;
      }
      if (action === 'continue') {
        const saved = loadSavedMatch();
        if (saved) {
          ui.menuScreen.classList.remove('active');
          ui.gameScreen.classList.add('active');
          state.match = saved;
          state.settings.viewMode = saved.viewMode || state.settings.viewMode;
          setViewMode(saved.viewMode || state.settings.viewMode);
          render();
        }
      }
    });
  });

  document.querySelectorAll('[data-close]').forEach((button) => {
    button.addEventListener('click', () => {
      const key = button.dataset.close;
      closeModal(key);
    });
  });

  ui.settingsBtn = document.getElementById('settingsBtn');
  ui.rulesBtn = document.getElementById('rulesBtn');
  ui.menuBtn = document.getElementById('menuBtn');
  ui.settingsBtn.addEventListener('click', () => openModal('settings'));
  ui.rulesBtn.addEventListener('click', () => openModal('rules'));
  ui.menuBtn.addEventListener('click', () => {
    ui.menuScreen.classList.add('active');
    ui.gameScreen.classList.remove('active');
    closeAllModals();
  });

  ui.viewToggleBtn.addEventListener('click', toggleViewMode);
  ui.reloadBtn.addEventListener('click', () => {
    if (isHumanTurn()) {
      const current = getCurrentPlayer();
      reloadDeck(current);
    }
  });
  ui.shootBtn.addEventListener('click', () => {
    const current = getCurrentPlayer();
    if (!current) return;
    if (!current.isHuman) return;
    handleAction('fire', { targetId: state.match.selectedTargetId });
  });
  ui.useItemBtn.addEventListener('click', () => {
    const current = getCurrentPlayer();
    if (!current || !current.isHuman) return;
    const available = Object.entries(ITEM_LABELS).find(([key, label]) => canUseItem(current, key));
    if (!available) return;
    const [key] = available;
    if (key === 'cuffs') {
      const target = getLivingOpponents(current.id)[0];
      if (target) {
        useItem(current, key, target.id);
      }
      return;
    }
    useItem(current, key, state.match.selectedTargetId);
  });

  ui.soundToggle.addEventListener('change', () => {
    state.settings.sound = ui.soundToggle.checked;
    saveSettings();
    if (state.settings.sound) ensureAudio();
  });

  ui.difficultySelect.addEventListener('change', () => {
    state.settings.difficulty = ui.difficultySelect.value;
    saveSettings();
    if (state.match) {
      state.match.difficulty = state.settings.difficulty;
      render();
    }
  });

  ui.botCountSelect.addEventListener('change', () => {
    state.settings.botCount = Number(ui.botCountSelect.value);
    saveSettings();
  });

  ui.viewModeSelect.addEventListener('change', () => {
    const nextMode = ui.viewModeSelect.value;
    state.settings.viewMode = nextMode;
    saveSettings();
    if (state.match) {
      state.match.viewMode = nextMode;
      render();
    }
  });

  document.getElementById('newGameBtn').addEventListener('click', () => {
    generateMatch();
    closeModal('settings');
    ui.menuScreen.classList.remove('active');
    ui.gameScreen.classList.add('active');
  });

  document.getElementById('rematchBtn').addEventListener('click', () => {
    generateMatch();
    closeModal('settings');
    ui.menuScreen.classList.remove('active');
    ui.gameScreen.classList.add('active');
  });

  window.addEventListener('keydown', handleKeydown);

  refreshSettingsControls();
}

function initialize() {
  setupUI();
  const saved = loadSavedMatch();
  if (saved) {
    state.match = saved;
    state.settings.viewMode = saved.viewMode || state.settings.viewMode;
    ui.menuScreen.classList.remove('active');
    ui.gameScreen.classList.add('active');
    setViewMode(saved.viewMode || state.settings.viewMode);
    render();
  } else {
    ui.menuScreen.classList.add('active');
    ui.gameScreen.classList.remove('active');
  }
  refreshSettingsControls();
  if (state.settings.sound) ensureAudio();
}

initialize();
