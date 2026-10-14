
"use strict";

// ОСТАННІЙ ПАТРОН — ігрова логіка
const $ = (id) => document.getElementById(id);

const state = {
  players: [],
  ammo: [],
  current: 0,
  selectedTarget: 1,
  selectedItem: null,
  view: "first",
  difficulty: "easy",
  botCount: 2,
  sound: true,
  poweredShot: false,
  roundOver: false,
  log: [],
  nextId: 0,
};

const items = [
  { id: "magnifier", name: "Лупа", icon: "🔎", description: "Перевірити наступний набій" },
  { id: "medkit", name: "Аптечка", icon: "🩹", description: "Відновити здоров'я" },
  { id: "cuffs", name: "Наручники", icon: "⛓️", description: "Пропустити хід суперника" },
  { id: "saw", name: "Ножівка", icon: "🪚", description: "Посилити наступний постріл" },
  { id: "eject", name: "Викинути набій", icon: "📤", description: "Видалити наступний набій" },
];

function player(name, isBot = false) {
  return {
    id: ++state.nextId,
    name,
    isBot,
    hp: 3,
    maxHp: 3,
    alive: true,
    skip: false,
    inventory: [],
    damageBoost: false,
  };
}

function shuffle(array) {
  for (let i = array.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [array[i], array[j]] = [array[j], array[i]];
  }
  return array;
}

function addLog(message) {
  state.log.unshift(message);
  state.log = state.log.slice(0, 35);
  const log = $("eventLog");
  if (log) {
    log.innerHTML = state.log
      .map((line) => `<p class="log-entry">${escapeHTML(line)}</p>`)
      .join("");
  }
}

function escapeHTML(value) {
  return String(value).replace(/[&<>"']/g, (char) => ({
    "&": "&amp;",
    "<": "&lt;",
    ">": "&gt;",
    '"': "&quot;",
    "'": "&#39;",
  })[char]);
}

function alivePlayers() {
  return state.players.filter((p) => p.alive);
}

function currentPlayer() {
  return state.players[state.current];
}

function makeAmmo() {
  const live = 2 + Math.floor(Math.random() * 3);
  const blank = 2 + Math.floor(Math.random() * 3);
  state.ammo = shuffle([
    ...Array(live).fill(true),
    ...Array(blank).fill(false),
  ]);
}

function giveItems(p) {
  p.inventory = [];
  const count = 1 + Math.floor(Math.random() * 3);
  for (let i = 0; i < count; i++) {
    p.inventory.push(items[Math.floor(Math.random() * items.length)].id);
  }
}

function startGame(botCount = 2) {
  closeModal("rules");
  closeModal("settings");

  state.players = [player("Ти")];
  for (let i = 0; i < botCount; i++) {
    state.players.push(player(`Бот ${i + 1}`, true));
  }

  state.current = 0;
  state.selectedTarget = state.players[1].id;
  state.selectedItem = null;
  state.poweredShot = false;
  state.roundOver = false;
  state.log = [];
  makeAmmo();
  state.players.forEach(giveItems);

  $("menuScreen").classList.remove("active");
  $("gameScreen").classList.add("active");
  $("matchTitle").textContent = "ОСТАННІЙ ПАТРОН";

  addLog("Матч розпочато. Набої перемішано.");
  addLog("Твій хід. Обери дію.");
  setView(state.view);
  render();
  runBotIfNeeded();
}

