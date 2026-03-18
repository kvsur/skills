---
name: stock-analysis-widget
description: >
  Integrating, configuring, and using the AInvest Stock Analysis widget (AInvestStockAnalysisWidget).
  Use this skill whenever the user wants to: embed stock analysis or diagnostics into a webpage,
  show investment ratings or AI insights for stocks, display fundamental/technical/sentiment analysis
  for US equities, use the Analysis component from AInvest, ask about the widget's props or tab
  configuration, or generate ready-to-run HTML/JS code that renders stock data. Trigger even if the
  user doesn't mention "AInvest" — questions like "show me a stock analysis dashboard for AAPL and
  TSLA" or "embed stock diagnostics on my page" are squarely in scope.
---

# AInvest Stock Analysis Widget

A React UMD component for US stock diagnostics. Loaded from CDN — no npm install needed.

## CDN URLs

```
React 18:      https://unpkg.com/react@18.3.1/umd/react.production.min.js
ReactDOM 18:   https://unpkg.com/react-dom@18.3.1/umd/react-dom.production.min.js
Widget:        https://cdn.ainvest.com/frontResources/static/regular/npm-packages/stock-analysis/analysis-1.0.0.js
```

> **React version constraint**: React/ReactDOM must be `>=18.3.1` and `< 19`. React 19 dropped UMD bundle support, so the widget will fail silently with React 19+.

After loading, the component is available as:
```js
const Analysis = window.AInvestStockAnalysisWidget.Analysis;
```

---

## Props

```typescript
interface Tab {
  label: "Overview" | "Fundamental" | "Technical" | "Sentiment";
  components: Array<
    "Investment-Rating" | "Analysis-Checks" | "AI-Insights" | "Analyst-Rating"
  >;
}

interface StockAnalysisItemConfig {
  ticker: string;       // e.g. "AAPL", "NVDA", "TSLA"
  tabs: Tab[];
}

interface StockAnalysisProps {
  stockAnalysisItems: StockAnalysisItemConfig[];
  AINVEST_OPEN_API_KEY: string;
}
```

---

## Valid components per tab

Each tab only supports specific sub-components. Using an unsupported component in a tab will silently do nothing.

| Tab         | Supported components                                          |
|-------------|---------------------------------------------------------------|
| Overview    | `Investment-Rating`, `AI-Insights`                            |
| Fundamental | `Investment-Rating`, `Analysis-Checks`, `AI-Insights`         |
| Technical   | `Investment-Rating`, `Analysis-Checks`, `AI-Insights`         |
| Sentiment   | `Investment-Rating`, `Analyst-Rating`, `AI-Insights`          |

---

## API Key

The user must supply an `AINVEST_OPEN_API_KEY`. They can create one at:
`https://www.ainvest.com/business/developer-manage/#api-key`

When generating example code, use a clearly fake placeholder like `'YOUR_AINVEST_OPEN_API_KEY'` — never invent a real-looking key.

---

## Ready-to-run HTML template

Use this structure when generating standalone HTML. Always include all three `<script>` tags in order (React → ReactDOM → Widget), and always render via `ReactDOM.createRoot`.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Stock Analysis</title>
  <script crossorigin src="https://unpkg.com/react@18.3.1/umd/react.production.min.js"></script>
  <script crossorigin src="https://unpkg.com/react-dom@18.3.1/umd/react-dom.production.min.js"></script>
  <script src="https://cdn.ainvest.com/frontResources/static/regular/npm-packages/stock-analysis/analysis-1.0.0.js"></script>
</head>
<body>
  <div id="root"></div>
  <script>
    const Analysis = window.AInvestStockAnalysisWidget.Analysis;
    const root = ReactDOM.createRoot(document.getElementById('root'));

    root.render(
      React.createElement(React.StrictMode, null,
        React.createElement(Analysis, {
          stockAnalysisItems: [
            // ← configure stocks here
          ],
          AINVEST_OPEN_API_KEY: 'YOUR_AINVEST_OPEN_API_KEY'
        })
      )
    );
  </script>
</body>
</html>
```

---

## Translating natural language → stockAnalysisItems

When the user describes what they want in plain language (e.g., "show AAPL with full analysis" or "just show NVDA sentiment and ratings"), translate it into the correct JSON config:

- "full analysis" or "everything" → all four tabs with all valid components each
- "overview only" → just the Overview tab
- "fundamentals" → Fundamental tab with `Investment-Rating`, `Analysis-Checks`, `AI-Insights`
- "technical analysis" → Technical tab with `Investment-Rating`, `Analysis-Checks`, `AI-Insights`
- "sentiment" → Sentiment tab with `Investment-Rating`, `Analyst-Rating`, `AI-Insights`
- "AI insights" → include `AI-Insights` in the relevant tabs
- "ratings" → include `Investment-Rating` (and `Analyst-Rating` for Sentiment)

Always validate that the requested components exist in the tab's allowed list (see table above) before including them.

---

## Dynamic loading (non-HTML environments)

If the user is working in a JS/TS app that doesn't use `<script>` tags directly:

```js
function loadScript(src) {
  return new Promise((resolve, reject) => {
    const script = document.createElement('script');
    script.src = src;
    script.onload = resolve;
    script.onerror = reject;
    document.head.appendChild(script);
  });
}

await loadScript('https://unpkg.com/react@18.3.1/umd/react.production.min.js');
await loadScript('https://unpkg.com/react-dom@18.3.1/umd/react-dom.production.min.js');
await loadScript('https://cdn.ainvest.com/frontResources/static/regular/npm-packages/stock-analysis/analysis-1.0.0.js');

const Analysis = window.AInvestStockAnalysisWidget.Analysis;
```

Scripts must load in order (React before ReactDOM, ReactDOM before the widget).

---

## CSS in generated HTML

Do not include a universal CSS reset like `* { margin: 0; padding: 0; box-sizing: border-box; }` in generated HTML unless the user explicitly asks for it. The `*` selector overrides browser defaults globally and can break the widget's own internal styles. Only add styles the user specifically requests.

---

## Troubleshooting

If the widget renders blank or silently fails, walk the user through these checks in order:

1. **Open browser DevTools console** — any error there is the first clue
2. **React version** — confirm the CDN URLs reference `18.x`, not `19.x`. React 19 dropped UMD support and will fail silently
3. **Script load order** — React must load before ReactDOM, ReactDOM before the widget. A reversed or missing tag causes `window.AInvestStockAnalysisWidget` to be undefined
4. **Prop name** — confirm the prop is `stockAnalysisItems`, not `stocks` or `items`
5. **API key** — an invalid or expired key typically renders nothing. Verify at `https://www.ainvest.com/business/developer-manage/#api-key`
6. **Network** — the CDN scripts must be reachable. Check the Network tab for 4xx/5xx on the script URLs
7. **Component names** — they are case-sensitive and hyphenated: `"Investment-Rating"`, `"Analysis-Checks"`, `"AI-Insights"`, `"Analyst-Rating"`. A typo silently drops that component

---

## Common mistakes to avoid

- Using `stocks` as the prop name — the correct prop is `stockAnalysisItems`
- Using React 19 — the widget requires React 18.x
- Including `Analysis-Checks` or `Analyst-Rating` in the Overview tab — neither is supported there
- Including `Analyst-Rating` in Fundamental or Technical tabs — only valid in Sentiment
- Loading scripts in the wrong order or forgetting to load React/ReactDOM before the widget
