# Работа с React-приложением meter.printecs.com

## Обзор

Сайт meter.printecs.com построен на React, что требует особого подхода при работе с DOM-элементами через пользовательские скрипты. React использует виртуальный DOM и систему отслеживания изменений, поэтому простое изменение `value` или `checked` через JavaScript не всегда приводит к обновлению состояния React-компонента.

## Проблемы при работе с React

### 1. Прямое изменение значений не работает
```javascript
// ❌ НЕ РАБОТАЕТ - React не узнает об изменении
input.value = 'новое значение';
checkbox.checked = true;
```

### 2. React использует внутренний трекер изменений
React отслеживает изменения через `_valueTracker`, который сравнивает текущее значение с предыдущим. Если мы просто меняем значение, React не видит разницы.

### 3. События должны быть правильно диспатчены
React слушает события `input`, `change`, `blur` и другие. Но простое создание события может не сработать, если не использовать правильные параметры.

### 4. Поля типа date требуют особого подхода
Поля `input[type="date"]` контролируются React через `valueAsDate` и могут автоматически обновлять отображаемый `span` с датой. Простое изменение `value` может не обновить отображаемую дату. Также важно учитывать часовой пояс при работе с датами.

## Решения

### Функция setReactInputValue

Универсальная функция для установки значений в React input/textarea поля:

```javascript
async function setReactInputValue(selector, value, maxRetries = 3, silent = false) {
    const input = await waitForSelector(selector);
    
    // Флаг для временной блокировки workflow при silent режиме
    let workflowBlocked = false;
    if (silent) {
        window._blockWorkflow = true;
        workflowBlocked = true;
    }
    
    try {
        for (let attempt = 1; attempt <= maxRetries; attempt++) {
            const lastValue = input.value;
            
            // Используем нативный setter для правильной работы с React
            const isTextarea = input.tagName.toLowerCase() === 'textarea';
            const prototype = isTextarea 
                ? window.HTMLTextAreaElement.prototype 
                : window.HTMLInputElement.prototype;
            
            const nativeValueSetter = Object.getOwnPropertyDescriptor(prototype, 'value')?.set;
            
            if (nativeValueSetter) {
                nativeValueSetter.call(input, value);
            } else {
                input.value = value;
            }
            
            // Работаем с React трекером
            const tracker = input._valueTracker;
            if (tracker) {
                try {
                    tracker.setValue(lastValue); // Устанавливаем старое значение в трекер
                } catch (e) {
                    // Игнорируем ошибки трекера
                }
            }
            
            // Диспатчим события с bubbles: true для React
            const inputEvent = new Event('input', { bubbles: true, cancelable: true });
            const changeEvent = new Event('change', { bubbles: true, cancelable: true });
            const blurEvent = new Event('blur', { bubbles: true, cancelable: true });
            
            // Добавляем флаг для идентификации программных событий
            if (silent) {
                inputEvent._programmatic = true;
                changeEvent._programmatic = true;
                blurEvent._programmatic = true;
            }
            
            input.dispatchEvent(inputEvent);
            input.dispatchEvent(changeEvent);
            input.dispatchEvent(blurEvent);
            
            await wait(150);
            
            // Проверяем, что значение установилось
            if (input.value === String(value)) {
                return true;
            }
            
            // Если не установилось и это не последняя попытка - ждем и пробуем снова
            if (attempt < maxRetries) {
                await wait(200);
            }
        }
        
        // Если после всех попыток значение не установилось
        if (input.value !== String(value)) {
            showToast(`⚠️ Не удалось установить значение в поле ${selector}`, 3000, '#a33');
            return false;
        }
        
        return true;
    } finally {
        // Снимаем блокировку workflow после небольшой задержки
        if (workflowBlocked) {
            setTimeout(() => {
                window._blockWorkflow = false;
            }, 500);
        }
    }
}
```

#### Ключевые моменты:

1. **Использование нативного setter**: `Object.getOwnPropertyDescriptor(prototype, 'value')?.set` позволяет обойти некоторые ограничения React.

2. **Работа с _valueTracker**: 
   - Сохраняем старое значение в трекер через `tracker.setValue(lastValue)`
   - Это заставляет React увидеть разницу между старым и новым значением

3. **Правильные события**:
   - `bubbles: true` - события всплывают, что важно для React
   - `cancelable: true` - события можно отменить
   - Диспатчим `input`, `change`, `blur` в правильном порядке

4. **Retry-логика**: 
   - React может асинхронно обновлять состояние
   - Делаем несколько попыток с задержками

5. **Silent режим**:
   - Флаг `_programmatic` позволяет отличить программные события от пользовательских
   - `window._blockWorkflow` блокирует другие обработчики, которые могут мешать

### Функция setDateValue

Специальная функция для работы с полями типа `input[type="date"]`. Эти поля требуют особого подхода, так как React может контролировать их через `valueAsDate` и автоматически обновлять отображаемый `span` с датой.