function render() {
  const alive = alivePlayers();
  const active = currentPlayer();

  $("ammoCount").textContent = state.ammo.length;
  $("liveCount").textContent = state.ammo.filter(Boolean).length;
  $("blankCount").textContent = state.ammo.filter((x) => !x).length;

  $("activePlayerCard").innerHTML = active
    ? `<strong>${escapeHTML(active.name)}</strong>
       <p>Здоров'я: ${"♥".repeat(active.hp)}${"♡".repeat(active.maxHp - active.hp)}</p>
       <p>${active.isBot ? "Керує бот" : "Твій хід"}</p>`
    : "";

  $("opponentsBank").innerHTML = state.players
    .filter((p) => p.id !== (active && active.id))
    .map((p) => `
      <button type="button"
        class="opponent-card ${p.alive ? "" : "eliminated"} ${p.id === state.selectedTarget ? "selected" : ""}"
        data-target="${p.id}" ${!p.alive || !active || active.isBot ? "disabled" : ""}>
        <strong>${escapeHTML(p.name)}</strong>
        <span>${p.alive ? "♥".repeat(p.hp) + "♡".repeat(p.maxHp - p.hp) : "Вибув"}</span>
      </button>
    `).join("");

  $("playersPanel").innerHTML = state.players.map((p) => `
    <div class="player-card ${p.alive ? "" : "eliminated"}">
      <h3>${escapeHTML(p.name)} ${p.isBot ? "🤖" : "👤"}</h3>
      <p>${p.alive ? "♥".repeat(p.hp) + "♡".repeat(p.maxHp - p.hp) : "Вибув із гри"}</p>
      <p>${p.id === (active && active.id) ? "▶ Активний гравець" : ""}</p>
    </div>
  `).join("");

  $("inventoryPanel").innerHTML = active
    ? active.inventory.map((id, index) => {
        const item = items.find((x) => x.id === id);
        if (!item) return "";
        return `<button type="button" class="inventory-item"
          data-item-index="${index}" ${active.isBot || state.roundOver ? "disabled" : ""}>
          ${item.icon} ${escapeHTML(item.name)}
        </button>`;
      }).join("")
    : "";

  $("targetBar").innerHTML = alive
    .filter((p) => p.id !== (active && active.id))
    .map((p) => `<button type="button" data-target="${p.id}"
      class="${p.id === state.selectedTarget ? "selected" : ""}"
      ${!active || active.isBot || state.roundOver ? "disabled" : ""}>
      ${escapeHTML(p.name)}
    </button>`).join("");

  $("shootBtn").disabled = state.roundOver || !active || active.isBot || !state.ammo.length;
  $("reloadBtn").disabled = state.roundOver || !active || active.isBot;
  $("useItemBtn").disabled = state.roundOver || !active || active.isBot || !active.inventory.length;

  $("continueBtn").disabled = false;
}

function setView(view) {
  state.view = view === "tactical" ? "tactical" : "first";
  $("firstPersonView").classList.toggle("hidden", state.view !== "first");
  $("firstPersonView").classList.toggle("active", state.view === "first");
  $("tacticalView").classList.toggle("hidden", state.view !== "tactical");
  $("tacticalView").classList.toggle("active", state.view === "tactical");
  $("viewModeSelect").value = state.view;
}

function selectTarget(id) {
  const target = state.players.find((p) => p.id === Number(id));
  const active = currentPlayer();
  if (!target || !target.alive || !active || target.id === active.id) return;
  state.selectedTarget = target.id;
  render();
}

function shoot(targetId, selfTarget = false) {
  if (state.roundOver || !state.ammo.length) {
    if (!state.roundOver) addLog("Набої закінчилися. Перезарядись.");
    render();
    return;
  }

  const shooter = currentPlayer();
  if (!shooter || !shooter.alive) return;

  const target = selfTarget
    ? shooter
    : state.players.find((p) => p.id === Number(targetId));

  if (!target || !target.alive || (!selfTarget && target.id === shooter.id)) {
    addLog("Спочатку обери живу ціль.");
    return;
  }

  const liveRound = state.ammo.shift();
  const damage = liveRound ? (shooter.damageBoost ? 2 : 1) : 0;
  shooter.damageBoost = false;

  if (liveRound) {
    target.hp -= damage;
    addLog(`${shooter.name}: бойовий постріл! ${target.name} втрачає ${damage} здоров'я.`);
    if (target.hp <= 0) {
      target.hp = 0;
      target.alive = false;
      addLog(`${target.name} вибуває з матчу.`);
    }
  } else {
    addLog(`${shooter.name}: холостий постріл.`);
  }

  state.poweredShot = false;

  if (checkWinner()) {
    render();
    return;
  }

  // Холостий постріл у себе дає повторний хід.
  if (!(selfTarget && !liveRound)) nextTurn();
  render();
  runBotIfNeeded();
}

function reload() {
  if (state.roundOver) return;
  makeAmmo();
  addLog("Набої перезаряджено та перемішано.");
  nextTurn();
  render();
  runBotIfNeeded();
}

