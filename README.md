Скрипт для массово извлечения данных из Ahrefs Batch Analysis 2.0 без использованя API.

Скрипт дается только в познавательных целях. Автор не несет ответственности за блокировку акков или любые другие проблемы, которые могут возникнуть. 

Автор - https://t.me/seregaseo

Работает через расширение Tampermonkey

Процесс работы:

<img width="2709" height="1725" alt="image" src="https://github.com/user-attachments/assets/cc1617f2-4547-4685-8a15-5f007fd4a64b" />

<img width="900" height="1059" alt="image" src="https://github.com/user-attachments/assets/7bd76c28-15c7-42c1-b817-78304a5a9d6e" />

<img width="3816" height="1664" alt="image" src="https://github.com/user-attachments/assets/5e891849-8f1d-4379-a446-df803b959f23" />

Итог:
<img width="2122" height="362" alt="image" src="https://github.com/user-attachments/assets/900d464e-3bb8-4168-88aa-495197fd582d" />





***

Сам код 

```// ==UserScript==
// @name         Ahrefs Batch Analysis 2.0 Export to CSV (without API KEY) — Firefox-stable
// @namespace    http://tampermonkey.net/
// @version      3.2
// @description  Ahrefs Batch Analysis 2.0 Export to CSV (without API KEY) — fixes for SPA routing & Firefox timings
// @author       Сергей Уткин (https://t.me/seregaseo)
// @match        https://app.ahrefs.com/v2-batch-analysis*
// @match        https://app.ahrefs.com/v2-batch-analysis/
// @match        https://app.ahrefs.com/v2-batch-analysis/report*
// @grant        GM_download
// @grant        GM_setValue
// @grant        GM_getValue
// @grant        window.onurlchange
// ==/UserScript==

(function () {
    'use strict';

    const CONFIG = {
        DOMAINS_PER_BATCH: 50,
        INITIAL_POSITION: { bottom: '20px', right: '40px' },
        CONTAINER_SIZE: { width: '380px', height: '480px' },
        DELAY_BETWEEN_ACTIONS: 2500,        // ↑ запас для Firefox
        WAIT_FOR_TABLE_TIMEOUT: 60000       // ↑ запас для Firefox
    };

    const styles = `
        .custom-container {
            position: fixed;
            bottom: ${CONFIG.INITIAL_POSITION.bottom};
            right: ${CONFIG.INITIAL_POSITION.right};
            width: ${CONFIG.CONTAINER_SIZE.width};
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            border: 2px solid #5a67d8;
            border-radius: 12px;
            box-shadow: 0 8px 24px rgba(0,0,0,0.25);
            z-index: 10000;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            overflow: hidden;
        }
        .custom-header {
            padding: 12px 15px;
            background: rgba(0,0,0,0.2);
            color: white;
            display: flex;
            justify-content: space-between;
            align-items: center;
            cursor: move;
            font-weight: 600;
            font-size: 15px;
            letter-spacing: 0.5px;
        }
        .custom-button {
            background: rgba(255,255,255,0.2);
            border: none;
            color: white;
            cursor: pointer;
            width: 28px;
            height: 28px;
            border-radius: 50%;
            font-size: 20px;
            display: flex;
            align-items: center;
            justify-content: center;
            transition: all 0.3s ease;
        }
        .custom-button:hover {
            background: rgba(255,255,255,0.3);
            transform: scale(1.1);
        }
        .custom-content {
            padding: 15px;
            display: flex;
            flex-direction: column;
            gap: 12px;
            background: white;
        }
        .custom-textarea {
            width: 100%;
            height: 260px;
            resize: none;
            padding: 10px;
            border: 2px solid #e2e8f0;
            border-radius: 8px;
            font-family: 'Courier New', monospace;
            font-size: 13px;
            transition: border-color 0.3s ease;
        }
        .custom-textarea:focus {
            outline: none;
            border-color: #667eea;
        }
        .custom-start-button {
            padding: 12px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-weight: 600;
            font-size: 14px;
            transition: all 0.3s ease;
            box-shadow: 0 4px 12px rgba(102, 126, 234, 0.3);
        }
        .custom-start-button:hover {
            transform: translateY(-2px);
            box-shadow: 0 6px 16px rgba(102, 126, 234, 0.4);
        }
        .custom-stop-button {
            padding: 12px;
            background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
            color: white;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-weight: 600;
            font-size: 14px;
            transition: all 0.3s ease;
            box-shadow: 0 4px 12px rgba(245, 87, 108, 0.3);
        }
        .custom-stop-button:hover {
            transform: translateY(-2px);
            box-shadow: 0 6px 16px rgba(245, 87, 108, 0.4);
        }
        .status-info {
            font-size: 12px;
            color: #4a5568;
            text-align: center;
            padding: 8px;
            background: #f7fafc;
            border-radius: 6px;
            border-left: 4px solid #667eea;
            font-weight: 500;
        }
        .copyright {
            font-size: 12px;
            text-align: center;
            color: #718096;
            padding: 8px 0 0 0;
            border-top: 1px solid #e2e8f0;
        }
        .copyright a {
            color: #667eea;
            text-decoration: none;
            font-weight: 600;
            transition: color 0.3s ease;
        }
        .copyright a:hover {
            color: #764ba2;
        }
    `;

    const styleSheet = document.createElement('style');
    styleSheet.textContent = styles;
    document.head.appendChild(styleSheet);

    // Состояние процесса
    let isProcessing = false;
    let currentBatch = 0;
    let totalDomains = 0;

    function saveTextarea(text) {
        GM_setValue('ahrefsHelperText', text);
    }

    function loadTextarea() {
        return GM_getValue('ahrefsHelperText', '');
    }

    function saveProcessingState(state) {
        GM_setValue('ahrefsProcessingState', JSON.stringify(state));
    }

    function loadProcessingState() {
        const state = GM_getValue('ahrefsProcessingState', null);
        return state ? JSON.parse(state) : null;
    }

    function clearProcessingState() {
        GM_setValue('ahrefsProcessingState', '');
    }

    function setReactTextareaValue(textarea, value) {
        const nativeInputValueSetter = Object.getOwnPropertyDescriptor(
            window.HTMLTextAreaElement.prototype,
            'value'
        ).set;
        nativeInputValueSetter.call(textarea, value);
        const event = new Event('input', { bubbles: true });
        textarea.dispatchEvent(event);
    }

    function waitForElement(selector, timeout = CONFIG.WAIT_FOR_TABLE_TIMEOUT) {
        return new Promise((resolve, reject) => {
            const existing = document.querySelector(selector);
            if (existing) return resolve(existing);

            const observer = new MutationObserver(() => {
                const el = document.querySelector(selector);
                if (el) {
                    observer.disconnect();
                    resolve(el);
                }
            });

            observer.observe(document.body, { childList: true, subtree: true });

            const to = setTimeout(() => {
                observer.disconnect();
                reject(new Error(`Элемент ${selector} не найден за ${timeout}ms`));
            }, timeout);

            // Безопасность: если промис завершился ранее
            Promise.resolve().finally(() => clearTimeout(to));
        });
    }

    // Устойчивое ожидание textarea и кнопки Analyze (исключая свою панель)
    async function waitForAnalyzeControls(timeout = 60000) {
        const deadline = Date.now() + timeout;

        // Ждём textarea (не нашу)
        const textarea = await waitForElement('textarea:not(.custom-textarea)', timeout);

        // Ждём кнопку Analyze вне нашей панели
        async function findAnalyzeBtn() {
            const btn = Array.from(document.querySelectorAll('button[type="button"]'))
                .find(b => b.textContent.trim().toLowerCase() === 'analyze'
                        && !b.closest('.custom-container'));
            return btn || null;
        }

        let analyzeButton = await findAnalyzeBtn();
        while (!analyzeButton && Date.now() < deadline) {
            await new Promise(r => setTimeout(r, 250));
            analyzeButton = await findAnalyzeBtn();
        }
        if (!analyzeButton) throw new Error('Кнопка Analyze не найдена');

        return { textarea, analyzeButton };
    }

    function tableToCSV(table) {
        const rows = [];

        const headers = [
            'Target',
            'Mode',
            'IP',
            'Protocol',
            'URL Rating',
            'Domain Rating',
            'Ahrefs Rank',
            'Organic / Total Keywords',
            'Organic / Keywords (Top 3)',
            'Organic / Keywords (4-10)',
            'Organic / Keywords (11-20)',
            'Organic / Keywords (21-50)',
            'Organic / Keywords (51+)',
            'Organic / Traffic',
            'Organic / Value',
            'Organic / Top Countries',
            'Organic / Location traffic',
            'Paid / Keywords',
            'Paid / Ads',
            'Paid / Traffic',
            'Paid / Cost',
            'Ref. domains / All',
            'Ref. domains / Followed',
            'Ref. domains / Not followed',
            'Ref. IPs / IPs',
            'Ref. IPs / Subnets',
            'Backlinks / All',
            'Backlinks / Followed',
            'Backlinks / Not followed',
            'Backlinks / Redirects',
            'Backlinks / Internal',
            'Outgoing domains / Linked domains',
            'Outgoing domains / Followed',
            'Outgoing links / Outgoing links',
            'Outgoing links / Followed'
        ];

        rows.push(headers.join('\t'));

        const dataRows = table.querySelectorAll('tbody tr');

        dataRows.forEach((tr) => {
            const cells = tr.querySelectorAll('td');

            let isEmpty = true;
            cells.forEach(td => {
                if (td.textContent.trim() !== '') {
                    isEmpty = false;
                }
            });
            if (isEmpty) return;

            const cellsArray = Array.from(cells);
            const row = [];

            cellsArray.forEach((td, index) => {
                let cellText = td.textContent.trim();

                cellText = cellText.replace(/"/g, '""')
                                  .replace(/\n/g, ' ')
                                  .replace(/\s+/g, ' ')
                                  .replace(/,/g, '');

                if (cellText.match(/^\d[\d\s]*[\d)]$/)) {
                    cellText = cellText.replace(/\s/g, '');
                }

                if (index === 0) {
                    // пропускаем
                } else if (index === 1) {
                    cellText = cellText.replace(/\/$/, '');
                    row.push(cellText);       // Target
                    row.push('Subdomains');   // Mode
                } else if (index === 2) {
                    // Protocol в исходной таблице — пропускаем
                } else if (index === 3) {
                    row.push(cellText);       // IP
                    row.push('both');         // Protocol
                } else {
                    row.push(cellText);
                }
            });

            for (let i = 0; i < row.length; i++) {
                if (typeof row[i] === 'string' && row[i].match(/\([a-z]{2},\s*\d+\)/i)) {
                    const match = row[i].match(/\(([a-z]{2}),\s*(\d+)\)/i);
                    if (match) {
                        const countryCode = match[1];
                        const traffic = match[2];
                        row[i] = countryCode;
                        row.splice(i + 1, 0, traffic);
                        break;
                    }
                }
            }

            if (row.length > 1 && row.some((cell, idx) => idx > 0 && cell !== '')) {
                rows.push(row.join('\t'));
            }
        });

        return rows.join('\n');
    }

    function downloadCSV(csvContent, filename) {
        const BOM = '\uFEFF';
        const blob = new Blob([BOM + csvContent], { type: 'text/csv;charset=utf-8;' });
        const url = URL.createObjectURL(blob);
        const link = document.createElement('a');
        link.setAttribute('href', url);
        link.setAttribute('download', filename);
        link.style.visibility = 'hidden';
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
        setTimeout(() => URL.revokeObjectURL(url), 100);
    }

    function getFirstDomainFromBatch(batchDomains) {
        if (!batchDomains || batchDomains.length === 0) return '';
        const firstDomain = batchDomains[0].trim();
        return firstDomain.replace(/^https?:\/\//, '').replace(/\/.*$/, '');
    }

    async function processNextBatch() {
        if (!isProcessing) return;

        const state = loadProcessingState();
        if (!state || state.domains.length === 0) {
            stopProcessing();
            alert('Все домены обработаны!');
            return;
        }

        currentBatch++;
        const batchDomains = state.domains.splice(0, CONFIG.DOMAINS_PER_BATCH);
        const remaining = state.domains.length;
        const firstDomain = getFirstDomainFromBatch(batchDomains);

        updateStatus(`Пакет ${currentBatch}: ${batchDomains.length} доменов (осталось: ${remaining})`);

        saveProcessingState(state);
        saveTextarea(state.domains.join('\n'));

        try {
            // Ждём реальную готовность формы
            const { textarea: inputTextarea, analyzeButton } = await waitForAnalyzeControls(60000);

            setReactTextareaValue(inputTextarea, batchDomains.join('\n'));
            inputTextarea.focus();

            await new Promise(resolve => setTimeout(resolve, CONFIG.DELAY_BETWEEN_ACTIONS));
            analyzeButton.click();

            // Ждём переход на отчёт и таблицу
            await new Promise(resolve => setTimeout(resolve, CONFIG.DELAY_BETWEEN_ACTIONS * 2));
            const table = await waitForElement('table', CONFIG.WAIT_FOR_TABLE_TIMEOUT * 2);

            // Дать время на полную отрисовку
            await new Promise(resolve => setTimeout(resolve, CONFIG.DELAY_BETWEEN_ACTIONS * 2));

            const csvContent = tableToCSV(table);
            const timestamp = new Date().toISOString().slice(0,10).replace(/-/g, '-');
            const filename = `batch_analysis_${firstDomain}_${timestamp}.csv`;

            downloadCSV(csvContent, filename);

            updateStatus(`Пакет ${currentBatch} экспортирован: ${filename}`);

            await new Promise(resolve => setTimeout(resolve, CONFIG.DELAY_BETWEEN_ACTIONS));
            window.location.href = 'https://app.ahrefs.com/v2-batch-analysis/';
        } catch (error) {
            console.error('Ошибка:', error);
            stopProcessing();
            alert(`Ошибка при обработке пакета ${currentBatch}: ${error.message}`);
        }
    }

    function startProcessing() {
        const textarea = document.querySelector('.custom-textarea');
        const lines = textarea.value.trim().split('\n').filter(l => l.trim());

        if (lines.length === 0) {
            alert('Нет доменов для обработки!');
            return;
        }

        const domainRegex = /^(?:[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?\.)+[a-zA-Z]{2,}$/;
        const validDomains = lines.filter(domain => domainRegex.test(domain.trim()));

        if (validDomains.length === 0) {
            alert('Не найдено валидных доменов!');
            return;
        }

        isProcessing = true;
        currentBatch = 0;
        totalDomains = validDomains.length;

        saveProcessingState({
            domains: validDomains,
            total: totalDomains,
            startTime: new Date().toISOString()
        });

        updateStatus(`Начало обработки: ${totalDomains} доменов`);

        const startBtn = document.querySelector('.custom-start-button');
        if (startBtn) {
            startBtn.textContent = 'Остановить';
            startBtn.className = 'custom-stop-button';
            startBtn.onclick = stopProcessing;
        }

        processNextBatch();
    }

    function stopProcessing() {
        isProcessing = false;

        const startBtn = document.querySelector('.custom-stop-button');
        if (startBtn) {
            startBtn.textContent = 'Запустить обработку';
            startBtn.className = 'custom-start-button';
            startBtn.onclick = startProcessing;
        }

        updateStatus('Обработка остановлена');
        clearProcessingState();
    }

    function updateStatus(message) {
        const statusElement = document.querySelector('.status-info');
        if (statusElement) {
            statusElement.textContent = message;
        }
    }

    function initContainer() {
        if (document.querySelector('.custom-container')) return;

        const container = document.createElement('div');
        container.className = 'custom-container';
        container.innerHTML = `
            <div class="custom-header">
                <span>Ahrefs Batch Helper v3.2</span>
                <button class="custom-button close-btn">×</button>
            </div>
            <div class="custom-content">
                <textarea class="custom-textarea" placeholder="Введите домены (по одному на строку)..."></textarea>
                <div class="status-info">Готов к работе</div>
                <button class="custom-start-button">Запустить обработку</button>
                <p class="copyright">TG: <a href="https://t.me/seregaseo" target="_blank">Канал Strong SEO</a></p>
            </div>
        `;
        document.body.appendChild(container);

        const textarea = container.querySelector('.custom-textarea');
        const startBtn = container.querySelector('.custom-start-button');
        const closeBtn = container.querySelector('.close-btn');

        textarea.value = loadTextarea();
        textarea.addEventListener('input', () => saveTextarea(textarea.value));
        closeBtn.addEventListener('click', () => container.remove());

        startBtn.addEventListener('click', startProcessing);

        const savedState = loadProcessingState();
        if (savedState && savedState.domains && savedState.domains.length > 0) {
            if (confirm('Найдена незавершенная обработка. Продолжить?')) {
                resumeIfPending();
            }
        }
    }

    // Резюме по сохранённому состоянию (не требует ввода в нашей textarea)
    function resumeIfPending() {
        const savedState = loadProcessingState();
        if (savedState && savedState.domains && savedState.domains.length > 0) {
            isProcessing = true;
            const stopBtn = document.querySelector('.custom-start-button, .custom-stop-button');
            if (stopBtn) {
                stopBtn.textContent = 'Остановить';
                stopBtn.className = 'custom-stop-button';
                stopBtn.onclick = stopProcessing;
            }
            updateStatus('Возобновление обработки...');
            processNextBatch();
        }
    }

    // Автозапуск при загрузке страницы
    if (document.readyState === 'complete') {
        initContainer();
        resumeIfPending();
    } else {
        window.addEventListener('load', () => {
            initContainer();
            resumeIfPending();
        });
    }

    // Автоматическое возобновление при навигации в SPA
    if (window.onurlchange === null) {
        window.addEventListener('urlchange', (info) => {
            const href = (info && info.url) || window.location.href;
            if (href.includes('/v2-batch-analysis/') && !href.includes('/report')) {
                setTimeout(() => resumeIfPending(), 1500);
            }
        });

        // Проверка текущего URL при инициализации
        const href = window.location.href;
        if (href.includes('/v2-batch-analysis/') && !href.includes('/report')) {
            setTimeout(() => resumeIfPending(), 1500);
        }
    }
})();

````