```javascript
async function setDateValue(input, value) {
  // Проверяем, что value имеет правильный формат YYYY-MM-DD
  if (!/^\d{4}-\d{2}-\d{2}$/.test(value)) {
    console.error('Неправильный формат даты:', value);
    return false;
  }
  
  // Проверяем текущее значение - если оно уже правильное, не устанавливаем снова
  const currentValue = input.value || '';
  if (currentValue === value) {
    return true;
  }
  
  const lastValue = currentValue || '';
  
  // Фокусируемся на поле для симуляции реального взаимодействия
  input.focus();
  await new Promise(r => setTimeout(r, 100));
  
  // Удаляем старый атрибут value, если он есть (важно для React)
  const oldValueAttr = input.getAttribute('value');
  if (oldValueAttr && oldValueAttr !== value) {
    input.removeAttribute('value');
    await new Promise(r => setTimeout(r, 50));
  }
  
  // Основной метод: Используем valueAsDate (работает лучше всего с React)
  // Используем T12:00:00 вместо T00:00:00 чтобы избежать проблем с часовыми поясами
  let valueSet = false;
  try {
    const dateObj = new Date(value + 'T12:00:00');
    if (!isNaN(dateObj.getTime())) {
      input.valueAsDate = dateObj;
      await new Promise(r => setTimeout(r, 150));
      
      // Проверяем результат - valueAsDate может изменить значение из-за часового пояса
      const resultValue = input.value || '';
      if (resultValue === value || resultValue.startsWith(value.split('-')[0] + '-')) {
        valueSet = true;
        // Если valueAsDate изменил значение из-за часового пояса, корректируем
        if (resultValue !== value) {
          // Используем нативный setter для точной установки значения
          const prototype = window.HTMLInputElement.prototype;
          const nativeValueSetter = Object.getOwnPropertyDescriptor(prototype, 'value')?.set;
          if (nativeValueSetter) {
            nativeValueSetter.call(input, value);
          } else {
            input.value = value;
          }
          await new Promise(r => setTimeout(r, 50));
        }
      }
    }
  } catch (e) {
    // Если valueAsDate не сработал, продолжаем
  }
  
  // Если valueAsDate не сработал, используем нативный setter
  if (!valueSet) {
    const prototype = window.HTMLInputElement.prototype;
    const nativeValueSetter = Object.getOwnPropertyDescriptor(prototype, 'value')?.set;
    
    if (nativeValueSetter) {
      nativeValueSetter.call(input, value);
    } else {
      input.value = value;
    }
    await new Promise(r => setTimeout(r, 50));
  }
  
  // Устанавливаем атрибут value (важно для React)
  input.setAttribute('value', value);
  
  // Работаем с React трекером (если доступен)
  try {
    const tracker = input._valueTracker;
    if (tracker && typeof tracker.setValue === 'function') {
      tracker.setValue(lastValue);
    }
  } catch (e) {
    // Игнорируем ошибки трекера
  }
  
  // Диспатчим события для React
  const inputEvent = new InputEvent('input', { 
    bubbles: true, 
    cancelable: true,
    data: value,
    inputType: 'insertText',
    isComposing: false
  });
  input.dispatchEvent(inputEvent);
  await new Promise(r => setTimeout(r, 100));
  
  const changeEvent = new Event('change', { bubbles: true, cancelable: true });
  input.dispatchEvent(changeEvent);
  await new Promise(r => setTimeout(r, 100));
  
  // Обновляем span с отформатированной датой (React может не обновить его автоматически)
  const container = input.closest('._inputContainer_27y0k_15') || 
                   input.closest('En') || 
                   input.closest('en') ||
                   input.parentElement;
  if (container) {
    const span = container.querySelector('span');
    if (span) {
      // Форматируем дату в формат "DD.MM.YYYY"
      const [year, month, day] = value.split('-');
      const formattedDate = `${day}.${month}.${year}`;
      span.textContent = formattedDate;
      
      // Обновляем span еще раз с задержкой (React может перезаписать его)
      await new Promise(r => setTimeout(r, 200));
      if (span.textContent !== formattedDate) {
        span.textContent = formattedDate;
      }
      
      // Финальное обновление span после всех событий
      await new Promise(r => setTimeout(r, 200));
      if (span.textContent !== formattedDate) {
        span.textContent = formattedDate;
      }
    }
  }
  
  // Убираем фокус
  input.blur();
  const blurEvent = new Event('blur', { bubbles: true, cancelable: true });
  input.dispatchEvent(blurEvent);
  
  await new Promise(r => setTimeout(r, 200));
  
  // Проверяем результат
  const finalValue = input.value || '';
  const finalAttr = input.getAttribute('value') || '';
  const success = finalValue === value || finalAttr === value || finalValue.startsWith(value.split('-')[0] + '-');
  
  return success;
}
```

#### Ключевые моменты работы с date полями:

1. **Использование `valueAsDate`**: Это основной метод для работы с `input[type="date"]`. React лучше всего реагирует на изменения через `valueAsDate`, и автоматически обновляет отображаемый `span`.