function useItem(index, targetId) {
  if (state.roundOver) return;

  const user = currentPlayer();
  if (!user || user.isBot) return;

  const itemId = user.inventory[index];
  const item = items.find((x) => x.id === itemId);
  if (!item) return;

  const target = state.players.find((p) => p.id === Number(targetId));
  if (["cuffs"].includes(itemId) &&
      (!target || !target.alive || target.id === user.id)) {
    addLog("Обери живого суперника для наручників.");
    return;
  }

  if (itemId === "medkit" && user.hp >= user.maxHp) {
    addLog("Здоров'я вже повне.");
    return;
  }

  if (itemId === "magnifier" && !state.ammo.length) {
    addLog("Немає набоїв для перевірки.");
    return;
  }

  if (itemId === "eject" && !state.ammo.length) {
    addLog("Немає набоїв для викидання.");
    return;
  }

  user.inventory.splice(index, 1);

  switch (itemId) {
    case "magnifier":
      addLog(`Лупа показує: ${state.ammo[0] ? "бойовий набій" : "холостий набій"}.`);
      break;
    case "medkit":
      user.hp = Math.min(user.maxHp, user.hp + 1);
      addLog(`${user.name} відновлює 1 здоров'я.`);
      break;
    case "cuffs":
      target.skip = true;
      addLog(`${target.name} пропустить наступний хід.`);
      break;
    case "saw":
      user.damageBoost = true;
      addLog(`${user.name} готує посилений постріл.`);
      break;
    case "eject":
      state.ammo.shift();
      addLog("Наступний набій видалено без пострілу.");
      break;
  }

  nextTurn();
  render();
  runBotIfNeeded();
}

function nextTurn() {
  if (state.roundOver) return;

  const total = state.players.length;
  for (let i = 0; i < total; i++) {
    state.current = (state.current + 1) % total;
    const p = currentPlayer();

    if (!p.alive) continue;

    if (p.skip) {
      p.skip = false;
      addLog(`${p.name} пропускає хід через наручники.`);
      continue;
    }

    addLog(`Хід гравця: ${p.name}.`);
    return;
  }
}

function checkWinner() {
  const alive = alivePlayers();
  if (alive.length > 1) return false;

  state.roundOver = true;
  const winner = alive[0];
  addLog(winner ? `🏆 Переможець: ${winner.name}!` : "Матч завершився внічию.");
  $("matchTitle").textContent = winner
    ? `ПЕРЕМОЖЕЦЬ: ${winner.name.toUpperCase()}`
    : "НІЧИЯ";
  return true;
}

function runBotIfNeeded() {
  if (state.roundOver) return;
  const bot = currentPlayer();
  if (!bot || !bot.isBot) return;

  // Простий локальний ШІ: приймає рішення без сервера.
  window.setTimeout(() => {
    if (state.roundOver || currentPlayer() !== bot) return;

    if (bot.inventory.length && Math.random() < (state.difficulty === "hard" ? 0.65 : 0.3)) {
      const idx = bot.inventory.findIndex((id) =>
        id === "medkit" && bot.hp < bot.maxHp ||
        id === "magnifier" ||
        id === "saw"
      );

      if (idx >= 0) {
        const id = bot.inventory[idx];
        if (id === "magnifier" && state.ammo.length) {
          const known = state.ammo[0];
          addLog(`${bot.name} перевіряє набій: ${known ? "бойовий" : "холостий"}.`);
          bot.inventory.splice(idx, 1);
          if (!known && Math.random() < 0.7) {
            // Бот не витрачає холостий набій на суперника.
            state.ammo.shift();
            addLog(`${bot.name} викидає холостий набій.`);
            nextTurn();
            render();
            runBotIfNeeded();
            return;
          }
        } else if (id === "medkit" && bot.hp < bot.maxHp) {
          bot.hp++;
          bot.inventory.splice(idx, 1);
          addLog(`${bot.name} використовує аптечку.`);
        } else if (id === "saw") {
          bot.damageBoost = true;
          bot.inventory.splice(idx, 1);
          addLog(`${bot.name} готує ножівку.`);
        }
      }
    }

    if (!state.ammo.length) {
      makeAmmo();
      addLog(`${bot.name} перезаряджається.`);
      nextTurn();
      render();
      runBotIfNeeded();
      return;
    }

    const targets = alivePlayers().filter((p) => p.id !== bot.id);
    const target = targets[Math.floor(Math.random() * targets.length)];
    if (!target) return;

    const selfShot = state.ammo[0] === false &&
      Math.random() < (state.difficulty === "hard" ? 0.7 : 0.35);

    shoot(selfShot ? bot.id : target.id, selfShot);
  }, 500);
}

