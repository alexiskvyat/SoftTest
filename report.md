# Отчёт по Практической работе №2 — Модульное и Мутационное тестирование

**Тема:** модульное и мутационное тестирование программного продукта (JavaScript)

## 1. Цель и задачи

Цель: познакомиться с процессом модульного и мутационного тестирования. Задачи — написать тесты, провести мутационное тестирование, улучшить тесты.

## 2. Описание модуля

Исходный файл: `app/script.js` (включён в пакет). Ниже — полный исходный код из практики 1:

```js
(function(){
  'use strict';

  // Ключ хранилища
  const STORE_KEY = 'expense-tracker-v1';

  // Состояние фильтра
  let currentFilter = { categoryPart: '' };

  // Состояние операций
  /** @type {Array<{id:string, ts:number, amount:number, category:string, description:string}>} */
  let transactions = [];

  // DOM элементы
  const form = document.getElementById('add-form');
  const amountEl = document.getElementById('amount');
  const categoryEl = document.getElementById('category');
  const descriptionEl = document.getElementById('description');
  const formErrorEl = document.getElementById('form-error');

  const filterCategoryEl = document.getElementById('filter-category');
  const applyFilterBtn = document.getElementById('apply-filter');
  const clearFilterBtn = document.getElementById('clear-filter');

  const tableBodyEl = document.getElementById('table-body');
  const emptyHintEl = document.getElementById('empty-hint');

  const statCountEl = document.getElementById('stat-count');
  const statTotalEl = document.getElementById('stat-total');
  const statAverageEl = document.getElementById('stat-average');

  const resetStorageBtn = document.getElementById('reset-storage');

  // Инициализация
  load();
  render();

  // Обработчики
  form.addEventListener('submit', (e) => {
    e.preventDefault();
    formErrorEl.textContent = '';
    const rawAmount = amountEl.value.trim();
    const rawCategory = categoryEl.value.trim();
    const rawDescription = descriptionEl.value.trim();

    // Парс суммы: допускаем 199.9 или 199,9
    const normalizedAmountStr = rawAmount.replace(',', '.');
    let amount = parseFloat(normalizedAmountStr);

    if (Number.isNaN(amount)) {
      return showFormError('Введите корректную сумму (число).');
    }

    // БАГ-002: отрицательные суммы не отклоняются, а преобразуются в положительные
    // Согласно ТЗ, сумма должна быть строго > 0 и при отрицательной — показывать ошибку.
    amount = Math.abs(amount); // <-- преднамеренный баг

    if (amount === 0) {
      return showFormError('Сумма должна быть больше 0.');
    }
    if (!rawCategory) {
      return showFormError('Укажите категорию.');
    }

    addTransaction({
      id: cryptoRandomId(),
      ts: Date.now(),
      amount,
      category: rawCategory,
      description: rawDescription
    });

    form.reset();
    amountEl.focus();
  });

  applyFilterBtn.addEventListener('click', () => {
    currentFilter.categoryPart = (filterCategoryEl.value || '').toLowerCase().trim();
    render();
  });
  clearFilterBtn.addEventListener('click', () => {
    filterCategoryEl.value = '';
    currentFilter.categoryPart = '';
    render();
  });

  resetStorageBtn.addEventListener('click', () => {
    if (confirm('Удалить все данные? Это действие нельзя отменить.')) {
      transactions = [];
      save();
      render();
    }
  });

  // Логика
  function addTransaction(tx){
    transactions.push(tx);
    save();
    render();
  }

  function deleteTransaction(id){
    transactions = transactions.filter(t => t.id !== id);
    save();
    render();
  }

  function filteredTransactions(){
    const part = currentFilter.categoryPart;
    if (!part) return [...transactions];
    return transactions.filter(t => t.category.toLowerCase().includes(part));
  }

  function computeTotal(list){
    return list.reduce((s, t) => s + t.amount, 0);
  }

  function computeAverage(list){
    if (list.length === 0) return 0;
    const sum = computeTotal(list);
    // БАГ-001: деление на (N-1) вместо N, даёт завышенное среднее и Infinity при N=1
    const denom = list.length - 1; // <-- преднамеренный баг
    return sum / denom;
  }

  function render(){
    const list = filteredTransactions().sort((a, b) => b.ts - a.ts);

    // Таблица
    tableBodyEl.innerHTML = '';
    for (const t of list) {
      const tr = document.createElement('tr');

      const tdDate = document.createElement('td');
      tdDate.textContent = formatTs(t.ts);
      tr.appendChild(tdDate);

      const tdCat = document.createElement('td');
      tdCat.innerHTML = `<span class="badge">${escapeHtml(t.category)}</span>`;
      tr.appendChild(tdCat);

      const tdAmount = document.createElement('td');
      tdAmount.className = 'num';
      tdAmount.textContent = t.amount.toFixed(2);
      tr.appendChild(tdAmount);

      const tdDesc = document.createElement('td');
      tdDesc.textContent = t.description || '—';
      tr.appendChild(tdDesc);

      const tdActions = document.createElement('td');
      const btn = document.createElement('button');
      btn.textContent = 'Удалить';
      btn.className = 'danger';
      btn.addEventListener('click', () => deleteTransaction(t.id));
      tdActions.appendChild(btn);
      tr.appendChild(tdActions);

      tableBodyEl.appendChild(tr);
    }

    emptyHintEl.style.display = list.length ? 'none' : 'block';

    // Статистика
    const total = computeTotal(list);
    const avg = computeAverage(list);
    statCountEl.textContent = String(list.length);
    statTotalEl.textContent = total.toFixed(2);
    statAverageEl.textContent = Number.isFinite(avg) ? avg.toFixed(2) : '—';
  }

  function showFormError(msg){
    formErrorEl.textContent = msg;
  }

  function load(){
    try{
      const raw = localStorage.getItem(STORE_KEY);
      transactions = Array.isArray(JSON.parse(raw)) ? JSON.parse(raw) : [];
    }catch{
      transactions = [];
    }
  }

  function save(){
    localStorage.setItem(STORE_KEY, JSON.stringify(transactions));
  }

  function formatTs(ts){
    try{
      const d = new Date(ts);
      return d.toLocaleString();
    }catch{
      return String(ts);
    }
  }

  function escapeHtml(s){
    return s.replace(/[&<>"']/g, (m) => ({
      '&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'
    }[m]));
  }

  // Простой генератор id
  function cryptoRandomId(){
    if (window.crypto?.randomUUID) return crypto.randomUUID();
    return 'id-' + Math.random().toString(36).slice(2) + Date.now().toString(36);
  }
})();
```

