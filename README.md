<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>信用卡現金回贈比較工具</title>
    <style>
        body { font-family: 'Microsoft JhengHei', sans-serif; margin: 20px; }
        h1 { text-align: center; }
        form { max-width: 600px; margin: 0 auto; }
        label { display: block; margin: 10px 0 5px; }
        input, select { width: 100%; padding: 8px; margin-bottom: 15px; }
        button { width: 100%; padding: 10px; background-color: #007bff; color: white; border: none; cursor: pointer; }
        button:hover { background-color: #0056b3; }
        table { width: 100%; border-collapse: collapse; margin-top: 20px; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: center; }
        th { background-color: #f2f2f2; }
        .note { font-size: 0.9em; color: #666; margin-top: 10px; }
    </style>
</head>
<body>
    <h1>信用卡現金回贈比較工具</h1>
    <form id="rebateForm">
        <label for="item">購買項目（請選擇）：</label>
        <select id="item" required>
            <option value="">-- 請選擇 --</option>
            <option value="local_retail">本地零售簽賬（非指定類別）</option>
            <option value="local_dining">本地餐飲簽賬（例如：餐廳用餐）</option>
            <option value="online">網上零售簽賬（例如：網上購物）</option>
            <option value="overseas">海外簽賬（非中國內地/澳門，例如：海外購物）</option>
            <option value="china_macau">中國內地/澳門簽賬</option>
            <option value="travel">旅遊相關（例如：機票/酒店）</option>
            <option value="entertainment">娛樂/休閒</option>
            <option value="electronics">電子產品</option>
            <option value="medical">醫療服務</option>
            <option value="pets">寵物生活</option>
            <option value="jewelry">珠寶服飾</option>
            <option value="other">其他</option>
        </select>
        
        <label for="amount">估計金額（HK$）：</label>
        <input type="number" id="amount" min="0" required>
        
        <label for="month_spent">本月已累積簽賬金額（HK$，用於計算上限）：</label>
        <input type="number" id="month_spent" min="0" value="0" required>
        
        <button type="submit">計算回贈</button>
    </form>
    
    <div id="results" style="display: none;">
        <h2>比較結果</h2>
        <table>
            <thead>
                <tr>
                    <th>信用卡</th>
                    <th>估計回贈 (HK$)</th>
                    <th>注意事項（考慮上限）</th>
                </tr>
            </thead>
            <tbody id="resultsTable"></tbody>
        </table>
        <div class="note">注意：計算基於2026年促銷規則簡化版。實際回贈依銀行最新條款及您的登記/累積簽賬而定。紅日/平日、單一簽賬滿額等條件可能影響結果。建議查閱銀行網站確認。</div>
    </div>

    <script>
        document.getElementById('rebateForm').addEventListener('submit', function(e) {
            e.preventDefault();
            const amount = parseFloat(document.getElementById('amount').value);
            const category = document.getElementById('item').value;
            const monthSpent = parseFloat(document.getElementById('month_spent').value);

            // 信用卡規則（簡化版，基於PDF提取）
            const cards = [
                { name: 'DBS Eminent Card', calc: calcDBS, caps: { spec: 8000, other: 20000 } }, // Visa Signature上限
                { name: '中銀Visa (狂賞派)', calc: calcBOC1, caps: { local: 5000 } },
                { name: '中銀Chill Card', calc: calcChill, caps: {} },
                { name: '恒生Travel+ Visa Signature', calc: calcTravelPlus, caps: { total: 500 } }, // +FUN Dollars 上限$500
                { name: '中銀Cheers Card', calc: calcCheers, caps: { dining: 10000, travel: 25000 } }, // Infinite版
                { name: '恒生MMPOWER World Mastercard', calc: calcMMPower, caps: { total: 500 } },
                { name: '中銀信用卡 (狂賞飛)', calc: calcBOCFly, caps: { china: 5000, overseas: 10000 } },
                { name: '富邦iN VISA白金卡', calc: calcFubon, caps: { online: 3125 } } // 網上上限$250 (3125*8%)
            ];

            const resultsTable = document.getElementById('resultsTable');
            resultsTable.innerHTML = '';
            cards.forEach(card => {
                const { rebate, note } = card.calc(amount, category, monthSpent, card.caps);
                const row = `<tr><td>${card.name}</td><td>${rebate.toFixed(2)}</td><td>${note}</td></tr>`;
                resultsTable.innerHTML += row;
            });

            document.getElementById('results').style.display = 'block';
        });

        // 計算函數（簡化，假設紅日5%、平日2%等；積分轉回贈假設1分=0.004HK$）
        function calcDBS(amount, cat, spent, caps) {
            let rate = (['medical', 'pets', 'entertainment', 'travel', 'electronics', 'local_dining', 'jewelry'].includes(cat) && amount >= 300) ? 0.05 : 0.01;
            let cap = rate === 0.05 ? caps.spec : caps.other;
            let effectiveSpent = spent > cap ? cap : spent;
            let remaining = cap - effectiveSpent;
            let rebate = Math.min(amount, remaining) * rate;
            return { rebate, note: `上限考慮：指定類別首${cap}享${rate*100}%` };
        }

        function calcBOC1(amount, cat, spent, caps) {
            if (cat !== 'local_retail' || amount < 500) return { rebate: 0, note: '需滿HK$500單一簽賬及累積滿5000' };
            let rate = Math.random() > 0.5 ? 0.05 : 0.02; // 隨機模擬紅日/平日
            let remaining = caps.local - spent;
            let rebate = Math.min(amount, remaining) * rate;
            return { rebate, note: `紅日5%/平日2%，累積上限5000` };
        }

        function calcChill(amount, cat, spent, caps) {
            let rate = ['local_dining', 'entertainment', 'travel'].includes(cat) ? 0.05 : 0.01; // 簡化
            return { rebate: amount * rate, note: '每月計算，無指定上限' };
        }

        function calcTravelPlus(amount, cat, spent, caps) {
            let rate = (cat === 'china_macau' || cat === 'overseas') ? 0.07 : (cat === 'local_dining') ? 0.05 : 0.004;
            let remaining = caps.total - spent / rate; // +FUN Dollars轉HK$
            let rebate = Math.min(amount, remaining / rate) * rate;
            return { rebate, note: `每月上限$500 +FUN Dollars` };
        }

        function calcCheers(amount, cat, spent, caps) {
            let rate = (cat === 'local_dining') ? 10 / 250 : (cat === 'overseas' || cat === 'travel') ? 10 / 250 : 1 / 250; // 積分轉HK$ (1分=HK$1/250)
            let cap = cat === 'local_dining' ? caps.dining : caps.travel;
            let remaining = cap - spent;
            let rebate = Math.min(amount, remaining) * rate;
            return { rebate, note: `需累積滿5000，積分轉回贈` };
        }

        function calcMMPower(amount, cat, spent, caps) {
            let rate = (cat === 'overseas') ? 0.06 : (cat === 'online') ? 0.05 : (['local_dining', 'electronics', 'entertainment'].includes(cat)) ? 0.01 : 0.004;
            let remaining = caps.total - spent / rate;
            let rebate = Math.min(amount, remaining / rate) * rate;
            return { rebate, note: `每月上限$500 +FUN Dollars` };
        }

        function calcBOCFly(amount, cat, spent, caps) {
            let rate = (cat === 'china_macau') ? 0.06 : (cat === 'overseas') ? 0.03 : 0;
            let cap = cat === 'china_macau' ? caps.china : caps.overseas;
            let remaining = cap - spent;
            let rebate = Math.min(amount, remaining) * rate;
            return { rebate, note: `階段上限HK$600` };
        }

        function calcFubon(amount, cat, spent, caps) {
            let rate = (cat === 'online') ? 0.08 : 0.004;
            let remaining = caps.online - spent;
            let rebate = Math.min(amount, remaining) * rate;
            return { rebate, note: `網上上限HK$250回贈 (首HK$3125簽賬享8%)` };
        }
    </script>
</body>
</html>
