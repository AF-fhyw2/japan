<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI 日文單字智能查詢系統</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        ::-webkit-scrollbar { height: 8px; width: 8px; }
        ::-webkit-scrollbar-track { background: #f1f1f1; border-radius: 4px; }
        ::-webkit-scrollbar-thumb { background: #c1c1c1; border-radius: 4px; }
        ::-webkit-scrollbar-thumb:hover { background: #a8a8a8; }
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; }
        
        /* 載入動畫 */
        .loader {
            border: 3px solid #f3f3f3;
            border-top: 3px solid #4f46e5;
            border-radius: 50%;
            width: 20px;
            height: 20px;
            animation: spin 1s linear infinite;
            display: inline-block;
            vertical-align: middle;
            margin-right: 8px;
        }
        @keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
    </style>
</head>
<body class="bg-slate-50 text-slate-800 min-h-screen p-4 md:p-8">

    <div class="max-w-6xl mx-auto">
        <!-- 標題區塊 -->
        <header class="mb-8 text-center md:text-left">
            <div class="inline-block bg-indigo-100 text-indigo-700 px-3 py-1 rounded-full text-sm font-bold mb-3 shadow-sm">AI 驅動</div>
            <h1 class="text-3xl md:text-4xl font-bold text-slate-800 mb-2">🤖 智能日文單字解析器</h1>
            <p class="text-slate-500">輸入任何你**沒看過的全新單字**（可一次輸入多個，用換行或空格隔開），AI 會自動為你查出重音、詞性與動詞分類。</p>
        </header>

        <!-- 查詢輸入區塊 -->
        <div class="bg-white p-6 rounded-xl shadow-sm border border-slate-200 mb-8">
            <div class="flex flex-col gap-4">
                <label for="searchInput" class="font-semibold text-slate-700">✍️ 輸入想查詢的新單字：</label>
                <textarea id="searchInput" rows="3" 
                    class="w-full p-4 text-lg border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 transition-colors bg-slate-50 focus:bg-white"
                    placeholder="例如輸入：&#10;諦めます&#10;冷蔵庫&#10;素晴らしい"></textarea>
                
                <div class="flex justify-between items-center">
                    <span id="statusMessage" class="text-sm font-medium text-slate-500">準備就緒</span>
                    <div class="flex gap-3">
                        <button id="clearBtn" class="px-5 py-2.5 rounded-lg font-medium text-slate-600 bg-slate-100 hover:bg-slate-200 transition-colors">
                            清空
                        </button>
                        <button id="searchBtn" class="px-6 py-2.5 rounded-lg font-bold text-white bg-indigo-600 hover:bg-indigo-700 transition-colors shadow-md flex items-center">
                            <span id="btnText">AI 智能解析</span>
                        </button>
                    </div>
                </div>
            </div>
        </div>

        <!-- 結果表格區塊 -->
        <div class="bg-white rounded-xl shadow-sm border border-slate-200 overflow-hidden">
            <div class="overflow-x-auto">
                <table class="w-full text-left border-collapse min-w-[800px]">
                    <thead>
                        <tr class="bg-slate-100 text-slate-700 border-b border-slate-200">
                            <th class="p-4 font-semibold w-24 text-center">1. 重音</th>
                            <th class="p-4 font-semibold">2. 漢字/單字</th>
                            <th class="p-4 font-semibold">3. 平假名</th>
                            <th class="p-4 font-semibold">4. 中文意思</th>
                            <th class="p-4 font-semibold">5. 詞性/分類</th>
                        </tr>
                    </thead>
                    <tbody id="tableBody" class="divide-y divide-slate-100">
                        <!-- 結果會動態插入在這裡 -->
                        <tr>
                            <td colspan="5" class="p-12 text-center text-slate-400">
                                <div class="text-4xl mb-3">✨</div>
                                <p>尚未查詢任何單字。請在上方輸入新單字並點擊解析！</p>
                            </td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>
    </div>

    <script>
        // 設定 API Key (環境執行時會自動提供)
        const apiKey = "";
        
        // UI 元素
        const searchInput = document.getElementById('searchInput');
        const searchBtn = document.getElementById('searchBtn');
        const clearBtn = document.getElementById('clearBtn');
        const btnText = document.getElementById('btnText');
        const statusMessage = document.getElementById('statusMessage');
        const tableBody = document.getElementById('tableBody');

        // 指數退避重試機制的 Fetch 函數
        async function fetchWithRetry(url, options, maxRetries = 5) {
            const delays = [1000, 2000, 4000, 8000, 16000];
            for (let i = 0; i < maxRetries; i++) {
                try {
                    const response = await fetch(url, options);
                    if (!response.ok) {
                        throw new Error(`HTTP 錯誤! 狀態碼: ${response.status}`);
                    }
                    return await response.json();
                } catch (error) {
                    if (i === maxRetries - 1) throw error;
                    // 等待後重試
                    await new Promise(resolve => setTimeout(resolve, delays[i]));
                }
            }
        }

        // 呼叫 Gemini API 進行單字解析
        async function analyzeWords(wordsText) {
            // 系統指令：設定 AI 的角色與回傳格式標準
            const systemInstruction = `
                你是一個專業的日文老師與辭典工具。
                用戶會給你一個或多個日文單字（或中文請你翻譯成日文）。
                請分析這些單字，並嚴格按照提供的 JSON Schema 格式回傳。
                
                規範：
                1. accent: 重音編號，請加上括號，例如 '[0]', '[3]', '[1]'。
                2. word: 日文漢字或原始寫法。
                3. reading: 對應的平假名讀音。
                4. meaning: 繁體中文的精準解釋。
                5. type: 詞性與分類。如果是動詞，務必標示「動詞 (第 1 類)」等；如果是名詞加上する的動詞，標示「名詞 + する (第 3 類)」；形容詞標示「い形容詞」或「な形容詞」。
            `;

            // API 要求的 JSON Schema 結構
            const generationConfig = {
                responseMimeType: "application/json",
                responseSchema: {
                    type: "OBJECT",
                    properties: {
                        words: {
                            type: "ARRAY",
                            items: {
                                type: "OBJECT",
                                properties: {
                                    accent: { type: "STRING" },
                                    word: { type: "STRING" },
                                    reading: { type: "STRING" },
                                    meaning: { type: "STRING" },
                                    type: { type: "STRING" }
                                },
                                required: ["accent", "word", "reading", "meaning", "type"]
                            }
                        }
                    },
                    required: ["words"]
                }
            };

            const payload = {
                contents: [
                    { parts: [{ text: `請分析以下單字：\n${wordsText}` }] }
                ],
                systemInstruction: {
                    parts: [{ text: systemInstruction }]
                },
                generationConfig: generationConfig
            };

            const url = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`;

            const data = await fetchWithRetry(url, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify(payload)
            });

            // 提取並解析 JSON 字串
            const jsonText = data.candidates?.[0]?.content?.parts?.[0]?.text;
            if (!jsonText) throw new Error("API 未回傳有效內容");
            
            return JSON.parse(jsonText).words;
        }

        // 渲染結果到表格
        function renderResults(wordsData) {
            tableBody.innerHTML = ''; // 清空表格
            
            if (!wordsData || wordsData.length === 0) {
                tableBody.innerHTML = `<tr><td colspan="5" class="p-8 text-center text-red-500">無法解析單字，請換個方式輸入試試。</td></tr>`;
                return;
            }

            wordsData.forEach(item => {
                const row = document.createElement('tr');
                row.className = "hover:bg-indigo-50/30 transition-colors";
                
                // 動態決定標籤顏色
                let typeBadgeClass = "bg-slate-100 text-slate-700";
                if (item.type.includes('動詞')) typeBadgeClass = "bg-blue-100 text-blue-700 border border-blue-200";
                if (item.type.includes('形容詞')) typeBadgeClass = "bg-amber-100 text-amber-700 border border-amber-200";
                if (item.type.includes('副詞')) typeBadgeClass = "bg-purple-100 text-purple-700 border border-purple-200";
                if (item.type === '名詞') typeBadgeClass = "bg-emerald-100 text-emerald-700 border border-emerald-200";

                row.innerHTML = `
                    <td class="p-4 text-center">
                        <span class="inline-block bg-slate-700 text-white text-xs font-bold px-2 py-1 rounded">
                            ${item.accent}
                        </span>
                    </td>
                    <td class="p-4 font-bold text-lg text-slate-800">${item.word}</td>
                    <td class="p-4 text-indigo-600 font-medium">${item.reading}</td>
                    <td class="p-4 text-slate-700">${item.meaning}</td>
                    <td class="p-4">
                        <span class="px-3 py-1 rounded-full text-xs font-semibold ${typeBadgeClass}">
                            ${item.type}
                        </span>
                    </td>
                `;
                tableBody.appendChild(row);
            });
        }

        // 按鈕點擊事件
        searchBtn.addEventListener('click', async () => {
            const query = searchInput.value.trim();
            if (!query) {
                alert("請先輸入想要查詢的單字喔！");
                return;
            }

            // 切換為載入狀態
            searchBtn.disabled = true;
            searchBtn.classList.add('opacity-75', 'cursor-not-allowed');
            btnText.innerHTML = `<span class="loader"></span> 正在呼叫 AI 分析中...`;
            statusMessage.innerHTML = '<span class="text-indigo-600 font-bold">正在連線查閱辭典...</span>';
            
            // 顯示載入中的表格狀態
            tableBody.innerHTML = `
                <tr>
                    <td colspan="5" class="p-16 text-center text-slate-500">
                        <span class="loader" style="border-top-color: #6366f1; width: 30px; height: 30px;"></span>
                        <p class="mt-4 font-medium animate-pulse">正在為您整理單字的平假名、重音與動詞分類，請稍候...</p>
                    </td>
                </tr>
            `;

            try {
                // 呼叫 API
                const parsedWords = await analyzeWords(query);
                // 渲染畫面
                renderResults(parsedWords);
                statusMessage.innerHTML = `<span class="text-emerald-600 font-bold">✅ 成功解析 ${parsedWords.length} 個單字！</span>`;
            } catch (error) {
                console.error(error);
                statusMessage.innerHTML = '<span class="text-red-500 font-bold">❌ 發生錯誤，請稍後再試。</span>';
                tableBody.innerHTML = `
                    <tr>
                        <td colspan="5" class="p-8 text-center text-red-500 font-medium">
                            系統暫時無法連線，或是輸入的格式太複雜。<br>錯誤訊息: ${error.message}
                        </td>
                    </tr>
                `;
            } finally {
                // 恢復按鈕狀態
                searchBtn.disabled = false;
                searchBtn.classList.remove('opacity-75', 'cursor-not-allowed');
                btnText.innerHTML = "AI 智能解析";
            }
        });

        // 清空按鈕
        clearBtn.addEventListener('click', () => {
            searchInput.value = '';
            tableBody.innerHTML = `
                <tr>
                    <td colspan="5" class="p-12 text-center text-slate-400">
                        <div class="text-4xl mb-3">✨</div>
                        <p>已清空。請輸入新單字並點擊解析！</p>
                    </td>
                </tr>
            `;
            statusMessage.textContent = "準備就緒";
            searchInput.focus();
        });
    </script>
</body>
</html>