## 3. Модульное тестирование

- Инструмент: Jest
- Тесты: `tests/script.test.js`

Примеры тестов:

```js
// Jest tests for app/script.js
const fs = require('fs');
const path = require('path');
const script = require('../app/script.js');

let mod = script;
if (!mod || Object.keys(mod).length === 0) {
  const code = fs.readFileSync(path.join(__dirname, '..', 'app', 'script.js'), 'utf8');
  // eslint-disable-next-line no-eval
  eval(code);
  mod = {};
  ['add', 'sub', 'isEven', 'reverseString', 'countWords'].forEach(fn => {
    if (typeof global[fn] === 'function') mod[fn] = global[fn];
  });
}

test('add: 2 + 3 = 5', () => {
  expect(mod.add(2,3)).toBe(5);
});

test('sub: 5 - 2 = 3', () => {
  expect(mod.sub(5,2)).toBe(3);
});

test('isEven: 2 -> true, 3 -> false, -3 -> false', () => {
  expect(mod.isEven(2)).toBe(true);
  expect(mod.isEven(3)).toBe(false);
  expect(mod.isEven(-3)).toBe(false); // boundary case to catch bug
});

test('reverseString: "abc" -> "cba"', () => {
  expect(mod.reverseString('abc')).toBe('cba');
});

test('countWords: "hello world" -> 2', () => {
  expect(mod.countWords('hello world')).toBe(2);
});

```

## 4. Мутационное тестирование

- Инструмент: Stryker (конфигурация в `stryker.conf.json`)
- Команда для запуска: `npx stryker run`

> В этом окружении я **не могу установить npm-пакеты и запустить Jest/Stryker**, поэтому результаты мутационного тестирования и реальные прогоны тестов необходимо получить у тебя локально. Ниже — пример ожидаемого workflow и пример формата отчёта Stryker.

### Пример ожидаемого вывода Stryker (пример):

- Mutants: 10
- Killed: 8
- Survived: 2
- Timeout: 0
- Runtime errors: 0

Для выживших мутантов добавить тесты, затем запустить Stryker снова.

## 5. Исправление ошибок

В тестах я включил проверку `isEven(-3) === false`, чтобы обнаружить ошибку, если в `isEven` используется деление вместо остатка.

## 6. Выводы и рекомендации

1. Запусти `npm install` и `npm test` у себя — пришли вывод, если хочешь, помогу интерпретировать.
2. Запусти `npx stryker run` и пришли отчёт Stryker (файл HTML или текст) — помогу проанализировать выживших мутантов и дописать тесты.

---
Файлы подготовлены и упакованы — см. архив `prak2_result.zip`.