2. **Учет часового пояса**: Используется `T12:00:00` вместо `T00:00:00` для избежания проблем с часовыми поясами, которые могут сдвинуть дату на день назад.

3. **Обновление span**: React может автоматически обновить `span` с отформатированной датой после `valueAsDate`, но для надежности мы обновляем его вручную несколько раз с задержками.

4. **Поиск поля по label**: Для надежности поле ищется по label "Дата предыдущей поверки", а не просто по типу `input[type="date"]`, чтобы не выбрать неправильное поле (например, "Дата проверки").

5. **Работа с React трекером**: Используется `_valueTracker` для правильного отслеживания изменений в React.

#### Функция findLastInspectionInput

Для поиска поля "Дата предыдущей поверки" используется специальная функция, которая ищет поле несколькими способами:

```javascript
function findLastInspectionInput() {
  // Сначала пробуем найти через компонент En с name="last_inspection_year"
  const enComponent = document.querySelector('En[name="last_inspection_year"]') ||
                     document.querySelector('en[name="last_inspection_year"]') ||
                     document.querySelector('[name="last_inspection_year"]');
  
  if (enComponent) {
    const input = enComponent.querySelector('input[name="last_inspection_year"]') ||
                 enComponent.querySelector('input#last_inspection_year') ||
                 enComponent.querySelector('input[type="date"]');
    if (input) return input;
  }
  
  // Ищем по name атрибуту
  const inputByName = document.querySelector('input[name="last_inspection_year"]') ||
                     document.querySelector('input#last_inspection_year') ||
                     document.querySelector('input[type="date"][name="last_inspection_year"]');
  if (inputByName) return inputByName;
  
  // Ищем по label "Дата предыдущей поверки" (самый надежный способ)
  const labels = Array.from(document.querySelectorAll('label'));
  const targetLabel = labels.find(l => l.textContent && l.textContent.includes('Дата предыдущей поверки'));
  
  if (targetLabel) {
    let container = targetLabel.nextElementSibling;
    if (!container || !container.classList.contains('_inputContainer_27y0k_15')) {
      container = targetLabel.parentElement.querySelector('._inputContainer_27y0k_15');
    }
    
    if (container) {
      const input = container.querySelector('input[type="date"]');
      if (input) return input;
    }
    
    const parentContainer = targetLabel.closest('._dateInput_27y0k_1') || 
                           targetLabel.closest('div');
    if (parentContainer) {
      const input = parentContainer.querySelector('input[type="date"]');
      if (input) return input;
    }
  }
  
  // В крайнем случае возвращаем null, чтобы не выбрать неправильное поле
  return null;
}
```

**Важно**: Функция возвращает `null` вместо первого попавшегося `input[type="date"]`, чтобы не выбрать неправильное поле (например, "Дата проверки" вместо "Дата предыдущей поверки").

### Функция setReactCheckboxValue

Аналогичная функция для работы с checkbox:

```javascript
async function setReactCheckboxValue(selector, checked, maxRetries = 3, silent = false) {
    const checkbox = await waitForSelector(selector);
    if (!checkbox || checkbox.type !== 'checkbox') {
        return false;
    }
    
    // Флаг для временной блокировки workflow при silent режиме
    let workflowBlocked = false;
    if (silent) {
        window._blockWorkflow = true;
        workflowBlocked = true;
    }
    
    try {
        for (let attempt = 1; attempt <= maxRetries; attempt++) {
            const lastChecked = checkbox.checked;
            
            // Используем нативный setter для правильной работы с React
            const prototype = window.HTMLInputElement.prototype;
            const nativeCheckedSetter = Object.getOwnPropertyDescriptor(prototype, 'checked')?.set;
            
            if (nativeCheckedSetter) {
                nativeCheckedSetter.call(checkbox, checked);
            } else {
                checkbox.checked = checked;
            }
            
            // Работаем с React трекером (для checkbox тоже используется _valueTracker)
            const tracker = checkbox._valueTracker;
            if (tracker) {
                try {
                    tracker.setValue(lastChecked);
                } catch (e) {
                    // Игнорируем ошибки трекера
                }
            }
            
            // Диспатчим события с bubbles: true для React
            const changeEvent = new Event('change', { bubbles: true, cancelable: true });
            const inputEvent = new Event('input', { bubbles: true, cancelable: true });
            const clickEvent = new Event('click', { bubbles: true, cancelable: true });
            
            // Добавляем флаг для идентификации программных событий
            if (silent) {
                changeEvent._programmatic = true;
                inputEvent._programmatic = true;
                clickEvent._programmatic = true;
            }
            
            checkbox.dispatchEvent(clickEvent);
            checkbox.dispatchEvent(inputEvent);
            checkbox.dispatchEvent(changeEvent);
            
            await wait(150);
            
            // Проверяем, что значение установилось
            if (checkbox.checked === checked) {
                return true;
            }
            
            // Если не установилось и это не последняя попытка - ждем и пробуем снова
            if (attempt < maxRetries) {
                await wait(200);
            }
        }
        
        // Если после всех попыток значение не установилось
        if (checkbox.checked !== checked) {
            return false;
        }
        
        return true;
    } finally {
        // Снимаем блокировку workflow после небольшой задержки
        if (workflowBlocked) {
            setTimeout(() => {
                window._blockWorkflow = false;
            }, 500);
        }
    }
}
```