function openModal(name) {
  const modal = $(`${name}Modal`);
  if (!modal) return;
  modal.classList.remove("hidden");
  modal.setAttribute("aria-hidden", "false");
}

function closeModal(name) {
  const modal = $(`${name}Modal`);
  if (!modal) return;
  modal.classList.add("hidden");
  modal.setAttribute("aria-hidden", "true");
}

function showMenu() {
  $("gameScreen").classList.remove("active");
  $("menuScreen").classList.add("active");
  closeModal("settings");
  closeModal("rules");
}

function startFromSettings() {
  state.difficulty = $("difficultySelect").value;
  state.botCount = Number($("botCountSelect").value);
  state.sound = $("soundToggle").checked;
  state.view = $("viewModeSelect").value;
  startGame(state.botCount);
}

document.addEventListener("click", (event) => {
  const actionButton = event.target.closest("[data-action]");
  if (actionButton) {
    switch (actionButton.dataset.action) {
      case "quick-game":
        startGame(1);
        break;
      case "bot-game":
        startGame(Number($("botCountSelect").value));
        break;
      case "rules":
        openModal("rules");
        break;
      case "settings":
        openModal("settings");
        break;
      case "continue":
        if (state.players.length && !state.roundOver) {
          $("menuScreen").classList.remove("active");
          $("gameScreen").classList.add("active");
          render();
        } else {
          addLog("Немає активного матчу. Почни нову гру.");
        }
        break;
    }
  }

  const closeButton = event.target.closest("[data-close]");
  if (closeButton) closeModal(closeButton.dataset.close);

  const targetButton = event.target.closest("[data-target]");
  if (targetButton && !targetButton.disabled) {
    selectTarget(targetButton.dataset.target);
  }

  const itemButton = event.target.closest("[data-item-index]");
  if (itemButton && !itemButton.disabled) {
    useItem(Number(itemButton.dataset.itemIndex), state.selectedTarget);
  }
});

$("shootBtn").addEventListener("click", () => {
  shoot(state.selectedTarget, false);
});

$("reloadBtn").addEventListener("click", reload);

$("useItemBtn").addEventListener("click", () => {
  const p = currentPlayer();
  if (!p || !p.inventory.length) {
    addLog("В інвентарі немає предметів.");
    return;
  }
  useItem(0, state.selectedTarget);
});

$("viewToggleBtn").addEventListener("click", () => {
  setView(state.view === "first" ? "tactical" : "first");
});

$("rulesBtn").addEventListener("click", () => openModal("rules"));
$("settingsBtn").addEventListener("click", () => openModal("settings"));
$("menuBtn").addEventListener("click", showMenu);

$("viewModeSelect").addEventListener("change", (event) => {
  setView(event.target.value);
});

$("soundToggle").addEventListener("change", (event) => {
  state.sound = event.target.checked;
});

$("difficultySelect").addEventListener("change", (event) => {
  state.difficulty = event.target.value;
});

$("botCountSelect").addEventListener("change", (event) => {
  state.botCount = Number(event.target.value);
});

$("newGameBtn").addEventListener("click", startFromSettings);
$("rematchBtn").addEventListener("click", startFromSettings);

document.querySelectorAll(".modal").forEach((modal) => {
  modal.addEventListener("click", (event) => {
    if (event.target === modal) {
      modal.classList.add("hidden");
      modal.setAttribute("aria-hidden", "true");
    }
  });
});

document.addEventListener("keydown", (event) => {
  if (event.key === "Escape") {
    closeModal("rules");
    closeModal("settings");
  }
});

// Початковий екран
$("menuScreen").classList.add("active");
$("gameScreen").classList.remove("active");
$("continueBtn").disabled = true;