### Работа с селектами (select)

React-селекты требуют особого подхода. В проекте используется функция `changeComplianceSelect` для изменения результата проверки и специальная логика для изменения типа счетчика.

#### Функция changeCounterType

Для изменения типа счетчика используется функция `changeCounterType`, которая работает с React-селектами и полями ввода:

```javascript
async function changeCounterType(counterTypeText) {
  // Проверяем, может быть поле уже доступно
  let factTypeInput = document.querySelector('input#fact_counter_type') ||
                     document.querySelector('input._input_gy901_8[name="fact_counter_type"]') ||
                     document.querySelector('input[name="fact_counter_type"]');
  
  if (factTypeInput && factTypeInput.offsetParent !== null) {
    // Поле уже видимо, просто заполняем его
    await setReactInputValue('input#fact_counter_type', counterTypeText);
    return true;
  }

  // Ищем элемент с классом _value_8fpl2_13, который содержит текст "Соответствует"
  const allValueElements = document.querySelectorAll('._value_8fpl2_13');
  let complianceValueEl = null;
  
  for (const valueEl of allValueElements) {
    if (valueEl.textContent.trim() === 'Соответствует') {
      complianceValueEl = valueEl;
      break;
    }
  }

  if (!complianceValueEl) {
    return false;
  }

  // Если уже установлено "Не соответствует", просто заполняем поле
  if (complianceValueEl.textContent.trim() === 'Не соответствует') {
    // Ждем появления поля с retry-логикой
    factTypeInput = null;
    for (let i = 0; i < 20; i++) {
      await new Promise(r => setTimeout(r, 100));
      factTypeInput = document.querySelector('input#fact_counter_type') ||
                     document.querySelector('input._input_gy901_8[name="fact_counter_type"]') ||
                     document.querySelector('input[name="fact_counter_type"]');
      if (factTypeInput && factTypeInput.offsetParent !== null) break;
    }
    
    if (factTypeInput && factTypeInput.offsetParent !== null) {
      await setReactInputValue('input#fact_counter_type', counterTypeText);
      return true;
    }
    return false;
  }

  // Кликаем на элемент "Соответствует" для открытия панели
  complianceValueEl.click();
  await new Promise(r => setTimeout(r, 400));

  // Находим панель с опциями "Не соответствует Соответствует"
  let panel = null;
  for (let i = 0; i < 15; i++) {
    await new Promise(r => setTimeout(r, 100));
    panel = document.querySelector('._panel_1q4rd_19.slide-up-enter-done') ||
            document.querySelector('._panel_1q4rd_19') ||
            document.querySelector('._panel_1i7mm_19.slide-up-enter-done') ||
            document.querySelector('._panel_1i7mm_19');
    if (panel) break;
  }
  
  if (!panel) {
    return false;
  }

  // Ищем опцию "Не соответствует" с классом _option_8fpl2_78
  let notCompliantOption = null;
  const allOptions = panel.querySelectorAll('._option_8fpl2_78, ._option_1oihn_78, [class*="option"]');
  for (const option of allOptions) {
    const span = option.querySelector('span');
    const text = span ? span.textContent.trim() : option.textContent.trim();
    if (text === 'Не соответствует') {
      notCompliantOption = option;
      break;
    }
  }

  if (!notCompliantOption) {
    notCompliantOption = panel.querySelector('._option_8fpl2_78:nth-child(2)') ||
                        panel.querySelector('._option_1oihn_78:nth-child(2)');
  }

  if (!notCompliantOption) {
    return false;
  }

  // Кликаем на опцию "Не соответствует"
  notCompliantOption.click();
  await new Promise(r => setTimeout(r, 400));

  // Ждем появления панели со списком типов счетчиков и "пропускаем" её
  let typeListPanel = null;
  for (let i = 0; i < 20; i++) {
    await new Promise(r => setTimeout(r, 100));
    const allPanels = document.querySelectorAll('._panel_1q4rd_19.slide-up-enter-done, ._panel_1q4rd_19');
    for (const p of allPanels) {
      const panelText = p.textContent.trim();
      if (panelText.length > 50 || panelText.includes('Меркурий') || panelText.includes('CE') || panelText.includes('STAR')) {
        typeListPanel = p;
        break;
      }
    }
    if (typeListPanel) break;
  }

  // "Пропускаем" панель - кликаем над ней
  if (typeListPanel) {
    const panelRect = typeListPanel.getBoundingClientRect();
    const clickX = panelRect.left + panelRect.width / 2;
    const clickY = panelRect.top - 50;
    
    const elementAtPoint = document.elementFromPoint(clickX, clickY);
    if (elementAtPoint && elementAtPoint !== typeListPanel && !typeListPanel.contains(elementAtPoint)) {
      elementAtPoint.click();
    } else {
      document.body.click();
    }
    await new Promise(r => setTimeout(r, 300));
  }

  // Ждем появления поля "Фактический тип ПУ"
  factTypeInput = null;
  for (let i = 0; i < 30; i++) {
    await new Promise(r => setTimeout(r, 100));
    factTypeInput = document.querySelector('input#fact_counter_type') ||
                   document.querySelector('input._input_gy901_8[name="fact_counter_type"]') ||
                   document.querySelector('input[name="fact_counter_type"]');
    if (factTypeInput && factTypeInput.offsetParent !== null) break;
  }

  if (factTypeInput && factTypeInput.offsetParent !== null) {
    await setReactInputValue('input#fact_counter_type', counterTypeText);
    return true;
  }
  
  return false;
}
```

**Ключевые моменты:**

1. **Множественные селекторы**: Используются несколько вариантов селекторов для поиска элементов (id, класс, name) для надежности
2. **Retry-логика**: Ожидание появления элементов с повторными попытками (до 30 попыток)
3. **Проверка видимости**: Используется `offsetParent !== null` для проверки, что элемент видим
4. **"Пропуск" панели**: Клик над панелью со списком типов для её закрытия
5. **Правильная последовательность**: Клик на "Соответствует" → выбор "Не соответствует" → пропуск панели → заполнение поля

**Примечание**: Аналогичный подход используется для работы с полем "Год изготовления ПУ" в функции "Выпуск" (кодовое слово голосового набора). Функция проверяет соответствие года в поле "Год изготовления ПУ -", извлекая значение из тега `<b>`, и при несоответствии автоматически изменяет селект на "Не соответствует" и заполняет поле фактического года изготовления. Подробнее см. раздел 4.2.3 в [PRD.md](./PRD.md#423-распознавание-года-выпуска).

### Функция changeComplianceSelect

Для изменения результата проверки используется функция `changeComplianceSelect`. Функция поддерживает как новые, так и старые селекторы для обратной совместимости:

```javascript
async function changeComplianceSelect(optionText) {
    // Временно блокируем клики на другие селекты
    window._blockingSelectClicks = true;
    
    // Глобальный обработчик для блокировки кликов на другие селекты
    const blockOtherSelects = (e) => {
        if (window._blockingSelectClicks) {
            const target = e.target;
            // Проверяем новые и старые селекторы
            const selectField = target.closest('._selectField_1ydb5_1') || target.closest('._selectField_1mapt_1');
            if (selectField) {
                const label = selectField.querySelector('label');
                const isComplianceField = label && label.textContent.includes('Результат проверки');
                if (!isComplianceField) {
                    e.preventDefault();
                    e.stopPropagation();
                    e.stopImmediatePropagation();
                    return false;
                }
            }
        }
    };
    
    document.addEventListener('click', blockOtherSelects, true);
    document.addEventListener('mousedown', blockOtherSelects, true);
    
    try {
        // Находим селект результата проверки
        const complianceField = findComplianceField();
        if (!complianceField) {
            return false;
        }

        // Пробуем новые селекторы, затем старые
        let container = complianceField.querySelector('._container_8fpl2_1');
        if (!container) {
            container = complianceField.querySelector('._container_1oihn_1');
        }
        if (!container) {
            return false;
        }

        // Сохраняем текущее значение для проверки
        let currentValueEl = complianceField.querySelector('._value_8fpl2_13');
        if (!currentValueEl) {
            currentValueEl = complianceField.querySelector('._value_1oihn_13');
        }
        const currentValue = currentValueEl ? currentValueEl.textContent.trim() : '';

        // Кликаем на контейнер для открытия меню
        container.click();
        await wait(400);

        // Находим панель (новый селектор)
        let panel = document.querySelector('._panel_1q4rd_19.slide-up-enter-done');
        if (!panel) {
            panel = document.querySelector('._panel_1q4rd_19');
        }
        // Fallback на старый селектор
        if (!panel) {
            panel = document.querySelector('._panel_1i7mm_19.slide-up-enter-done');
        }
        if (!panel) {
            panel = document.querySelector('._panel_1i7mm_19');
        }
        if (!panel) return false;

        // Ищем опции (новый селектор)
        const allOptions = panel.querySelectorAll('._option_8fpl2_78, ._option_1oihn_78');
        let option = null;
        
        for (const opt of allOptions) {
            const span = opt.querySelector('span');
            const text = span ? span.textContent.trim() : opt.textContent.trim();
            if (text === optionText) {
                option = opt;
                break;
            }
        }

        if (option) {
            option.click();
            await wait(300);
            
            // Проверяем, что значение изменилось
            let newValueEl = complianceField.querySelector('._value_8fpl2_13');
            if (!newValueEl) {
                newValueEl = complianceField.querySelector('._value_1oihn_13');
            }
            const newValue = newValueEl ? newValueEl.textContent.trim() : '';
            
            if (newValue === optionText || newValue !== currentValue) {
                return true;
            }
        }
        return false;
    } finally {
        // Удаляем обработчики
        document.removeEventListener('click', blockOtherSelects, true);
        document.removeEventListener('mousedown', blockOtherSelects, true);
        
        // Снимаем блокировку после небольшой задержки
        setTimeout(() => {
            window._blockingSelectClicks = false;
        }, 500);
    }
}
```

**Ключевые моменты:**
- Поддержка новых селекторов: `._selectField_1ydb5_1`, `._container_8fpl2_1`, `._value_8fpl2_13`, `._panel_1q4rd_19`, `._option_8fpl2_78`
- Fallback на старые селекторы для обратной совместимости
- Поиск опций по тексту вместо использования nth-child для большей надежности

### Функция findComplianceField

Для поиска поля "Результат проверки" используется функция `findComplianceField`, которая поддерживает новые и старые селекторы:

```javascript
function findComplianceField() {
    // Пробуем новый селектор
    const allFields = document.querySelectorAll('._selectField_1ydb5_1');
    for (let i = 0; i < allFields.length; i++) {
        const field = allFields[i];
        const label = field.querySelector('label');
        if (label && label.textContent.includes('Результат проверки')) {
            return field;
        }
    }
    // Fallback на старый селектор
    const oldFields = document.querySelectorAll('._selectField_1mapt_1');
    for (let i = 0; i < oldFields.length; i++) {
        const field = oldFields[i];
        const label = field.querySelector('label');
        if (label && label.textContent.includes('Результат проверки')) {
            return field;
        }
    }
    return null;
}
```

### Функция updateNextInspectionDate

Для автоматического обновления поля "Дата следующей поверки" при изменении "Дата предыдущей поверки" используется функция `updateNextInspectionDate`:

```javascript
async function updateNextInspectionDate(lastInspectionDate) {
    // Проверяем формат даты
    if (!lastInspectionDate || !/^\d{4}-\d{2}-\d{2}$/.test(lastInspectionDate)) {
        return false;
    }
    
    // Вычисляем дату следующей поверки (+16 лет, но на день раньше)
    const dateParts = lastInspectionDate.split('-');
    if (dateParts.length !== 3) {
        return false;
    }
    
    // Создаем объект Date из исходной даты
    const lastDate = new Date(lastInspectionDate + 'T12:00:00');
    // Добавляем 16 лет
    lastDate.setFullYear(lastDate.getFullYear() + 16);
    // Вычитаем один день
    lastDate.setDate(lastDate.getDate() - 1);
    
    // Форматируем в YYYY-MM-DD
    const year = lastDate.getFullYear();
    const month = String(lastDate.getMonth() + 1).padStart(2, '0');
    const day = String(lastDate.getDate()).padStart(2, '0');
    const nextInspectionDate = `${year}-${month}-${day}`;
    
    // Находим поле "Дата следующей поверки"
    const nextInspectionInput = findNextInspectionInput();
    if (!nextInspectionInput) {
        return false;
    }
    
    // Проверяем, не нужно ли обновлять (если значение уже правильное, не обновляем)
    const currentNextValue = nextInspectionInput.value || nextInspectionInput.getAttribute('value') || '';
    if (currentNextValue === nextInspectionDate) {
        return true;
    }
    
    // Устанавливаем флаг, чтобы избежать бесконечного цикла
    if (window._updatingNextInspectionDate) {
        return false;
    }
    window._updatingNextInspectionDate = true;
    
    try {
        // Устанавливаем значение
        const success = await setDateValue(nextInspectionInput, nextInspectionDate);
        return success;
    } finally {
        // Снимаем флаг после небольшой задержки
        setTimeout(() => {
            window._updatingNextInspectionDate = false;
        }, 500);
    }
}
```

**Ключевые моменты:**
- Автоматически вычисляет дату следующей поверки (+16 лет, но на день раньше)
- Использует объект `Date` для корректной обработки переходов между месяцами и годами
- Использует флаг `_updatingNextInspectionDate` для предотвращения бесконечного цикла
- Проверяет, нужно ли обновлять значение перед установкой
- Использует функцию `setDateValue` для корректной работы с React

## Best Practices

### 1. Всегда используйте функции-обертки
Не пытайтесь напрямую изменять значения React-полей. Всегда используйте:
- `setReactInputValue` для обычных input/textarea полей
- `setReactCheckboxValue` для checkbox полей
- `setDateValue` для полей типа `input[type="date"]`

### 2. Используйте retry-логику
React может асинхронно обновлять состояние. Всегда делайте несколько попыток с задержками.

### 3. Проверяйте результат
После установки значения всегда проверяйте, что оно действительно установилось:
```javascript
if (input.value === String(value)) {
    return true;
}
```

### 4. Используйте silent режим когда нужно
Если изменение значения не должно триггерить другие обработчики (например, workflow автозаполнения), используйте `silent: true`:
```javascript
await setReactInputValue('input#field', 'value', 3, true);
```

### 5. Работайте с событиями правильно
- Всегда используйте `bubbles: true` для событий
- Диспатчите события в правильном порядке: `input` → `change` → `blur`
- Для checkbox также диспатчите `click`

### 6. Используйте MutationObserver для динамических элементов
React может динамически создавать и удалять элементы. Используйте `MutationObserver` для отслеживания появления нужных элементов:
```javascript
const observer = new MutationObserver(() => {
    const input = document.querySelector('input#field');
    if (input) {
        // Элемент появился, можно работать с ним
        doSomething(input);
    }
});
observer.observe(document.body, { childList: true, subtree: true });
```

### 7. Ждите стабилизации значений
Иногда значение может меняться после установки. Используйте функцию `getStableInputValue`:
```javascript
async function getStableInputValue(input, attempts = 5, delay = 50) {
    let lastVal = input.value;
    for (let i = 0; i < attempts; i++) {
        await wait(delay);
        if (input.value === lastVal) return lastVal;
        lastVal = input.value;
    }
    return lastVal;
}
```

### 8. Избегайте прокрутки страницы при работе с полями
При программном обновлении полей не используйте методы, которые могут вызвать прокрутку:
- **Не используйте `input.focus()`** - это может автоматически прокрутить страницу к полю
- **Не используйте `window.scrollTo()`** для восстановления позиции - лучше вообще не менять позицию прокрутки
- **Используйте диспатч событий вместо реальных методов**: вместо `input.blur()` используйте `input.dispatchEvent(new Event('blur', { bubbles: true }))`
- Это особенно важно при обновлении нескольких полей подряд (например, `last_inspection_year` и `next_inspection_year`)

### 9. Автоматическое обновление связанных полей
При изменении поля "Дата предыдущей поверки" автоматически обновляется поле "Дата следующей поверки":
- Используйте `MutationObserver` для отслеживания изменений в поле даты
- Добавляйте обработчики событий `input` и `change` для надежности
- Помечайте программные события флагом `_programmatic`, чтобы избежать бесконечного цикла
- Используйте функцию `updateNextInspectionDate()` для автоматического обновления следующей даты

## Отладка

### Проблема: Значение не устанавливается
1. Проверьте, что элемент существует и доступен
2. Увеличьте количество попыток (`maxRetries`)
3. Увеличьте задержки между попытками
4. Проверьте, не блокирует ли что-то установку значения (другие скрипты, обработчики)

### Проблема: Значение устанавливается, но React не видит изменений
1. Убедитесь, что вы работаете с `_valueTracker`
2. Проверьте, что события диспатчатся с `bubbles: true`
3. Попробуйте добавить больше задержек после диспатча событий

### Проблема: Срабатывают другие обработчики
1. Используйте `silent: true` режим
2. Добавьте флаг `_programmatic` к событиям
3. Используйте `window._blockWorkflow` для блокировки workflow

### Проблема: Дата в date поле не обновляется или обновляется неправильно
1. Используйте `valueAsDate` вместо прямого изменения `value`
2. Используйте `T12:00:00` вместо `T00:00:00` для избежания проблем с часовыми поясами
3. Обновляйте `span` с отформатированной датой вручную несколько раз с задержками
4. Убедитесь, что вы используете правильное поле (ищите по label, а не просто по типу)
5. Проверьте, что поле найдено правильно через `findLastInspectionInput()`

## Примеры использования

### Пример 1: Установка значения в input
```javascript
const success = await setReactInputValue('input#last_inspection_year', '2024');
if (!success) {
    showToast('⚠️ Не удалось установить год', 3000, '#a33');
}
```

### Пример 2: Установка значения в checkbox
```javascript
const success = await setReactCheckboxValue('input#owner_is_absent', true);
if (!success) {
    console.warn('Не удалось установить checkbox');
}
```

### Пример 3: Установка значения без триггера workflow
```javascript
// Silent режим - не триггерит другие обработчики
await setReactInputValue('textarea#inaccordance_reason', 'Истек срок поверки', 3, true);
```

### Пример 4: Работа с динамически появляющимися элементами
```javascript
const observer = new MutationObserver(async () => {
    const yearInput = document.querySelector('input#last_inspection_year');
    if (yearInput && !yearInput.dataset.processed) {
        yearInput.dataset.processed = 'true';
        await setReactInputValue('input#last_inspection_year', '2024');
    }
});
observer.observe(document.body, { childList: true, subtree: true });
```

### Пример 5: Изменение типа счетчика
```javascript
// Изменение типа счетчика с автоматическим открытием селекта и заполнением поля
const success = await changeCounterType('СЕ 101 S6 145 M6');
if (!success) {
    showToast('❌ Не удалось изменить тип счетчика', 3000, '#a33');
}
```

**Важно**: Функция `changeCounterType` автоматически:
- Находит и кликает на селект "Соответствует"
- Выбирает опцию "Не соответствует"
- Пропускает панель со списком типов счетчиков
- Ожидает появления поля `fact_counter_type`
- Заполняет поле через `setReactInputValue`

### Пример 6: Установка даты в поле типа date
```javascript
// Поиск поля "Дата предыдущей поверки"
const input = findLastInspectionInput();
if (input) {
    // Установка даты с сохранением дня и месяца
    const newDate = '2016-05-08'; // Формат YYYY-MM-DD
    const success = await setDateValue(input, newDate);
    if (!success) {
        showToast('❌ Не удалось установить дату', 3000, '#a33');
    }
}
```

**Важно**: Функция `setDateValue`:
- Использует `valueAsDate` как основной метод (лучше всего работает с React)
- Автоматически обновляет отображаемый `span` с датой
- Учитывает часовой пояс (использует `T12:00:00`)
- Работает с React `_valueTracker`
- Обновляет `span` несколько раз с задержками для надежности

### Пример 7: Обновление года в дате с сохранением дня и месяца
```javascript
// Получение текущей даты из поля
const input = findLastInspectionInput();
if (input) {
    let currentDate = input.value;
    
    // Если поле пустое, пытаемся извлечь из span
    if (!currentDate || currentDate.length < 10) {
        const container = input.closest('._inputContainer_27y0k_15');
        if (container) {
            const span = container.querySelector('span');
            if (span) {
                const dateMatch = span.textContent.match(/(\d{2})\.(\d{2})\.(\d{4})/);
                if (dateMatch) {
                    currentDate = `${dateMatch[3]}-${dateMatch[2]}-${dateMatch[1]}`;
                }
            }
        }
    }
    
    // Заменяем только год, сохраняя месяц и день
    const dateParts = currentDate.split('-');
    const newYear = '2016';
    const newDate = `${newYear}-${dateParts[1]}-${dateParts[2]}`;
    
    await setDateValue(input, newDate);
}
```

### Пример 8: Автоматическое обновление даты следующей поверки при изменении даты предыдущей поверки

**Простой способ (использование готовой функции):**
```javascript
// При изменении поля "Дата предыдущей поверки" автоматически обновляем следующую дату
const lastInspectionInput = findLastInspectionInput();
if (lastInspectionInput) {
    const currentDate = lastInspectionInput.value || lastInspectionInput.getAttribute('value') || '';
    if (currentDate && /^\d{4}-\d{2}-\d{2}$/.test(currentDate)) {
        await updateNextInspectionDate(currentDate);
    }
}
```

**С использованием MutationObserver для автоматического отслеживания изменений:**
```javascript
const lastInspectionInput = findLastInspectionInput();
if (lastInspectionInput) {
    let lastObservedValue = lastInspectionInput.value || lastInspectionInput.getAttribute('value') || '';
    
    const dateObserver = new MutationObserver(async (mutations) => {
        // Проверяем изменения в value атрибуте
        for (const mutation of mutations) {
            if (mutation.type === 'attributes' && mutation.attributeName === 'value') {
                const currentValue = lastInspectionInput.value || lastInspectionInput.getAttribute('value') || '';
                if (currentValue && currentValue !== lastObservedValue && /^\d{4}-\d{2}-\d{2}$/.test(currentValue)) {
                    lastObservedValue = currentValue;
                    // Обновляем поле "Дата следующей поверки"
                    await updateNextInspectionDate(currentValue);
                }
            }
        }
    });
    
    // Начинаем наблюдение за изменениями
    dateObserver.observe(lastInspectionInput, {
        attributes: true,
        attributeFilter: ['value'],
        childList: false,
        subtree: false
    });
    
    // Также добавляем обработчик событий для надежности
    const handleDateChange = async (e) => {
        // Пропускаем программные события
        if (e && (e._programmatic || e.isTrusted === false)) {
            return;
        }
        
        const currentValue = lastInspectionInput.value || lastInspectionInput.getAttribute('value') || '';
        if (currentValue && currentValue !== lastObservedValue && /^\d{4}-\d{2}-\d{2}$/.test(currentValue)) {
            lastObservedValue = currentValue;
            await wait(100);
            await updateNextInspectionDate(currentValue);
        }
    };
    
    lastInspectionInput.addEventListener('input', handleDateChange);
    lastInspectionInput.addEventListener('change', handleDateChange);
}
```

**Важно**: 
- Функция `updateNextInspectionDate()` автоматически вычисляет дату следующей поверки (+16 лет) и обновляет поле
- Используется флаг `_updatingNextInspectionDate` для предотвращения бесконечного цикла
- MutationObserver отслеживает изменения атрибута `value` и отображаемого текста в `span`
- Обработчики событий `input` и `change` пропускают программные события (с флагом `_programmatic`)
- При обновлении полей даты скрипт не прокручивает страницу (позиция прокрутки сохраняется)

---

**Важно**: При изменении структуры сайта или обновлении React-версии эти методы могут потребовать корректировки. Всегда тестируйте скрипты после обновлений сайта.

