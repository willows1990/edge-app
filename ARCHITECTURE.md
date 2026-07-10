const SHEET_DOC_ID = "1mb96Dt3QKUn8WFrWcz2Bs2LSgFTRK6UqEzCjp-hn-wM";
    const SHEET_BASE_URL = `https://docs.google.com/spreadsheets/d/${SHEET_DOC_ID}`;

    const MARKET_FEEDS = {
      wh_shop: {
        shopId: "wh_shop",
        shopName: "William Hill Shop",
        sheetName: "WH_SHOP",
        gid: "1873059466",
        bookCodes: ["wh", "william hill", "williamhill"],
        edges: [
          { edgeId: "wh_shop_horses", edgeName: "Horses", edgeType: "Horses", sheetName: "WH_SHOP", gid: "1873059466" },
          { edgeId: "wh_shop_golf", edgeName: "Golf", edgeType: "Golf", sheetName: "DG_GOLF_PGA", sheetNames: ["DG_GOLF_PGA", "DG_GOLF_DP", "BFEX_GOLF_PGA", "BFEX_GOLF_DP"], dgSheetNames: ["DG_GOLF_PGA", "DG_GOLF_DP"], exSheetNames: ["BFEX_GOLF_PGA", "BFEX_GOLF_DP"], gid: "", dgOnly: false },
          { edgeId: "wh_shop_horse_golf", edgeName: "Horse/Golf", edgeType: "Mixed", mixEdges: ["wh_shop_horses", "wh_shop_golf"] }
        ]
      },
      bf_shop: {
        shopId: "bf_shop",
        shopName: "Betfred Shop",
        sheetName: "BF_SHOP",
        gid: "179235073",
        bookCodes: ["bf", "betfred"],
        edges: [
          { edgeId: "bf_shop_horses", edgeName: "Horses", edgeType: "Horses", sheetName: "BF_SHOP", gid: "179235073" },
          { edgeId: "bf_shop_golf", edgeName: "Golf", edgeType: "Golf", sheetName: "DG_GOLF_PGA", sheetNames: ["DG_GOLF_PGA", "DG_GOLF_DP", "BFEX_GOLF_PGA", "BFEX_GOLF_DP"], dgSheetNames: ["DG_GOLF_PGA", "DG_GOLF_DP"], exSheetNames: ["BFEX_GOLF_PGA", "BFEX_GOLF_DP"], gid: "", dgOnly: false },
          { edgeId: "bf_shop_horse_golf", edgeName: "Horse/Golf", edgeType: "Mixed", mixEdges: ["bf_shop_horses", "bf_shop_golf"] }
        ]
      },
      lads_shop: {
        shopId: "lads_shop",
        shopName: "Ladbrokes Shop",
        sheetName: "LADS_SHOP",
        gid: "1604381861",
        bookCodes: ["sb", "lads", "ladbrokes"],
        edges: [
          { edgeId: "lads_shop_horses", edgeName: "Horses", edgeType: "Horses", sheetName: "LADS_SHOP", gid: "1604381861" },
          { edgeId: "lads_shop_golf", edgeName: "Golf", edgeType: "Golf", sheetName: "DG_GOLF_PGA", sheetNames: ["DG_GOLF_PGA", "DG_GOLF_DP", "BFEX_GOLF_PGA", "BFEX_GOLF_DP"], dgSheetNames: ["DG_GOLF_PGA", "DG_GOLF_DP"], exSheetNames: ["BFEX_GOLF_PGA", "BFEX_GOLF_DP"], gid: "", dgOnly: false },
          { edgeId: "lads_shop_horse_golf", edgeName: "Horse/Golf", edgeType: "Mixed", mixEdges: ["lads_shop_horses", "lads_shop_golf"] }
        ]
      }
    };

    function buildFeedUrls(feedConfig) {
      const sheet = encodeURIComponent(feedConfig.sheetName);
      const urls = [
        `${SHEET_BASE_URL}/gviz/tq?tqx=out:csv&sheet=${sheet}&headers=1`
      ];
      if (feedConfig.gid) {
        const gid = encodeURIComponent(feedConfig.gid);
        urls.push(`${SHEET_BASE_URL}/gviz/tq?tqx=out:csv&gid=${gid}&headers=1`);
        urls.push(`${SHEET_BASE_URL}/export?format=csv&single=true&gid=${gid}`);
      }
      return urls;
    }

    let activeShopId = "wh_shop";
    let activeEdgeId = MARKET_FEEDS[activeShopId].edges[0].edgeId;
    let runners = [];
    let savedSelections = JSON.parse(localStorage.getItem('savedSelections') || '[]');
    let selectionEvMetric = localStorage.getItem('selectionEvMetric') || 'totalEv';
    let selectionGroupOpen = JSON.parse(localStorage.getItem('selectionGroupOpenV2') || '{}');
    let latestSheetHash = "";
    let renderedSheetHash = "";
    let pendingCsvText = "";
    const feedCache = {};

    const fallbackRunners = [
      { horse:"Phantom Watch", meet:"Newmarket", time:"20:15", odds:17, totalEv:115.47, p:3, t:"1/5", winOdds:17, winF:14.5, winEv:117.24, placeOdds:4.2, placeF:3.69, placeEv:113.82 },
      { horse:"Westport", meet:"Royal Ascot", time:"18:10", odds:21, totalEv:107.97, p:5, t:"1/5", winOdds:21, winF:23.5, winEv:89.36, placeOdds:5, placeF:3.95, placeEv:126.58 },
      { horse:"French Duke", meet:"Royal Ascot", time:"15:40", odds:29, totalEv:107.10, p:5, t:"1/5", winOdds:29, winF:41, winEv:70.73, placeOdds:6.6, placeF:4.6, placeEv:143.48 }
    ];

    function getActiveFeedConfig() {
      return MARKET_FEEDS[activeShopId] || MARKET_FEEDS.wh_shop;
    }

    function getActiveEdgeConfig() {
      const feed = getActiveFeedConfig();
      return feed.edges.find(edge => edge.edgeId === activeEdgeId) || feed.edges[0];
    }

    function isGolfEdge() {
      return (getActiveEdgeConfig()?.edgeType || getActiveEdgeConfig()?.edgeName || "").toLowerCase() === "golf";
    }

    function isMixedEdge() {
      return (getActiveEdgeConfig()?.edgeType || getActiveEdgeConfig()?.edgeName || "").toLowerCase() === "mixed";
    }

    function isRunnerGolf(r = {}) {
      const type = String(r.edgeType || r.itemType || r.dataType || r.edgeName || "").toLowerCase();
      return type === "golf" || String(r.dataEdgeName || "").toLowerCase() === "golf" || String(r.edgeId || "").toLowerCase().includes("golf") && !String(r.edgeId || "").toLowerCase().includes("horse_golf");
    }

    function getActiveDataConfig() {
      const feed = getActiveFeedConfig();
      const edge = getActiveEdgeConfig();
      return {
        ...feed,
        ...edge,
        shopId: feed.shopId,
        shopName: feed.shopName,
        bookCodes: feed.bookCodes || [],
        sheetName: edge.sheetName || feed.sheetName,
        sheetNames: edge.sheetNames || null,
        dgSheetNames: edge.dgSheetNames || null,
        exSheetNames: edge.exSheetNames || null,
        gid: edge.gid || ""
      };
    }

    function applyDefaultSortForEdge() {
      const sort = document.getElementById("sortBy");
      if (sort) sort.value = (isGolfEdge() || isMixedEdge()) ? "placeEv" : "totalEv";
    }

    function sourceMetaForActiveFeed() {
      const feed = getActiveFeedConfig();
      const edge = getActiveEdgeConfig();
      return {
        shopId: feed.shopId,
        shopName: feed.shopName,
        sheetName: edge.sheetName || feed.sheetName,
        sheetNames: edge.sheetNames || null,
        dgSheetNames: edge.dgSheetNames || null,
        exSheetNames: edge.exSheetNames || null,
        edgeId: edge.edgeId,
        edgeName: edge.edgeName,
        edgeType: edge.edgeType || edge.edgeName,
        dgOnly: !!edge.dgOnly,
        sourceLabel: `${feed.shopName} - ${edge.edgeName}`
      };
    }

    function shopClassFromId(shopId = "") {
      if (String(shopId).includes("bf")) return "shop-bf";
      if (String(shopId).includes("lads")) return "shop-lads";
      return "shop-wh";
    }

    function shopClassFromName(name = "") {
      const value = String(name).toLowerCase();
      if (value.includes("betfred")) return "shop-bf";
      if (value.includes("ladbrokes")) return "shop-lads";
      return "shop-wh";
    }

    function sourcePartsFromLabel(label = "") {
      const parts = String(label).split(" - ");
      return {
        shopName: parts[0] || "Legacy Feed",
        edgeName: parts.slice(1).join(" - ") || "Horses",
        shopClass: shopClassFromName(parts[0] || "")
      };
    }


    function compFromSheetName(sheetName = "") {
      const value = String(sheetName || "").toUpperCase();
      if (value.endsWith("_PGA") || value.includes("GOLF_PGA")) return "PGA";
      if (value.endsWith("_DP") || value.includes("GOLF_DP")) return "DP";
      return "";
    }

    function escapeHtml(value = "") {
      return String(value ?? "")
        .replace(/&/g, "&amp;")
        .replace(/</g, "&lt;")
        .replace(/>/g, "&gt;")
        .replace(/\"/g, "&quot;")
        .replace(/'/g, "&#39;");
    }

    function selectionGroupInfo(selection = {}) {
      const label = selection.sourceLabel || `${selection.shopName || "Legacy Feed"} - ${selection.edgeName || "Legacy Edge"}`;
      const parts = sourcePartsFromLabel(label);
      const isGolfSelection = String(selection.edgeType || selection.edgeName || "").toLowerCase() === "golf" || String(selection.edgeId || "").toLowerCase().includes("golf");
      const comp = isGolfSelection ? (selection.comp || compFromSheetName(selection.sourceSheetName || selection.sheetName || "")) : "";
      return {
        key: `${parts.shopName} - ${parts.edgeName}${comp ? ` - ${comp}` : ""}`,
        shopName: parts.shopName,
        edgeName: parts.edgeName,
        comp,
        shopClass: selection.shopId ? shopClassFromId(selection.shopId) : parts.shopClass
      };
    }

    function calcEvPercent(odds, fair) {
      const o = Number(odds || 0);
      const f = Number(fair || 0);
      return o > 0 && f > 0 ? (o / f) * 100 : 0;
    }

    function updateShopVisualState() {
      const select = document.getElementById("shopSelect");
      if (!select) return;
      select.classList.remove("shop-wh", "shop-bf", "shop-lads");
      select.classList.add(shopClassFromId(activeShopId));
    }

    function populateEdgeSelect() {
      const edgeSelect = document.getElementById("edgeSelect");
      if (!edgeSelect) return;
      const feed = getActiveFeedConfig();
      if (!feed.edges.some(edge => edge.edgeId === activeEdgeId)) activeEdgeId = feed.edges[0].edgeId;
      edgeSelect.innerHTML = feed.edges.map(edge => `<option value="${edge.edgeId}">${edge.edgeName}</option>`).join("");
      edgeSelect.value = activeEdgeId;
    }

    function parseCSV(text) {
      const rows = [];
      let row = [];
      let cell = "";
      let inQuotes = false;

      for (let i = 0; i < text.length; i++) {
        const char = text[i];
        const next = text[i + 1];

        if (char === '"' && inQuotes && next === '"') {
          cell += '"';
          i++;
        } else if (char === '"') {
          inQuotes = !inQuotes;
        } else if (char === "," && !inQuotes) {
          row.push(cell);
          cell = "";
        } else if ((char === "\n" || char === "\r") && !inQuotes) {
          if (char === "\r" && next === "\n") i++;
          row.push(cell);
          rows.push(row);
          row = [];
          cell = "";
        } else {
          cell += char;
        }
      }

      if (cell || row.length) {
        row.push(cell);
        rows.push(row);
      }

      return rows;
    }

    function simpleHash(text) {
      let hash = 0;
      for (let i = 0; i < text.length; i++) {
        hash = ((hash << 5) - hash) + text.charCodeAt(i);
        hash |= 0;
      }
      return String(hash);
    }

    let refreshCooldownUntil = 0;
    let refreshCooldownTimer = null;

    function updateRefreshButton() {
      const btn = document.getElementById("refreshBtn");
      if (!btn) return;

      const remainingMs = Math.max(0, refreshCooldownUntil - Date.now());
      const isCoolingDown = remainingMs > 0;
      btn.disabled = isCoolingDown;
      btn.classList.toggle("refresh-disabled", isCoolingDown);
      btn.classList.toggle("refresh-ready", !isCoolingDown);
      btn.title = isCoolingDown
        ? `Refresh available in ${Math.ceil(remainingMs / 1000)}s`
        : "Refresh current visible feed";
    }

    function startRefreshCooldown(ms = 60000) {
      refreshCooldownUntil = Date.now() + ms;
      updateRefreshButton();
      if (refreshCooldownTimer) clearInterval(refreshCooldownTimer);
      refreshCooldownTimer = setInterval(() => {
        if (Date.now() >= refreshCooldownUntil) {
          clearInterval(refreshCooldownTimer);
          refreshCooldownTimer = null;
          refreshCooldownUntil = 0;
        }
        updateRefreshButton();
      }, 1000);
    }

    function setRefreshState() {
      updateRefreshButton();
    }

    function cleanNumber(value) {
      if (value === null || value === undefined) return 0;
      return Number(String(value).replace("%", "").replace("£", "").replace(/,/g, "").trim()) || 0;
    }

    function mapSheetRows(csvText, sourceMeta = sourceMetaForActiveFeed()) {
      const rows = parseCSV(csvText).filter(r => r.some(c => String(c).trim() !== ""));
      const headers = (rows[0] || []).map(h => String(h || "").toLowerCase().trim());
      const dataRows = rows.slice(1);
      const activeFeed = getActiveFeedConfig();
      const edgeType = String(sourceMeta.edgeType || sourceMeta.edgeName || "Horses").toLowerCase();
      const isGolf = edgeType === "golf";

      const idx = (...terms) => headers.findIndex(h => terms.some(term => h.includes(term)));
      const val = (row, indexes, fallback = "") => {
        for (const i of indexes) {
          if (i >= 0 && row[i] !== undefined && String(row[i]).trim() !== "") return row[i];
        }
        return fallback;
      };
      const validGolfPlaces = [5, 8, 10, 12];
      const golfPlacesFromRow = (row, primaryIdx) => {
        const primary = cleanNumber(val(row, [primaryIdx], 0));
        if (validGolfPlaces.includes(primary)) return primary;
        for (let i = 0; i < row.length; i++) {
          if (i === primaryIdx) continue;
          const raw = String(row[i] || "").trim();
          if (!/^\d+(?:\.0+)?$/.test(raw)) continue;
          const n = cleanNumber(raw);
          if (validGolfPlaces.includes(n)) return n;
        }
        return 0;
      };

      if (isGolf) {
        const sheetNameLower = String(sourceMeta.sheetName || "").toLowerCase();
        const isExchangeGolfSheet = sheetNameLower.includes("bfex_golf") || sheetNameLower.includes("ex_golf");

        if (isExchangeGolfSheet) {
          const bookCodes = (activeFeed.bookCodes || []).map(c => String(c).toLowerCase());
          const output = [];
          let currentEvent = "";
          let currentMarketMatched = "";

          rows.forEach((row, rowIndex) => {
            const first = String(row[0] || "").trim();
            const helper = String(row[9] || "").trim();
            const winOdds = cleanNumber(row[1]);
            const placeOdds = cleanNumber(row[4]);
            const totalPct = cleanNumber(row[7]);
            const selectionMatched = String(row[8] || "").trim();
            const lowerFirst = first.toLowerCase();

            // EX master can contain stacked event blocks. A block title has a name in Col A and market matched in I1.
            if (first && !winOdds && !placeOdds && !helper && !["player", "player name", "name"].includes(lowerFirst)) {
              currentEvent = first;
              currentMarketMatched = selectionMatched;
              return;
            }

            if (rowIndex < 2 || !first || !helper || !winOdds || !placeOdds) return;
            if (["player", "player name", "name"].includes(lowerFirst)) return;

            const placesMatch = helper.match(/(5|8|10|12)/);
            const places = placesMatch ? Number(placesMatch[1]) : 0;
            if (!places) return;
            const terms = places === 5 ? "1/4" : "1/5";

            output.push({
              ...sourceMeta,
              sourceSheetName: sourceMeta.sheetName || "",
              comp: compFromSheetName(sourceMeta.sheetName || ""),
              dataSource: "ex",
              exOn: true,
              dgOn: false,
              lastUpdated: "",
              horse: first,
              meet: currentEvent || String((rows[0] && rows[0][0]) || "").trim(),
              time: "",
              bookie: helper,
              marketMatched: currentMarketMatched || String((rows[0] && rows[0][8]) || "").trim(),
              selectionMatched,
              totalEv: totalPct,
              odds: winOdds,
              winOdds: winOdds,
              winF: cleanNumber(row[2]),
              winEv: cleanNumber(row[3]),
              placeOdds: placeOdds,
              placeF: cleanNumber(row[5]),
              placeEv: cleanNumber(row[6]),
              p: places,
              t: terms
            });
          });

          return output
            .filter(r => r.horse && r.odds > 0 && r.p)
            .filter(r => {
              if (!bookCodes.length) return true;
              const b = String(r.bookie || "").toLowerCase();
              return bookCodes.some(code => b === code || b.includes(code));
            })
            .sort((a, b) => b.placeEv - a.placeEv);
        }

        const playerIdx = idx("player", "golfer", "name");
        const eventIdx = idx("event", "tournament");
        const bookIdx = idx("book", "shop", "firm", "bookie");
        const placesIdx = idx("places", "place count", "ew places");
        const oddsIdx = idx("odds", "win odds", "price");
        const winFairIdx = idx("bfex w/fair", "win fair", "bfex", "w/fair");
        const winRatingIdx = idx("win %", "win rating", "win ev", "win edge");
        const placeOddsIdx = idx("place odds", "top odds", "pl odds");
        const placeFairIdx = idx("top fair", "place fair", "pl fair");
        const placeRatingIdx = idx("place rating", "place %", "place ev", "place edge", "place rating %");
        const totalRatingIdx = idx("total rating", "total %", "total ev", "total edge", "total rating %");
        const dgIdx = idx("dg", "data golf");
        const bookCodes = (activeFeed.bookCodes || []).map(c => String(c).toLowerCase());

        return dataRows
          .map(row => {
            const bookieRaw = String(val(row, [bookIdx, 2], "")).trim();
            const places = cleanNumber(val(row, [placesIdx, 3], 0));
            const terms = places === 5 ? "1/4" : ([8, 10, 12].includes(places) ? "1/5" : "");
            const winOdds = cleanNumber(val(row, [oddsIdx, 4], 0));
            const winFair = cleanNumber(val(row, [winFairIdx, 5], 0));
            const winRating = cleanNumber(val(row, [winRatingIdx, 6], 0));
            const placeOdds = cleanNumber(val(row, [placeOddsIdx, 7], 0));
            const placeFair = cleanNumber(val(row, [placeFairIdx, 8], 0));
            const placeRating = cleanNumber(val(row, [placeRatingIdx, 9], 0));
            const totalRating = cleanNumber(val(row, [totalRatingIdx, 10], 0));
            const dgValue = String(val(row, [dgIdx], sourceMeta.dgOnly ? "on" : "")).toLowerCase();
            const isDgSheet = String(sourceMeta.sheetName || "").toLowerCase().includes("dg_golf");
            return {
              ...sourceMeta,
              sourceSheetName: sourceMeta.sheetName || "",
              comp: compFromSheetName(sourceMeta.sheetName || ""),
              dataSource: "dg",
              exOn: false,
              lastUpdated: row[13] || row[0] || "",
              horse: String(val(row, [playerIdx, 1], "")).trim(),
              meet: String(val(row, [eventIdx, 0], "")).trim(),
              time: "",
              bookie: bookieRaw,
              dgOn: isDgSheet || sourceMeta.dgOnly || dgValue === "on" || dgValue === "yes" || dgValue === "true" || dgValue === "1",
              totalEv: totalRating,
              odds: winOdds,
              winOdds: winOdds,
              winF: winFair,
              winEv: winRating,
              placeOdds: placeOdds,
              placeF: placeFair,
              placeEv: placeRating,
              p: places,
              t: terms
            };
          })
          .filter(r => r.horse && r.odds > 0)
          .filter(r => {
            if (!bookCodes.length) return true;
            const b = String(r.bookie || "").toLowerCase();
            return bookCodes.some(code => b === code || b.includes(code));
          })
          .sort((a, b) => b.placeEv - a.placeEv);
      }

      return dataRows
        .map(row => ({
          ...sourceMeta,
          sourceSheetName: sourceMeta.sheetName || "",
          comp: "",
          lastUpdated: row[0] || "",
          horse: row[1] || "",
          meet: row[2] || "",
          time: row[3] || "",
          bookie: row[4] || "",
          totalEv: cleanNumber(row[5]),
          odds: cleanNumber(row[6]),
          winOdds: cleanNumber(row[6]),
          winF: cleanNumber(row[7]),
          placeF: cleanNumber(row[8]),
          p: cleanNumber(row[9]),
          t: row[10] ? `1/${cleanNumber(row[10])}` : "",
          placeOdds: cleanNumber(row[11]),
          winEv: cleanNumber(row[12]),
          placeEv: cleanNumber(row[13])
        }))
        .filter(r => r.horse && r.odds > 0)
        .sort((a, b) => b.totalEv - a.totalEv);
    }

    function getLastUpdatedFromSheet(csvText, edgeType = "Horses") {
      const rows = parseCSV(csvText).filter(r => r.some(c => String(c).trim() !== ""));
      if (String(edgeType).toLowerCase() === "golf") {
        // Golf sheets store the true source refresh timestamp in N2.
        return rows[1] && rows[1][13] ? String(rows[1][13]).trim() : "";
      }
      return rows[1] && rows[1][0] ? String(rows[1][0]).trim() : "";
    }

    async function fetchSingleSheetCsv(feedConfig) {
      const urls = buildFeedUrls(feedConfig);
      let lastError = null;

      for (const url of urls) {
        try {
          const separator = url.includes("?") ? "&" : "?";
          const freshUrl = url + separator + "_livePull=" + Date.now() + "_" + Math.random().toString(36).slice(2);
          const response = await fetch(freshUrl, {
            cache: "no-store",
            mode: "cors",
            headers: {
              "Accept": "text/csv,text/plain,*/*",
              "Cache-Control": "no-cache, no-store, max-age=0",
              "Pragma": "no-cache"
            }
          });
          if (!response.ok) throw new Error(`${feedConfig.shopName || feedConfig.sheetName} sheet fetch failed: ${response.status}`);
          const text = await response.text();
          if (!text || !text.trim()) throw new Error(`${feedConfig.shopName || feedConfig.sheetName} sheet returned blank CSV`);
          if (/^\s*</.test(text)) throw new Error(`${feedConfig.shopName || feedConfig.sheetName} returned HTML, not CSV`);
          return text;
        } catch (error) {
          lastError = error;
          console.warn("Sheet fetch route failed:", url, error);
        }
      }

      throw lastError || new Error(`${feedConfig.shopName || feedConfig.sheetName} sheet fetch failed`);
    }

    function csvRowsToText(rows) {
      return rows.map(row => row.map(cell => {
        const value = String(cell ?? "");
        return /[",\n\r]/.test(value) ? `"${value.replace(/"/g, '""')}"` : value;
      }).join(",")).join("\n");
    }

    async function fetchSheetCsv(feedConfig = getActiveFeedConfig()) {
      const sheetNames = Array.isArray(feedConfig.sheetNames) && feedConfig.sheetNames.length
        ? feedConfig.sheetNames
        : null;

      if (!sheetNames) return fetchSingleSheetCsv(feedConfig);

      const allRows = [];
      for (const sheetName of sheetNames) {
        try {
          const csv = await fetchSingleSheetCsv({ ...feedConfig, sheetName, gid: "" });
          const rows = parseCSV(csv).filter(r => r.some(c => String(c).trim() !== ""));
          if (!rows.length) continue;
          if (!allRows.length) allRows.push(rows[0]);
          allRows.push(...rows.slice(1));
        } catch (error) {
          console.warn("Optional sheet skipped:", sheetName, error);
        }
      }

      if (!allRows.length) throw new Error(`${feedConfig.shopName || feedConfig.sheetName} sheets returned no rows`);
      return csvRowsToText(allRows);
    }



    async function fetchMappedRows(dataConfig, meta) {
      let sheetNames = Array.isArray(dataConfig.sheetNames) && dataConfig.sheetNames.length
        ? dataConfig.sheetNames
        : null;

      const edgeTypeLower = String(meta.edgeType || meta.edgeName || "").toLowerCase();
      if (edgeTypeLower === "golf") {
        const dgMode = String(isMixedEdge() ? getAppliedFilters("golf").dg : getAppliedFilters("golf").dg || document.getElementById("dgFilter")?.value || "on").toLowerCase();
        const dgSheets = Array.isArray(dataConfig.dgSheetNames) && dataConfig.dgSheetNames.length ? dataConfig.dgSheetNames : null;
        const exSheets = Array.isArray(dataConfig.exSheetNames) && dataConfig.exSheetNames.length ? dataConfig.exSheetNames : null;
        if (dgMode === "on" && dgSheets) sheetNames = dgSheets;
        if (dgMode === "off" && exSheets) sheetNames = exSheets;
      }

      if (!sheetNames) {
        const csv = await fetchSingleSheetCsv(dataConfig);
        return {
          rows: mapSheetRows(csv, meta),
          csvText: csv,
          lastUpdated: getLastUpdatedFromSheet(csv, meta.edgeType)
        };
      }

      const allRows = [];
      const csvParts = [];
      const latestParts = [];
      const latestValues = [];

      for (const sheetName of sheetNames) {
        try {
          const sheetConfig = { ...dataConfig, sheetName, sheetNames: null, gid: "" };
          const sheetMeta = { ...meta, sheetName, sheetNames: null };
          const csv = await fetchSingleSheetCsv(sheetConfig);
          const mapped = mapSheetRows(csv, sheetMeta);
          if (mapped.length) {
            allRows.push(...mapped);
            csvParts.push(csv);
            const lu = getLastUpdatedFromSheet(csv, sheetMeta.edgeType);
            if (lu) {
              latestParts.push(lu);
              latestValues.push(lu);
            }
          }
        } catch (error) {
          console.warn("Optional sheet skipped:", sheetName, error);
        }
      }

      const uniqueRows = [];
      const seenRows = new Set();
      allRows.forEach(row => {
        const key = [row.dataSource || "", row.shopId || "", row.edgeId || "", row.horse || "", row.meet || "", row.time || "", row.p || "", row.bookie || "", row.winOdds || "", row.winF || "", row.placeOdds || "", row.placeF || "", row.totalEv || ""].join("|").toLowerCase();
        if (seenRows.has(key)) return;
        seenRows.add(key);
        uniqueRows.push(row);
      });

      return {
        rows: uniqueRows,
        csvText: csvParts.join("\n---SHEET---\n"),
        lastUpdated: latestValues.length ? latestValues[0] : (latestParts[0] || "")
      };
    }
    let marketLoadSeq = 0;

    async function loadLiveData() {
      const loadSeq = ++marketLoadSeq;
      const feedConfig = getActiveFeedConfig();
      const edgeConfig = getActiveEdgeConfig();
      const dataConfig = getActiveDataConfig();
      const meta = sourceMetaForActiveFeed();
      populateEdgeSelect();

      runners = [];
      latestSheetHash = "";
      renderedSheetHash = "";
      pendingCsvText = "";
      document.querySelector(".updated").textContent = `${meta.sourceLabel} - loading...`;
      document.getElementById("runnerCount").textContent = 0;
      document.getElementById("meetingCount").textContent = 0;
      populateFilters();
      render();
      renderBetMaker();

      try {
        if (String(edgeConfig.edgeType || "").toLowerCase() === "mixed") {
          const combinedRows = [];
          const latestParts = [];
          const csvParts = [];
          const mixEdges = (edgeConfig.mixEdges || []).map(id => feedConfig.edges.find(edge => edge.edgeId === id)).filter(Boolean);

          for (const mixEdge of mixEdges) {
            const mixDataConfig = {
              ...feedConfig,
              ...mixEdge,
              shopId: feedConfig.shopId,
              shopName: feedConfig.shopName,
              bookCodes: feedConfig.bookCodes || [],
              sheetName: mixEdge.sheetName || feedConfig.sheetName,
              sheetNames: mixEdge.sheetNames || null,
              dgSheetNames: mixEdge.dgSheetNames || null,
              exSheetNames: mixEdge.exSheetNames || null,
              gid: mixEdge.gid || ""
            };
            const mixMeta = {
              ...meta,
              sheetName: mixEdge.sheetName || feedConfig.sheetName,
              sheetNames: mixEdge.sheetNames || null,
              dgSheetNames: mixEdge.dgSheetNames || null,
              exSheetNames: mixEdge.exSheetNames || null,
              edgeId: edgeConfig.edgeId,
              edgeName: edgeConfig.edgeName,
              edgeType: mixEdge.edgeType || mixEdge.edgeName,
              dataEdgeName: mixEdge.edgeName,
              dgOnly: !!mixEdge.dgOnly,
              sourceLabel: `${feedConfig.shopName} - ${edgeConfig.edgeName}`
            };
            const mixResult = await fetchMappedRows(mixDataConfig, mixMeta);
            csvParts.push(mixResult.csvText);
            const mapped = mixResult.rows.map(row => ({
              ...row,
              edgeId: edgeConfig.edgeId,
              edgeName: edgeConfig.edgeName,
              sourceLabel: `${feedConfig.shopName} - ${edgeConfig.edgeName}`,
              dataEdgeName: mixEdge.edgeName,
              edgeType: mixEdge.edgeType || mixEdge.edgeName
            }));
            combinedRows.push(...mapped);
            if (mixResult.lastUpdated) latestParts.push(`${mixEdge.edgeName}: ${mixResult.lastUpdated}`);
          }

          if (loadSeq !== marketLoadSeq || activeShopId !== feedConfig.shopId || activeEdgeId !== edgeConfig.edgeId) return;
          if (!combinedRows.length) throw new Error("No rows found");

          runners = combinedRows;
          latestSheetHash = simpleHash(csvParts.join("\n---MIX---\n"));
          renderedSheetHash = latestSheetHash;
          pendingCsvText = csvParts.join("\n---MIX---\n");
          const lastUpdatedFromA2 = latestParts.join(" | ");
          feedCache[`${feedConfig.shopId}|${edgeConfig.edgeId}`] = {
            rows: runners,
            latestSheetHash,
            renderedSheetHash,
            pendingCsvText,
            lastUpdatedFromA2
          };

          document.querySelector(".updated").textContent = lastUpdatedFromA2 ? "Last Updated: " + lastUpdatedFromA2 : `Last Updated: ${meta.sourceLabel}`;
          populateFilters();
          render();
          renderSelections();
          renderBetMaker();
          updateRefreshButton();
          return;
        }

        const mappedResult = await fetchMappedRows(dataConfig, meta);
        const csv = mappedResult.csvText;
        const newHash = simpleHash(csv);

        if (loadSeq !== marketLoadSeq || activeShopId !== feedConfig.shopId || activeEdgeId !== edgeConfig.edgeId) return;

        runners = mappedResult.rows;
        if (!runners.length) throw new Error("No rows found");

        latestSheetHash = newHash;
        renderedSheetHash = newHash;
        pendingCsvText = csv;
        const lastUpdatedFromA2 = mappedResult.lastUpdated;
        feedCache[`${feedConfig.shopId}|${edgeConfig.edgeId}`] = {
          rows: runners,
          latestSheetHash,
          renderedSheetHash,
          pendingCsvText,
          lastUpdatedFromA2
        };

        document.querySelector(".updated").textContent = lastUpdatedFromA2 ? "Last Updated: " + lastUpdatedFromA2 : `Last Updated: ${meta.sourceLabel}`;

        populateFilters();
        render();
        renderSelections();
        renderBetMaker();
        updateRefreshButton();
      } catch (error) {
        if (loadSeq !== marketLoadSeq) return;
        console.warn("Live sheet failed:", error);
        runners = [];
        document.getElementById("runnerCount").textContent = 0;
        document.getElementById("meetingCount").textContent = 0;
        document.querySelector(".updated").textContent = `${meta.sourceLabel} - no data available yet`;
        applyDefaultSortForEdge();
        populateFilters();
        render();
        renderBetMaker();
        updateRefreshButton();
      }
    }

    async function checkForNewData() {
      return;
    }

    const feed = document.getElementById("feed");
    const filters = document.getElementById("filters");
    const meetFilter = document.getElementById("meetFilter");
    const timeFilter = document.getElementById("timeFilter");

    function signedMoney(v) {
      const n = Number(v || 0);
      const sign = n < 0 ? "-" : "";
      return sign + "£" + Math.abs(n).toFixed(2);
    }

    function money(n) {
      return Number(n).toFixed(2);
    }

    function percent(n) {
      return Number(n).toFixed(2) + "%";
    }

    function evStyle(n) {
      n = Number(n);
      if (n < 100) return "";
      const ranges = [
        [100, 105, 0.20, 0.30],
        [105, 110, 0.20, 0.32],
        [110, 120, 0.22, 0.36],
        [120, 150, 0.31, 0.38],
        [150, 175, 0.39, 0.46],
        [175, 250, 0.47, 0.55]
      ];
      const range = ranges.find(([min, max]) => n >= min && n < max) || ranges[ranges.length - 1];
      const [, max, low, high] = range;
      const min = range[0];
      const t = Math.max(0, Math.min(1, (n - min) / (max - min)));
      const opacity = (low + (high - low) * t).toFixed(3);
      return `--ev-alpha:${opacity}`;
    }

    function evClass(n) {
      n = Number(n);
      if (n < 100) return "ev-grey";
      if (n >= 175) return "ev-maroon-3";
      if (n >= 150) return "ev-maroon-2";
      if (n >= 120) return "ev-maroon-1";
      if (n >= 110) return "ev-purple";
      if (n >= 105) return "ev-green";
      return "ev-yellow";
    }

    const FILTER_DEFAULTS = { meet: "", time: "", minOdds: 1, maxOdds: 41, minEv: 0, minPlaceEv: 0, dg: "on" };
    const GOLF_FILTER_DEFAULTS = { meet: "", time: "", minOdds: 1, maxOdds: 1001, minEv: 0, minPlaceEv: 0, dg: "on" };
    const savedFilterState = JSON.parse(localStorage.getItem("marketFilterStateByEdgeType") || "{}");
    const workingFilterState = JSON.parse(JSON.stringify(savedFilterState));
    let activeMixedFilterTab = "horse";

    function currentFilterKey(kind = null) {
      const edgeName = (getActiveEdgeConfig()?.edgeName || "Horses").trim() || "Horses";
      if (isMixedEdge()) return `${edgeName}:${kind || activeMixedFilterTab || "horse"}`;
      return edgeName;
    }

    function currentFilterDefaults(kind = null) {
      const target = isMixedEdge() ? (kind || activeMixedFilterTab || "horse") : (isGolfEdge() ? "golf" : "horse");
      return target === "golf" ? GOLF_FILTER_DEFAULTS : FILTER_DEFAULTS;
    }

    function normaliseFilters(filters = {}, kind = null) {
      const defaults = currentFilterDefaults(kind);
      return {
        meet: filters.meet || "",
        time: filters.time || "",
        minOdds: Number(filters.minOdds ?? defaults.minOdds),
        maxOdds: Number(filters.maxOdds ?? defaults.maxOdds),
        minEv: Number(filters.minEv ?? defaults.minEv),
        minPlaceEv: Number(filters.minPlaceEv ?? defaults.minPlaceEv),
        dg: filters.dg || defaults.dg
      };
    }

    function filtersEqual(a, b) {
      a = normaliseFilters(a);
      b = normaliseFilters(b);
      return a.meet === b.meet && a.time === b.time &&
        Number(a.minOdds) === Number(b.minOdds) &&
        Number(a.maxOdds) === Number(b.maxOdds) &&
        Number(a.minEv) === Number(b.minEv) &&
        Number(a.minPlaceEv) === Number(b.minPlaceEv) &&
        String(a.dg || "on") === String(b.dg || "on");
    }

    function getAppliedFilters(kind = null) {
      const key = currentFilterKey(kind);
      const defaults = currentFilterDefaults(kind);
      savedFilterState[key] = normaliseFilters(savedFilterState[key] || defaults, kind);
      if ((isGolfEdge() || (isMixedEdge() && (kind || activeMixedFilterTab) === "golf")) && Number(savedFilterState[key].maxOdds) === 41) savedFilterState[key].maxOdds = GOLF_FILTER_DEFAULTS.maxOdds;
      return savedFilterState[key];
    }

    function getWorkingFilters(kind = null) {
      const key = currentFilterKey(kind);
      workingFilterState[key] = normaliseFilters(workingFilterState[key] || getAppliedFilters(kind), kind);
      if ((isGolfEdge() || (isMixedEdge() && (kind || activeMixedFilterTab) === "golf")) && Number(workingFilterState[key].maxOdds) === 41) workingFilterState[key].maxOdds = GOLF_FILTER_DEFAULTS.maxOdds;
      return workingFilterState[key];
    }

    function readFilterInputs() {
      return normaliseFilters({
        meet: meetFilter.value,
        time: timeFilter.value,
        minOdds: document.getElementById("minOdds").value,
        maxOdds: document.getElementById("maxOdds").value,
        minEv: document.getElementById("minEv").value,
        minPlaceEv: document.getElementById("minPlaceEv").value,
        dg: document.getElementById("dgFilter")?.value || "on"
      }, isMixedEdge() ? activeMixedFilterTab : null);
    }

    function writeFilterInputs(filters) {
      filters = normaliseFilters(filters);
      if ([...meetFilter.options].some(o => o.value === filters.meet)) meetFilter.value = filters.meet; else meetFilter.value = "";
      if ([...timeFilter.options].some(o => o.value === filters.time)) timeFilter.value = filters.time; else timeFilter.value = "";
      document.getElementById("minOdds").value = filters.minOdds;
      document.getElementById("maxOdds").value = filters.maxOdds;
      document.getElementById("minEv").value = filters.minEv;
      document.getElementById("minPlaceEv").value = filters.minPlaceEv;
      const dg = document.getElementById("dgFilter");
      if (dg) {
        dg.value = filters.dg || "on";
        updateDgToggleUI(dg);
      }
    }

    function updateDgToggleUI(button) {
      if (!button) return;
      const isOn = String(button.value || "on") === "on";
      button.classList.toggle("on", isOn);
      button.setAttribute("aria-pressed", isOn ? "true" : "false");
      const text = button.querySelector(".dg-toggle-text");
      if (text) text.textContent = isOn ? "On" : "Off";
    }

    function syncApplyButton() {
      const applyBtn = document.getElementById("applyFilters");
      if (!applyBtn) return;
      applyBtn.disabled = filtersEqual(getWorkingFilters(), getAppliedFilters());
    }

    function setFilterLabel(inputId, text) {
      const field = document.getElementById(inputId)?.closest(".field");
      const label = field?.querySelector("label");
      if (label) label.textContent = text;
    }

    function updateMixedFilterTabs() {
      const tabs = document.getElementById("mixedFilterTabs");
      if (!tabs) return;
      tabs.style.display = isMixedEdge() ? "grid" : "none";
      tabs.querySelectorAll(".mixed-filter-tab").forEach(btn => {
        btn.classList.toggle("active", btn.dataset.mixedFilterTab === activeMixedFilterTab);
      });
    }

    function populateFilters() {
      updateMixedFilterTabs();
      const activeKind = isMixedEdge() ? activeMixedFilterTab : (isGolfEdge() ? "golf" : "horse");
      const working = getWorkingFilters(activeKind);
      const golf = activeKind === "golf";
      const mixed = isMixedEdge();
      const scopedRunners = mixed ? runners.filter(r => golf ? isRunnerGolf(r) : !isRunnerGolf(r)) : runners;

      setFilterLabel("meetFilter", golf ? "Event" : "Meet");
      setFilterLabel("timeFilter", golf ? "Places" : "Time");
      setFilterLabel("minEv", golf ? "Min Total %" : "Min EV %");
      setFilterLabel("minPlaceEv", golf ? "Min Place %" : "Min Place EV %");
      const dgField = document.querySelector(".dg-filter-field");
      if (dgField) dgField.style.display = golf ? "block" : "none";

      meetFilter.innerHTML = golf ? `<option value="">All Events</option>` : `<option value="">All Meets</option>`;
      timeFilter.innerHTML = golf ? `<option value="">All Places</option>` : `<option value="">All Times</option>`;

      [...new Set(scopedRunners.map(r => r.meet).filter(Boolean))].sort().forEach(meet => {
        meetFilter.insertAdjacentHTML("beforeend", `<option value="${escapeHtml(meet)}">${escapeHtml(meet)}</option>`);
      });

      const timeOptions = scopedRunners.map(r => {
        const value = golf ? Number(r.p || 0) : r.time;
        return { value: String(value), label: golf ? `${value} Places` : String(value), sort: String(value) };
      }).filter(opt => golf ? [5, 8, 10, 12].includes(Number(opt.value)) : Boolean(opt.value));

      [...new Map(timeOptions.map(opt => [opt.value, opt])).values()]
        .sort((a,b) => String(a.sort).localeCompare(String(b.sort), undefined, { numeric: true }))
        .forEach(opt => {
          timeFilter.insertAdjacentHTML("beforeend", `<option value="${escapeHtml(opt.value)}">${escapeHtml(opt.label)}</option>`);
        });

      writeFilterInputs(working);
      syncApplyButton();
    }

    function runnerMatchesFilters(r, filters, kind) {
      const minOdds = Number(filters.minOdds || 0);
      const maxOdds = Number(filters.maxOdds || 1001);
      const minEv = Number(filters.minEv || 0);
      const minPlaceEv = Number(filters.minPlaceEv || 0);
      const meet = filters.meet;
      const time = filters.time;
      const golf = kind === "golf";
      const dg = filters.dg || "on";

      if (meet && r.meet !== meet) return false;
      if (time) {
        if (golf) {
          if (String(r.p || "") !== String(time)) return false;
        } else if (String(r.time || "") !== String(time)) return false;
      }
      if (golf && isRunnerGolf(r) && dg !== "all" && (dg === "on" ? !r.dgOn : !!r.dgOn)) return false;
      if (Number(r.odds || 0) < minOdds || Number(r.odds || 0) > maxOdds) return false;
      if (Number(r.totalEv || 0) < minEv) return false;
      if (Number(r.placeEv || 0) < minPlaceEv) return false;
      return true;
    }

    function getFiltered() {
      const golf = isGolfEdge();
      const mixed = isMixedEdge();
      let data;

      if (mixed) {
        const horseFilters = getAppliedFilters("horse");
        const golfFilters = getAppliedFilters("golf");
        data = runners.filter(r => {
          const isGolf = isRunnerGolf(r);
          return runnerMatchesFilters(r, isGolf ? golfFilters : horseFilters, isGolf ? "golf" : "horse");
        });
      } else {
        const filters = getAppliedFilters(golf ? "golf" : "horse");
        data = runners.filter(r => runnerMatchesFilters(r, filters, golf ? "golf" : "horse"));
      }

      return data.sort((a, b) => (golf || mixed) ? b.placeEv - a.placeEv : b.totalEv - a.totalEv);
    }


    function runnerKey(r) {
      const edgeId = r.edgeId || "legacy";
      const edgeType = String(r.edgeType || "").toLowerCase();
      const isGolf = edgeType === "golf" || String(edgeId).toLowerCase().includes("golf");
      const base = isGolf
        ? [edgeId, r.dataSource || "", r.horse || "", r.meet || "", r.p || "", r.bookie || ""]
        : [edgeId, r.horse || "", r.meet || "", r.time || "", r.bookie || ""];
      const version = [r.winOdds || r.odds || "", r.winF || "", r.placeOdds || "", r.placeF || "", r.winEv || "", r.placeEv || "", r.totalEv || ""];
      return base.concat(version).map(v => String(v ?? "").trim()).join("|");
    }

    function isSaved(r) {
      const key = runnerKey(r);
      return savedSelections.some(s => runnerKey(s) === key);
    }


    function updateMyBetsNavState() {
      const myBetsNav = document.querySelector('[data-nav="mybets"]');
      if (!myBetsNav) return;
      const hasBets = savedBets.length > 0;
      myBetsNav.classList.toggle("has-bets", hasBets);
      const label = myBetsNav.querySelector(".nav-label");
      if (label) label.textContent = `My Bets (${savedBets.length})`;
    }

    function updateSelectionNavState() {
      const selectionsNav = document.querySelector('[data-nav="selections"]');
      if (!selectionsNav) return;

      const hasSelections = savedSelections.length > 0;
      selectionsNav.classList.toggle("has-selections", hasSelections);

      const icon = selectionsNav.querySelector("div");
      if (icon) icon.textContent = hasSelections ? "★" : "☆";

      const label = selectionsNav.querySelector(".nav-label");
      if (label) label.textContent = `Selections (${savedSelections.length})`;
    }

    function saveSelections() {
      localStorage.setItem('savedSelections', JSON.stringify(savedSelections));
      updateSelectionNavState();
    }

    function toggleSelection(key) {
      const source = runners.find(r => runnerKey(r) === key) || savedSelections.find(r => runnerKey(r) === key);
      if (!source) return;

      const index = savedSelections.findIndex(s => runnerKey(s) === key);
      const willSave = index < 0;

      if (willSave) {
        savedSelections.push({
          ...source,
          savedAt: source.savedAt || new Date().toLocaleString("en-GB"),
          originalSnapshot: source.originalSnapshot || {
            winOdds: source.winOdds, winF: source.winF, placeOdds: source.placeOdds, placeF: source.placeF,
            winEv: source.winEv, placeEv: source.placeEv, totalEv: source.totalEv
          }
        });
      } else {
        savedSelections.splice(index, 1);
      }

      saveSelections();

      document.querySelectorAll(`.save-horse[data-key="${CSS.escape(key)}"]`).forEach(button => {
        button.classList.toggle("saved", willSave);
        button.innerHTML = willSave
          ? "<span>★</span><span>Saved</span>"
          : "<span>☆</span><span>Save</span>";
      });

      document.querySelectorAll(`.save-star-button[data-key="${CSS.escape(key)}"]`).forEach(button => {
        button.classList.toggle("saved", willSave);
        button.setAttribute("aria-label", willSave ? "Remove selection" : "Save selection");
      });

      document.querySelectorAll(`.horse-line[data-key="${CSS.escape(key)}"]`).forEach(line => {
        line.classList.toggle("is-saved", willSave);
      });

      renderSelections();
      renderBetMaker();
    }

    function openSelectionEdit(button) {
      const card = button.closest(".selection-card");
      if (card) card.classList.add("editing", "open");
    }

    function closeSelectionEdit(button) {
      const card = button.closest(".selection-card");
      if (card) card.classList.remove("editing");
    }

    function saveSelectionEdit(button) {
      const card = button.closest(".selection-card");
      if (!card) return;
      const oldKey = card.dataset.key || "";
      const index = savedSelections.findIndex(s => runnerKey(s) === oldKey);
      if (index < 0) return;
      const get = name => Number(card.querySelector(`[data-edit="${name}"]`)?.value || 0);
      const updated = { ...savedSelections[index] };
      updated.winOdds = get("winOdds");
      updated.odds = updated.winOdds;
      updated.winF = get("winF");
      updated.placeOdds = get("placeOdds");
      updated.placeF = get("placeF");
      updated.winEv = calcEvPercent(updated.winOdds, updated.winF);
      updated.placeEv = calcEvPercent(updated.placeOdds, updated.placeF);
      updated.totalEv = updated.winEv && updated.placeEv ? (updated.winEv + updated.placeEv) / 2 : Math.max(updated.winEv, updated.placeEv);
      updated.editedAt = new Date().toLocaleString("en-GB");
      savedSelections[index] = updated;
      saveSelections();
      renderSelections();
      render();
      renderBetMaker();
    }


    function resetSelectionToOriginal(button) {
      const card = button.closest(".selection-card");
      if (!card) return;
      if (!button.classList.contains("confirming")) {
        button.classList.add("confirming");
        button.textContent = "Confirm Reset?";
        clearTimeout(button._resetTimer);
        button._resetTimer = setTimeout(() => {
          button.classList.remove("confirming");
          button.textContent = "Reset";
        }, 3500);
        return;
      }

      const oldKey = card.dataset.key || "";
      const index = savedSelections.findIndex(s => runnerKey(s) === oldKey);
      if (index < 0) return;
      const current = savedSelections[index];
      const original = current.originalSnapshot || current;
      const updated = { ...current };
      updated.winOdds = Number(original.winOdds || original.odds || updated.winOdds || 0);
      updated.odds = updated.winOdds;
      updated.winF = Number(original.winF || updated.winF || 0);
      updated.placeOdds = Number(original.placeOdds || updated.placeOdds || 0);
      updated.placeF = Number(original.placeF || updated.placeF || 0);
      updated.winEv = Number(original.winEv || calcEvPercent(updated.winOdds, updated.winF));
      updated.placeEv = Number(original.placeEv || calcEvPercent(updated.placeOdds, updated.placeF));
      updated.totalEv = Number(original.totalEv || (updated.winEv && updated.placeEv ? (updated.winEv + updated.placeEv) / 2 : Math.max(updated.winEv, updated.placeEv)));
      delete updated.editedAt;
      savedSelections[index] = updated;
      saveSelections();
      renderSelections();
      render();
      renderBetMaker();
    }

    function applyMarketSort() {
      render();
    }

    function sortFilteredRows(rows) {
      const sortBy = document.getElementById("sortBy")?.value || "totalEv";
      const metric = sortBy === "placeEv" ? "placeEv" : "totalEv";
      return [...rows].sort((a,b) => Number(b[metric] || 0) - Number(a[metric] || 0));
    }

    function render() {
      const sortBy = document.getElementById("sortBy")?.value || "totalEv";
      const activeEvMetric = sortBy === "placeEv" ? "placeEv" : "totalEv";
      const golf = isGolfEdge();
      const mixed = isMixedEdge();
      const activeEvLabel = golf
        ? (activeEvMetric === "placeEv" ? "Place Rating %" : "Total Rating %")
        : mixed
          ? (activeEvMetric === "placeEv" ? "Place %" : "Total %")
          : (activeEvMetric === "placeEv" ? "Place EV %" : "Total EV %");
      const marketEvHeader = document.getElementById("marketEvHeader");
      if (marketEvHeader) marketEvHeader.textContent = activeEvLabel;
      const headerLabels = document.querySelectorAll(".header-row span");
      if (headerLabels[1]) headerLabels[1].textContent = mixed ? "Selection" : (golf ? "Player" : "Horse");
      if (headerLabels[3]) headerLabels[3].textContent = golf || mixed ? "PT" : "P/T";
      const data = sortFilteredRows(getFiltered());
      document.getElementById("runnerCount").textContent = data.length || 0;
      document.getElementById("meetingCount").textContent = data.length ? new Set(data.map(r => r.meet).filter(Boolean)).size : 0;

      if (!data.length) {
        const meta = sourceMetaForActiveFeed();
        feed.innerHTML = `<div class="empty-state">No visible runners for ${meta.sourceLabel}. Check filters or refresh the live sheet.</div>`;
        return;
      }

      feed.innerHTML = data.map((r, i) => {
        const rowGolf = isRunnerGolf(r);
        const sourceBadge = rowGolf && r.dataSource === "dg" ? `<span class="source-badge dg">DG</span>` : (rowGolf && r.dataSource === "ex" ? `<span class="source-badge ex">EX</span>` : "");
        const oddsMatched = rowGolf && r.selectionMatched ? `<span class="matched-under">(${r.selectionMatched})</span>` : "";
        const subLine = rowGolf ? `${r.meet} - ${Number(r.p || 0)} places` : `${r.meet} · ${r.time}`;
        return `
        <article class="card">
          <div class="runner" onclick="this.closest('article').classList.toggle('open')">
            <button class="save-star-button ${isSaved(r) ? 'saved' : ''}" data-key="${runnerKey(r)}" type="button" aria-label="${isSaved(r) ? 'Remove selection' : 'Save selection'}">★</button>
            <span class="horse-cell">
              <span class="horse-line ${isSaved(r) ? 'is-saved' : ''}" data-key="${runnerKey(r)}">${sourceBadge}<span class="horse">${r.horse}</span></span>
              <span class="meet-under">${subLine}</span>
            </span>
            <span class="odds">${money(r.odds)}${oddsMatched}</span>
            <span class="pt-stack">
              <span class="pill p${r.p}">${r.p}</span>
              <span class="pill term ${r.t === '1/4' ? 'term-quarter' : ''}">${r.t}</span>
            </span>
            <span class="ev metric-ev-pill ${activeEvMetric === 'placeEv' ? 'place-ev-display' : 'total-ev-display'} ${evClass(r[activeEvMetric])}" onclick="this.closest('.card').classList.toggle('open')" style="cursor:pointer;${evStyle(r[activeEvMetric])}">${Number(r[activeEvMetric]).toFixed(1)}%</span>
          </div>
          <div class="more market-more ${rowGolf ? 'golf-more' : ''}">
            <div class="more-data">
              ${rowGolf ? `
                <div class="more-col">
                  <h4>WIN</h4>
                  <div class="info-row"><span>Odds</span><strong>${money(r.winOdds)}</strong></div>
                  <div class="info-row"><span>Fair</span><strong>${money(r.winF)}</strong></div>
                  <div class="info-row"><span>%</span><strong class="ev-chip ${evClass(r.winEv)}" style="${evStyle(r.winEv)}">${Number(r.winEv).toFixed(1)}%</strong></div>
                </div>
                <div class="more-col">
                  <h4>PLACE</h4>
                  <div class="info-row"><span>Odds</span><strong>${money(r.placeOdds)}</strong></div>
                  <div class="info-row"><span>Fair</span><strong>${money(r.placeF)}</strong></div>
                  <div class="info-row"><span>%</span><strong class="ev-chip ${evClass(r.placeEv)}" style="${evStyle(r.placeEv)}">${Number(r.placeEv).toFixed(1)}%</strong></div>
                </div>
                <div class="more-col">
                  <h4>TOTAL</h4>
                  <div class="info-row"><span>Places</span><strong>${Number(r.p || 0)}</strong></div>
                  <div class="info-row"><span>Terms</span><strong>${r.t}</strong></div>
                  <div class="info-row"><span>%</span><strong class="ev-chip ${evClass(r.totalEv)}" style="${evStyle(r.totalEv)}">${Number(r.totalEv).toFixed(1)}%</strong></div>
                </div>
              ` : `
                <div class="more-col">
                  <h4>WIN</h4>
                  <div class="info-row"><span>Odds</span><strong>${money(r.winOdds)}</strong></div>
                  <div class="info-row"><span>Fair</span><strong>${money(r.winF)}</strong></div>
                  <div class="info-row"><span>EV</span><strong class="ev-chip ${evClass(r.winEv)}" style="${evStyle(r.winEv)}">${Number(r.winEv).toFixed(1)}%</strong></div>
                </div>
                <div class="more-col">
                  <h4>PLACE</h4>
                  <div class="info-row"><span>Odds</span><strong>${money(r.placeOdds)}</strong></div>
                  <div class="info-row"><span>Fair</span><strong>${money(r.placeF)}</strong></div>
                  <div class="info-row"><span>EV</span><strong class="ev-chip ${evClass(r.placeEv)}" style="${evStyle(r.placeEv)}">${Number(r.placeEv).toFixed(1)}%</strong></div>
                </div>
              `}
            </div>
          </div>
        </article>
      `;
      }).join("");

    }



    function isSelectionGroupOpen(label) {
      return selectionGroupOpen[label] === true;
    }

    function toggleSelectionGroup(label) {
      selectionGroupOpen[label] = !isSelectionGroupOpen(label);
      localStorage.setItem('selectionGroupOpenV2', JSON.stringify(selectionGroupOpen));
      renderSelections();
    }

    function updateSelectionEvControls() {
      const metric = selectionEvMetric === 'placeEv' ? 'placeEv' : 'totalEv';
      document.querySelectorAll('[data-selection-ev]').forEach(btn => {
        btn.classList.toggle('active', btn.dataset.selectionEv === metric);
      });
      const header = document.getElementById('selectionEvHeader');
      if (header) header.textContent = metric === 'placeEv' ? 'Place EV %' : 'Total EV %';
    }

    function setSelectionEvMetric(metric) {
      selectionEvMetric = metric === 'placeEv' ? 'placeEv' : 'totalEv';
      localStorage.setItem('selectionEvMetric', selectionEvMetric);
      renderSelections();
    }

    function renderSelections() {
      updateSelectionNavState();
      updateSelectionEvControls();
      const target = document.getElementById("selectionsFeed");
      if (!target) return;

      if (!savedSelections.length) {
        target.innerHTML = `<div class="empty-state">No saved horses yet. Tap the star beside a runner to save it.</div>`;
        return;
      }

      const edgeOrder = { "Golf": 1, "Horse/Golf": 2, "Horses": 3 };
      const shopOrder = { "Betfred Shop": 1, "Ladbrokes Shop": 2, "William Hill Shop": 3 };
      const subOrder = { "Horse": 0, "Horses": 0, "DP": 1, "PGA": 2 };

      function normalisedSelectionEdge(selection = {}) {
        const label = selection.sourceLabel || `${selection.shopName || "Legacy Feed"} - ${selection.edgeName || "Legacy Edge"}`;
        const parts = sourcePartsFromLabel(label);
        const edge = String(parts.edgeName || selection.edgeName || selection.edgeType || "Horses");
        if (edge.toLowerCase().includes("horse/golf")) return "Horse/Golf";
        if (edge.toLowerCase().includes("golf")) return "Golf";
        return "Horses";
      }

      function selectionSubGroup(selection = {}, mainEdge = "Horses") {
        const isGolfSelection = typeof isRunnerGolf === "function" ? isRunnerGolf(selection) : (
          String(selection.edgeType || selection.edgeName || "").toLowerCase() === "golf"
          || String(selection.sourceSheetName || selection.sheetName || "").toLowerCase().includes("golf")
        );

        if (mainEdge === "Horses") return "";
        if (mainEdge === "Golf") return selection.comp || compFromSheetName(selection.sourceSheetName || selection.sheetName || "") || "Golf";
        if (mainEdge === "Horse/Golf") {
          return isGolfSelection ? (selection.comp || compFromSheetName(selection.sourceSheetName || selection.sheetName || "") || "Golf") : "Horse";
        }
        return "";
      }

      const groups = {};
      savedSelections.forEach(selection => {
        const rowInfo = selectionGroupInfo(selection);
        const mainEdge = normalisedSelectionEdge(selection);
        const subGroup = selectionSubGroup(selection, mainEdge);
        const shopName = rowInfo.shopName || "Legacy Feed";
        const shopClass = rowInfo.shopClass || "shop-wh";
        const key = `${shopName}||${mainEdge}`;
        if (!groups[key]) groups[key] = { key, shopName, shopClass, edgeName: mainEdge, selections: [], subGroups: {} };
        groups[key].selections.push(selection);
        const subKey = subGroup || "__none__";
        if (!groups[key].subGroups[subKey]) groups[key].subGroups[subKey] = [];
        groups[key].subGroups[subKey].push(selection);
      });

      const sortedGroups = Object.values(groups).sort((a, b) => {
        return (shopOrder[a.shopName] || 99) - (shopOrder[b.shopName] || 99)
          || String(a.shopName || "").localeCompare(String(b.shopName || ""), undefined, { sensitivity: "base" })
          || (edgeOrder[a.edgeName] || 99) - (edgeOrder[b.edgeName] || 99)
          || String(a.edgeName || "").localeCompare(String(b.edgeName || ""), undefined, { sensitivity: "base" });
      });

      function renderSelectionCard(r) {
        rank += 1;
        const golfSelection = String(r.edgeType || r.edgeName || "").toLowerCase() === "golf" || String(r.edgeId || "").toLowerCase().includes("golf") || String(r.sourceSheetName || r.sheetName || "").toLowerCase().includes("golf");
        const sourceBadge = golfSelection && r.dataSource === "dg" ? `<span class="source-badge dg">DG</span>` : (golfSelection && r.dataSource === "ex" ? `<span class="source-badge ex">EX</span>` : "");
        const oddsMatched = golfSelection && r.selectionMatched ? `<span class="matched-under">(${r.selectionMatched})</span>` : "";
        const selectionMetric = selectionEvMetric === "placeEv" ? "placeEv" : "totalEv";
        const subLine = golfSelection ? `${r.meet} - ${Number(r.p || 0)} places` : `${r.meet} · ${r.time}`;
        const editKey = runnerKey(r);
        const savedAtLine = r.savedAt ? `<span class="matched-under">Saved ${escapeHtml(r.savedAt)}</span>` : "";
        return `
        <article class="card selection-card" data-key="${escapeHtml(editKey)}">
          <div class="runner" onclick="this.closest('article').classList.toggle('open')">
            <button class="save-star-button saved" data-key="${runnerKey(r)}" type="button" aria-label="Remove selection">★</button>
            <span class="horse-cell">
              <span class="horse-line is-saved" data-key="${runnerKey(r)}">${sourceBadge}<span class="horse">${escapeHtml(r.horse)}</span></span>
              <span class="meet-under">${escapeHtml(subLine)}</span>${savedAtLine}
            </span>
            <span class="odds">${money(r.odds)}${oddsMatched}</span>
            <span class="pt-stack">
              <span class="pill p${r.p}">${r.p}</span>
              <span class="pill term ${r.t === '1/4' ? 'term-quarter' : ''}">${r.t}</span>
            </span>
            <span class="ev ${evClass(r[selectionMetric])}" style="${evStyle(r[selectionMetric])}">${Number(r[selectionMetric]).toFixed(1)}%</span>
          </div>
          <div class="more market-more selection-more ${golfSelection ? 'golf-more' : ''}">
            <div class="more-data">
              ${golfSelection ? `
                <div class="more-col">
                  <h4>WIN</h4>
                  <div class="info-row"><span>Odds</span><strong>${money(r.winOdds)}</strong></div>
                  <div class="info-row"><span>Fair</span><strong>${money(r.winF)}</strong></div>
                  <div class="info-row"><span>%</span><strong class="ev-chip ${evClass(r.winEv)}" style="${evStyle(r.winEv)}">${Number(r.winEv).toFixed(1)}%</strong></div>
                </div>
                <div class="more-col">
                  <h4>PLACE</h4>
                  <div class="info-row"><span>Odds</span><strong>${money(r.placeOdds)}</strong></div>
                  <div class="info-row"><span>Fair</span><strong>${money(r.placeF)}</strong></div>
                  <div class="info-row"><span>%</span><strong class="ev-chip ${evClass(r.placeEv)}" style="${evStyle(r.placeEv)}">${Number(r.placeEv).toFixed(1)}%</strong></div>
                </div>
                <div class="more-col">
                  <h4>TOTAL</h4>
                  <div class="info-row"><span>Places</span><strong>${Number(r.p || 0)}</strong></div>
                  <div class="info-row"><span>Terms</span><strong>${r.t}</strong></div>
                  <div class="info-row"><span>%</span><strong class="ev-chip ${evClass(r.totalEv)}" style="${evStyle(r.totalEv)}">${Number(r.totalEv).toFixed(1)}%</strong></div>
                </div>
              ` : `
                <div class="more-col">
                  <h4>WIN</h4>
                  <div class="info-row"><span>Odds</span><strong>${money(r.winOdds)}</strong></div>
                  <div class="info-row"><span>Fair</span><strong>${money(r.winF)}</strong></div>
                  <div class="info-row"><span>EV</span><strong class="ev-chip ${evClass(r.winEv)}" style="${evStyle(r.winEv)}">${Number(r.winEv).toFixed(1)}%</strong></div>
                </div>
                <div class="more-col">
                  <h4>PLACE</h4>
                  <div class="info-row"><span>Odds</span><strong>${money(r.placeOdds)}</strong></div>
                  <div class="info-row"><span>Fair</span><strong>${money(r.placeF)}</strong></div>
                  <div class="info-row"><span>EV</span><strong class="ev-chip ${evClass(r.placeEv)}" style="${evStyle(r.placeEv)}">${Number(r.placeEv).toFixed(1)}%</strong></div>
                </div>
              `}
            </div>
            <div class="selection-edit-actions"><button class="selection-edit-btn" type="button" onclick="openSelectionEdit(this); event.stopPropagation();">Edit</button></div>
            <div class="selection-edit-panel" onclick="event.stopPropagation();">
              <label>Win Odds<input data-edit="winOdds" type="number" step="0.01" value="${Number(r.winOdds || r.odds || 0)}"></label>
              <label>Win Fair<input data-edit="winF" type="number" step="0.01" value="${Number(r.winF || 0)}"></label>
              <label>Place Odds<input data-edit="placeOdds" type="number" step="0.01" value="${Number(r.placeOdds || 0)}"></label>
              <label>Place Fair<input data-edit="placeF" type="number" step="0.01" value="${Number(r.placeF || 0)}"></label>
              <div class="selection-edit-actions"><button class="selection-reset-btn" type="button" onclick="resetSelectionToOriginal(this);">Reset</button><button class="selection-cancel-btn" type="button" onclick="closeSelectionEdit(this);">Cancel</button><button class="selection-save-btn" type="button" onclick="saveSelectionEdit(this);">Save</button></div>
            </div>
          </div>
        </article>`;
      }

      let rank = 0;
      target.innerHTML = sortedGroups.map(group => {
        const label = group.key;
        const safeLabel = escapeHtml(label);
        const groupOpen = isSelectionGroupOpen(label);
        const countLabel = `${group.selections.length} saved`;
        const header = `<div class="source-group-header pill-header selection-source-toggle ${group.shopClass} ${groupOpen ? "" : "is-closed"}" data-group="${safeLabel}" onclick="toggleSelectionGroup(this.dataset.group)"><span class="shop-chip ${group.shopClass}">${escapeHtml(group.shopName)}</span><span class="edge-pill">${escapeHtml(group.edgeName)}</span><span class="selection-group-count">${countLabel}</span><span class="selection-group-chevron">›</span></div>`;
        const sortedSubGroups = Object.entries(group.subGroups).sort(([a], [b]) => {
          if (a === "__none__" && b === "__none__") return 0;
          if (a === "__none__") return -1;
          if (b === "__none__") return 1;
          return (subOrder[a] ?? 99) - (subOrder[b] ?? 99) || String(a).localeCompare(String(b), undefined, { sensitivity: "base" });
        });
        const body = sortedSubGroups.map(([subLabel, selections]) => {
          const showSubHeader = subLabel !== "__none__";
          const subHeader = showSubHeader ? `<div class="selection-subgroup-header"><span class="edge-pill">${escapeHtml(subLabel)}</span><span class="selection-subgroup-count">${selections.length} saved</span></div>` : "";
          return `${subHeader}${selections.map(renderSelectionCard).join("")}`;
        }).join("");
        return `${header}<div class="selection-group-body ${groupOpen ? "" : "is-closed"}">${body}</div>`;
      }).join("");
    }

    function showScreen(name) {
      document.querySelectorAll(".screen").forEach(s => s.classList.remove("active"));
      document.querySelectorAll(".nav-item").forEach(n => n.classList.remove("active"));

      if (name === "selections") {
        document.getElementById("selectionsScreen").classList.add("active");
        renderSelections();
      } else if (name === "betmaker") {
        document.getElementById("betmakerScreen").classList.add("active");
        renderBetMaker();
      } else if (name === "mybets") {
        document.getElementById("mybetsScreen").classList.add("active");
        renderMyBets();
      } else {
        document.getElementById("marketsScreen").classList.add("active");
      }

      const nav = document.querySelector(`[data-nav="${name}"]`);
      if (nav) nav.classList.add("active");
    }





    const BET_DEFS = {
      single: { name: "Single", picks: 1, folds: [1] },
      double: { name: "Double", picks: 2, folds: [2] },
      treble: { name: "Treble", picks: 3, folds: [3] },
      fourfold: { name: "4 Fold", picks: 4, folds: [4] },
      fivefold: { name: "5 Fold", picks: 5, folds: [5] },
      sixfold: { name: "6 Fold", picks: 6, folds: [6] },
      sevenfold: { name: "7 Fold", picks: 7, folds: [7] },
      trixie: { name: "Trixie", picks: 3, folds: [2,3] },
      patent: { name: "Patent", picks: 3, folds: [1,2,3] },
      yankee: { name: "Yankee", picks: 4, folds: [2,3,4] },
      lucky15: { name: "Lucky 15", picks: 4, folds: [1,2,3,4] },
      lucky31: { name: "Lucky 31", picks: 5, folds: [1,2,3,4,5] },
      lucky63: { name: "Lucky 63", picks: 6, folds: [1,2,3,4,5,6] },
      canadian: { name: "Canadian", picks: 5, folds: [2,3,4,5] },
      heinz: { name: "Heinz", picks: 6, folds: [2,3,4,5,6] },
      superheinz: { name: "Super Heinz", picks: 7, folds: [2,3,4,5,6,7] },
      goliath: { name: "Goliath", picks: 8, folds: [2,3,4,5,6,7,8] },
      combo: { name: "Combo Matrix", picks: 6, folds: [3,4,5,6] }
    };

    let savedBets = JSON.parse(localStorage.getItem("savedBets") || "[]");
    let myBetsSelectMode = false;
    let myBetsSelectedIds = new Set();
    let myBetsBulkConfirm = false;

    let betMakerState = {
      calcMode: "basic",
      ew: true,
      sourceMode: "market",
      selectedKeys: [],
      comboFolds: [3],
      comboStakes: {2:1,3:1,4:1,5:1,6:1,7:1,8:1},
      lastBetType: "",
      activePickerIndex: null,
      animateSlots: [],
      clearingSlots: [],
      clearTimer: null,
      savedSourceShopName: "",
      savedSourceEdgeName: ""
    };

    function foldPluralLabel(fold) {
      const labels = {1:"Singles",2:"Doubles",3:"Trebles",4:"4 Folds",5:"5 Folds",6:"6 Folds",7:"7 Folds",8:"8 Folds"};
      return labels[fold] || `${fold} Folds`;
    }

    function foldLabel(fold) {
      const labels = {1:"Single",2:"Double",3:"Treble",4:"4 Fold",5:"5 Fold",6:"6 Fold",7:"7 Fold",8:"8 Fold"};
      return labels[fold] || `${fold} Fold`;
    }

    function combinations(items, size) {
      const result = [];
      function walk(start, combo) {
        if (combo.length === size) { result.push(combo); return; }
        for (let i = start; i < items.length; i++) walk(i + 1, combo.concat(items[i]));
      }
      walk(0, []);
      return result;
    }

    function betSelectionIsGolfGlobal(selection = {}) {
      return typeof isRunnerGolf === "function" ? isRunnerGolf(selection) : (
        String(selection.edgeType || selection.edgeName || "").toLowerCase() === "golf"
        || String(selection.dataEdgeName || "").toLowerCase() === "golf"
        || String(selection.sourceSheetName || selection.sheetName || "").toLowerCase().includes("golf")
      );
    }

    function betConflictKey(selection = {}) {
      const isGolf = betSelectionIsGolfGlobal(selection);
      const normalise = value => String(value || "").trim().toLowerCase();

      if (isGolf) {
        const event = normalise(selection.meet || selection.event || selection.tournament);
        return event ? `golf|${event}` : "";
      }

      const meet = normalise(selection.meet || selection.meeting);
      const time = normalise(selection.time || selection.raceTime || selection.race_time);
      const raceGroup = [meet, time].filter(Boolean).join("|");
      return raceGroup ? `horse|${raceGroup}` : "";
    }

    function hasSameRace(combo) {
      const keys = combo.map(betConflictKey).filter(Boolean);
      return new Set(keys).size !== keys.length;
    }

    function termDivisor(selection) {
      const raw = String(selection.t || "").trim();
      if (raw.includes("/")) return Number(raw.split("/").pop()) || 5;
      return Number(raw) || 5;
    }

    function effectivePlaceOdds(selection) {
      const existing = Number(selection.placeOdds || 0);
      if (existing > 1) return existing;
      const win = Number(selection.odds || selection.winOdds || 0);
      const divisor = termDivisor(selection);
      return win > 1 ? ((win - 1) / divisor) + 1 : 1;
    }

    function comboOdds(combo, key) {
      return combo.reduce((total, r) => total * (key === "placeOdds" ? effectivePlaceOdds(r) : Number(r[key] || 1)), 1);
    }

    function comboCompoundedEv(combo, key) {
      if (!combo.length) return 0;
      const product = combo.reduce((total, r) => total * (Number(r[key] || 0) / 100), 1);
      return product * 100;
    }

    function comboFair(combo, key) {
      return combo.reduce((total, r) => total * Number(r[key] || 1), 1);
    }

    function comboCalc(combo, stake) {
      const winOdds = comboOdds(combo, "odds");
      const placeOdds = comboOdds(combo, "placeOdds");
      const winEv = comboCompoundedEv(combo, "winEv");
      const placeEv = comboCompoundedEv(combo, "placeEv");
      const totalEv = (winEv + placeEv) / 2;
      return { winOdds, placeOdds, winReturn: stake * winOdds, placeReturn: stake * placeOdds, winEv, placeEv, totalEv };
    }

    function renderBreakdown(combo, calc, stake) {
      const winOddsText = combo.map(r => money(r.odds)).join(" × ");
      const placeOddsText = combo.map(r => money(effectivePlaceOdds(r))).join(" × ");
      const winEvText = combo.map(r => `${(Number(r.winEv || 0) / 100).toFixed(3)}`).join(" × ");
      const placeEvText = combo.map(r => `${(Number(r.placeEv || 0) / 100).toFixed(3)}`).join(" × ");

      return `
        <div class="combo-breakdown">
          <div><span>Win Odds</span><strong>${winOddsText} = ${calc.winOdds.toFixed(2)}</strong></div>
          <div><span>Win Fair</span><strong>${combo.map(r => money(r.winF)).join(" × ")} = ${comboFair(combo, "winF").toFixed(2)}</strong></div>
          <div><span>Win EV</span><strong>${winEvText} × 100 = ${calc.winEv.toFixed(1)}%</strong></div>
          <div><span>Win Return</span><strong>£${stake.toFixed(2)} × ${calc.winOdds.toFixed(2)} = £${calc.winReturn.toFixed(2)}</strong></div>
          <div class="debug-gap"><span></span><strong></strong></div>
          <div><span>Place Odds</span><strong>${placeOddsText} = ${calc.placeOdds.toFixed(2)}</strong></div>
          <div><span>Place Fair</span><strong>${combo.map(r => money(r.placeF)).join(" × ")} = ${comboFair(combo, "placeF").toFixed(2)}</strong></div>
          <div><span>Place EV</span><strong>${placeEvText} × 100 = ${calc.placeEv.toFixed(1)}%</strong></div>
          <div><span>Place Return</span><strong>£${stake.toFixed(2)} × ${calc.placeOdds.toFixed(2)} = £${calc.placeReturn.toFixed(2)}</strong></div>
          <div class="debug-gap"><span></span><strong></strong></div>
          <div><span>Total EV</span><strong>(${calc.winEv.toFixed(1)} + ${calc.placeEv.toFixed(1)}) / 2 = ${calc.totalEv.toFixed(1)}%</strong></div>
          <div><span>Line EV</span><strong>£${stake.toFixed(2)} × (${calc.totalEv.toFixed(1)}% - 100%) = ${signedMoney(stake * ((calc.totalEv - 100) / 100))} EV</strong></div>
        </div>
      `;
    }

    function renderComboControls(picks) {
      const comboOptions = document.getElementById("comboOptions");
      const comboChecks = document.getElementById("comboChecks");
      if (!comboOptions || !comboChecks) return;

      comboOptions.classList.add("open");

      const folds = [];
      for (let i = 2; i <= picks; i++) folds.push(i);
      betMakerState.comboFolds = betMakerState.comboFolds.filter(f => f <= picks && f >= 2);
      if (!betMakerState.comboFolds.length) betMakerState.comboFolds = folds.includes(3) ? [3] : [folds[0]];

      comboChecks.innerHTML = `
        <div class="combo-mini-label">Selections</div>
        <div class="combo-select-row">
          ${[4,5,6,7].map(n => `
            <button type="button" class="combo-count-pill ${picks === n ? "active" : ""}" data-picks="${n}">${n}</button>
          `).join("")}
        </div>
        <div class="combo-mini-label">Bet Type(s)</div>
        <div class="combo-fold-row">
          ${[2,3,4,5,6,7].map(f => {
            const disabled = f > picks;
            const checked = !disabled && betMakerState.comboFolds.includes(f);
            const short = f === 2 ? "D" : f === 3 ? "T" : String(f);
            return `
              <button type="button" class="combo-fold-pill ${checked ? "active" : ""} ${disabled ? "disabled" : ""}" data-fold="${f}" ${disabled ? "disabled" : ""}>${short}</button>
            `;
          }).join("")}
        </div>
      `;

      comboChecks.querySelectorAll(".combo-count-pill").forEach(btn => {
        btn.addEventListener("click", () => {
          const comboPickEl = document.getElementById("comboPickCount");
          if (comboPickEl) comboPickEl.value = btn.dataset.picks;
          betMakerState.comboFolds = betMakerState.comboFolds.filter(f => f <= Number(btn.dataset.picks));
          if (!betMakerState.comboFolds.length) betMakerState.comboFolds = [3];
          renderBetMaker();
        });
      });

      comboChecks.querySelectorAll(".combo-fold-pill").forEach(btn => {
        btn.addEventListener("click", () => {
          const fold = Number(btn.dataset.fold);
          if (btn.disabled) return;
          if (betMakerState.comboFolds.includes(fold)) {
            betMakerState.comboFolds = betMakerState.comboFolds.filter(f => f !== fold);
          } else {
            betMakerState.comboFolds.push(fold);
          }
          betMakerState.comboFolds.sort((a,b) => a-b);
          renderBetMaker();
        });
      });
    }

    function updateBetSelectionWarnings() {
      document.querySelectorAll(".bet-selection").forEach(select => select.classList.remove("same-time"));

      const selected = betMakerState.selectedKeys
        .map(key => findBetSelectionByKey(key))
        .filter(Boolean);

      const conflictKeys = selected.map(betConflictKey).filter(Boolean);
      const duplicateTimes = new Set(conflictKeys.filter((time, idx) => conflictKeys.indexOf(time) !== idx));

      document.querySelectorAll(".bet-selection").forEach(select => {
        const selection = findBetSelectionByKey(select.value);
        if (!selection) return;
        const key = betConflictKey(selection);
        select.classList.toggle("same-time", duplicateTimes.has(key));
      });
    }

    function betSavedSelectionGroup(selection = {}) {
      const info = typeof selectionGroupInfo === "function"
        ? selectionGroupInfo(selection)
        : (() => {
            const parts = sourcePartsFromLabel(selection.sourceLabel || `${selection.shopName || "Legacy Feed"} - ${selection.edgeName || "Horses"}`);
            return { shopName: parts.shopName, edgeName: parts.edgeName };
          })();
      return {
        shopName: info.shopName || selection.shopName || "Legacy Feed",
        edgeName: info.edgeName || selection.edgeName || "Horses",
        shopClass: info.shopClass || (selection.shopId ? shopClassFromId(selection.shopId) : shopClassFromName(info.shopName || selection.shopName || ""))
      };
    }

    function betSavedSourceOptions() {
      const shopMap = new Map();
      (savedSelections || []).forEach(selection => {
        const group = betSavedSelectionGroup(selection);
        if (!shopMap.has(group.shopName)) shopMap.set(group.shopName, { shopName: group.shopName, shopClass: group.shopClass, edges: new Set() });
        shopMap.get(group.shopName).edges.add(group.edgeName);
      });

      const edgeOrder = { "Horses": 1, "Golf": 2, "Horse/Golf": 3 };
      return [...shopMap.values()]
        .sort((a, b) => a.shopName.localeCompare(b.shopName, undefined, { sensitivity: "base" }))
        .map(shop => ({
          ...shop,
          edges: [...shop.edges].sort((a, b) => (edgeOrder[a] || 99) - (edgeOrder[b] || 99) || a.localeCompare(b, undefined, { sensitivity: "base" }))
        }));
    }

    function ensureBetSavedSourceFilters() {
      const shops = betSavedSourceOptions();
      if (!shops.length) {
        betMakerState.savedSourceShopName = "";
        betMakerState.savedSourceEdgeName = "";
        return { shops, activeShop: null, activeEdge: "" };
      }

      let activeShop = shops.find(shop => shop.shopName === betMakerState.savedSourceShopName);
      if (!activeShop) {
        const activeFeed = getActiveFeedConfig();
        activeShop = shops.find(shop => shop.shopName === activeFeed.shopName) || shops[0];
        betMakerState.savedSourceShopName = activeShop.shopName;
      }

      if (!activeShop.edges.includes(betMakerState.savedSourceEdgeName)) {
        const activeEdge = getActiveEdgeConfig();
        betMakerState.savedSourceEdgeName = activeShop.edges.includes(activeEdge.edgeName) ? activeEdge.edgeName : activeShop.edges[0];
      }

      return { shops, activeShop, activeEdge: betMakerState.savedSourceEdgeName };
    }

    function cycleBetSavedSourceFilter(kind) {
      const options = ensureBetSavedSourceFilters();
      if (!options.shops.length) return;

      if (kind === "shop") {
        const idx = options.shops.findIndex(shop => shop.shopName === betMakerState.savedSourceShopName);
        const nextShop = options.shops[(idx + 1 + options.shops.length) % options.shops.length];
        betMakerState.savedSourceShopName = nextShop.shopName;
        betMakerState.savedSourceEdgeName = nextShop.edges[0] || "";
      } else if (options.activeShop) {
        const edges = options.activeShop.edges;
        const idx = edges.indexOf(betMakerState.savedSourceEdgeName);
        betMakerState.savedSourceEdgeName = edges[(idx + 1 + edges.length) % edges.length];
      }

      betMakerState.selectedKeys = [];
      betMakerState.activePickerIndex = null;
      renderBetMaker();
    }

    function getBetSourceSelections() {
      if (betMakerState.sourceMode === "market") {
        const marketRows = typeof getFiltered === "function" ? getFiltered() : runners;
        return marketRows && marketRows.length ? marketRows : runners;
      }

      const source = savedSelections || [];
      const filters = ensureBetSavedSourceFilters();
      if (!filters.activeShop || !filters.activeEdge) return source;
      return source.filter(selection => {
        const group = betSavedSelectionGroup(selection);
        return group.shopName === filters.activeShop.shopName && group.edgeName === filters.activeEdge;
      });
    }

    function findBetSelectionByKey(key) {
      const sources = [
        getBetSourceSelections(),
        runners || [],
        savedSelections || []
      ];
      for (const source of sources) {
        const found = source.find(s => runnerKey(s) === key);
        if (found) return found;
      }
      return null;
    }

    function getRequiredBetPicks() {
      const typeEl = document.getElementById("betType");
      const comboPickEl = document.getElementById("comboPickCount");
      if (!typeEl) return 0;
      if (betMakerState.calcMode === "advanced") return Number(comboPickEl?.value || 6);
      return (BET_DEFS[typeEl.value] || BET_DEFS.trixie).picks;
    }

    function uniqueRaceSelections(list, count) {
      const picked = [];
      const seen = new Set();
      for (const selection of list) {
        const key = betConflictKey(selection);
        if (seen.has(key)) continue;
        picked.push(selection);
        seen.add(key);
        if (picked.length >= count) break;
      }
      return picked;
    }

    function autoFillBetMaker(mode) {
      const count = getRequiredBetPicks();
      let pool = [...getBetSourceSelections()];

      if (mode === "bestEv") {
        pool.sort((a, b) => {
          const av = (Number(a.winEv || 0) + Number(a.placeEv || 0)) / 2;
          const bv = (Number(b.winEv || 0) + Number(b.placeEv || 0)) / 2;
          return bv - av;
        });
      } else if (mode === "bestPlace") {
        pool.sort((a, b) => Number(b.placeEv || 0) - Number(a.placeEv || 0));
      } else {
        pool.sort(() => Math.random() - 0.5);
      }

      const picked = uniqueRaceSelections(pool, count);
      const nextKeys = picked.map(runnerKey);
      clearTimeout(betMakerState.clearTimer);
      betMakerState.activePickerIndex = null;
      betMakerState.clearingSlots = [];
      betMakerState.selectedKeys = nextKeys;
      betMakerState.animateSlots = nextKeys.map((_, idx) => idx);
      renderBetMaker();
    }

    function showToast(message, type = "success") {
      const toast = document.getElementById("toastBanner");
      if (!toast) return;
      toast.textContent = message;
      toast.classList.remove("success", "danger");
      toast.classList.add(type);
      toast.classList.add("show");
      clearTimeout(showToast.timer);
      showToast.timer = setTimeout(() => toast.classList.remove("show"), 2000);
    }

    function currentBetSnapshot() {
      const typeEl = document.getElementById("betType");
      const stakeEl = document.getElementById("betStake");
      const type = betMakerState.calcMode === "advanced" ? "combo" : (typeEl?.value || "double");
      const source = getBetSourceSelections();
      const selected = betMakerState.selectedKeys.map(key => source.find(s => runnerKey(s) === key)).filter(Boolean);
      const firstSelection = selected.find(s => s.sourceLabel) || selected[0] || sourceMetaForActiveFeed();
      const firstSource = firstSelection.sourceLabel || sourceMetaForActiveFeed().sourceLabel;
      return { id: Date.now(), created: new Date().toLocaleString("en-GB"), sourceLabel: firstSource, shopId: firstSelection.shopId, shopName: firstSelection.shopName, edgeName: firstSelection.edgeName, type, ew: betMakerState.ew, stake: Number(stakeEl?.value || 0), comboFolds: [...betMakerState.comboFolds], selections: selected.map(s => ({...s})) };
    }

    function saveCurrentBetSlip() {
      const snap = currentBetSnapshot();
      if (!snap.selections.length) return;
      savedBets.unshift(snap);
      localStorage.setItem("savedBets", JSON.stringify(savedBets));
      renderMyBets();
      updateMyBetsNavState();
      showToast("Added to My Bets", "success");
    }

    function savedBetCalc(bet) {
      const def = BET_DEFS[bet.type] || BET_DEFS.double;
      let folds = def.folds || [];
      if (bet.type === "combo" && bet.comboFolds) folds = bet.comboFolds;
      const stake = Number(bet.stake || 0);
      let lineCount = 0;
      let winReturn = 0;
      let placeReturn = 0;
      folds.forEach(fold => {
        combinations(bet.selections || [], fold)
          .filter(combo => !hasSameRace(combo))
          .forEach(combo => {
            lineCount += 1;
            const calc = comboCalc(combo, stake);
            winReturn += calc.winReturn;
            if (bet.ew) placeReturn += calc.placeReturn;
          });
      });
      const validCombos = [];
      folds.forEach(fold => {
        combinations(bet.selections || [], fold)
          .filter(combo => !hasSameRace(combo))
          .forEach(combo => validCombos.push(comboCalc(combo, stake)));
      });
      const winEv = validCombos.length ? validCombos.reduce((s, c) => s + c.winEv, 0) / validCombos.length : 0;
      const placeEv = validCombos.length ? validCombos.reduce((s, c) => s + c.placeEv, 0) / validCombos.length : 0;
      const totalEv = (winEv + placeEv) / 2;

      return {
        lineCount: bet.ew ? lineCount * 2 : lineCount,
        totalStake: lineCount * stake * (bet.ew ? 2 : 1),
        winReturn,
        placeReturn,
        potential: winReturn + placeReturn,
        winEv,
        placeEv,
        totalEv
      };
    }

    function removeSavedBet(id) {
      savedBets = savedBets.filter(b => b.id !== id);
      myBetsSelectedIds.delete(String(id));
      localStorage.setItem("savedBets", JSON.stringify(savedBets));
      renderMyBets();
      updateMyBetsNavState();
      showToast("Removed from My Bets", "danger");
    }

    function toggleMyBetsSelectMode() {
      myBetsSelectMode = !myBetsSelectMode;
      myBetsSelectedIds = new Set();
      myBetsBulkConfirm = false;
      renderMyBets();
    }

    function toggleMyBetSelected(id) {
      const key = String(id);
      if (myBetsSelectedIds.has(key)) myBetsSelectedIds.delete(key);
      else myBetsSelectedIds.add(key);
      myBetsBulkConfirm = false;
      renderMyBets();
    }

    function selectAllMyBets() {
      myBetsSelectedIds = new Set(savedBets.map(b => String(b.id)));
      myBetsBulkConfirm = false;
      renderMyBets();
    }

    function clearMyBetsSelection() {
      myBetsSelectedIds = new Set();
      myBetsBulkConfirm = false;
      renderMyBets();
    }

    function beginBulkDeleteSelectedBets() {
      if (!myBetsSelectedIds.size) return;
      myBetsBulkConfirm = true;
      renderMyBets();
    }

    function cancelBulkDeleteSelectedBets() {
      myBetsBulkConfirm = false;
      renderMyBets();
    }

    function bulkDeleteSelectedBets() {
      if (!myBetsSelectedIds.size) return;
      const selected = new Set(myBetsSelectedIds);
      const removed = savedBets.filter(b => selected.has(String(b.id))).length;
      savedBets = savedBets.filter(b => !selected.has(String(b.id)));
      myBetsSelectedIds = new Set();
      myBetsSelectMode = false;
      myBetsBulkConfirm = false;
      localStorage.setItem("savedBets", JSON.stringify(savedBets));
      renderMyBets();
      updateMyBetsNavState();
      showToast(`Removed ${removed} bet${removed === 1 ? "" : "s"}`, "danger");
    }

    function savedBetTotalStake(bet) {
      const def = BET_DEFS[bet.type] || BET_DEFS.double;
      let folds = def.folds || [];
      if (bet.type === "combo" && bet.comboFolds) folds = bet.comboFolds;
      let count = 0;
      folds.forEach(fold => {
        count += combinations(bet.selections || [], fold).filter(combo => !hasSameRace(combo)).length;
      });
      return count * Number(bet.stake || 0) * (bet.ew ? 2 : 1);
    }

    function renderMyBets() {
      updateMyBetsNavState();
      const target = document.getElementById("myBetsFeed");
      if (!target) return;
      if (!savedBets.length) {
        myBetsSelectMode = false;
        myBetsSelectedIds = new Set();
        target.innerHTML = `<div class="empty-state">No saved bets yet.</div>`;
        return;
      }

      const selectedCount = myBetsSelectedIds.size;
      const toolbar = `
        <div class="mybets-toolbar">
          <div class="mybets-toolbar-left">
            ${myBetsSelectMode ? `<span class="mybets-select-count">${selectedCount} selected</span>` : `<span class="mybets-select-count">${savedBets.length} saved</span>`}
          </div>
          <div class="mybets-toolbar-right">
            ${!myBetsSelectMode ? `
              <button type="button" class="mybets-mini-btn" onclick="toggleMyBetsSelectMode()">Select</button>
            ` : `
              <button type="button" class="mybets-mini-btn" onclick="selectAllMyBets()">Select All</button>
              <button type="button" class="mybets-mini-btn" onclick="clearMyBetsSelection()">Clear</button>
              <button type="button" class="mybets-mini-btn danger" ${selectedCount ? "" : "disabled"} onclick="beginBulkDeleteSelectedBets()">Delete</button>
              <button type="button" class="mybets-mini-btn" onclick="toggleMyBetsSelectMode()">Done</button>
            `}
          </div>
        </div>`;

      const bulkDeleteModal = myBetsBulkConfirm && selectedCount ? `
        <div class="mybets-modal-backdrop" onclick="cancelBulkDeleteSelectedBets()">
          <div class="mybets-modal" onclick="event.stopPropagation()">
            <h4>Delete ${selectedCount} selected bet${selectedCount === 1 ? "" : "s"}?</h4>
            <p>This cannot be undone.</p>
            <div class="mybets-modal-actions">
              <button type="button" class="mybets-modal-cancel" onclick="cancelBulkDeleteSelectedBets()">Cancel</button>
              <button type="button" class="mybets-modal-delete" onclick="bulkDeleteSelectedBets()">Delete</button>
            </div>
          </div>
        </div>` : "";

      const cards = savedBets.map(b => {
        const calc = savedBetCalc(b);
        const isSelected = myBetsSelectedIds.has(String(b.id));
        return `
        <div class="combo-card mybet-card ${myBetsSelectMode ? "selecting" : ""} ${isSelected ? "selected" : ""}">
          <div class="mybet-top" onclick="${myBetsSelectMode ? `toggleMyBetSelected(${b.id})` : `this.closest('.mybet-card').classList.toggle('open')`}">
            ${myBetsSelectMode ? `<span class="mybet-check-wrap"><input class="mybet-check" type="checkbox" ${isSelected ? "checked" : ""} onclick="event.stopPropagation(); toggleMyBetSelected(${b.id});"></span>` : ""}
            <span class="mybet-title"><span class="expand-mark">▾</span>${(() => { const parts = sourcePartsFromLabel(b.sourceLabel || `${b.shopName || "Legacy Feed"} - ${b.edgeName || "Horses"}`); const cls = b.shopId ? shopClassFromId(b.shopId) : parts.shopClass; return `<span class="mybet-shop-pill ${cls}">${parts.shopName}</span><span class="edge-pill">${parts.edgeName}</span><br>`; })()}${(BET_DEFS[b.type]?.name || b.type)} ${b.ew ? "E/W" : "Win"} - ${b.selections.length} selections</span>
            <span class="mybet-right"><span>£${calc.totalStake.toFixed(2)}</span><span class="ev-chip ${evClass(calc.totalEv)}">${calc.totalEv.toFixed(1)}%</span></span>
          </div>
          <div class="meet-under">${b.created}</div>
          <div class="combo-breakdown">
            ${b.selections.map(s => {
              const winEv = Number(s.winEv || 0);
              const placeEv = Number(s.placeEv || 0);
              const totalEv = (winEv + placeEv) / 2;
              return `<div class="mybet-runner-flat">
                <div class="mybet-runner-meta">
                  <span>${s.meet}</span>
                  <span>${s.time}</span>
                </div>
                <div class="mybet-runner-mainline">
                  <strong>${s.horse} @ ${money(s.odds)}</strong>
                  <span class="runner-pills">
                    <em class="ev-chip ${evClass(winEv)}">${winEv.toFixed(1)}%</em>
                    <em class="ev-chip ${evClass(placeEv)}">${placeEv.toFixed(1)}%</em>
                    <span class="ev-sep">-</span>
                    <em class="ev-chip ${evClass(totalEv)}">${totalEv.toFixed(1)}%</em>
                  </span>
                </div>
                <div class="mybet-runner-prices">Win ${money(s.odds)} / ${money(s.winF)} - Place ${money(effectivePlaceOdds(s))} / ${money(s.placeF)}${(() => { const pn = Number(s.p || s.places || 0); if (!pn) return ""; const tt = s.terms || s.t || (pn === 5 ? "1/4" : "1/5"); return ` <span class="mybet-runner-pt"><span class="pill p${pn}">${pn}</span><span class="pill ${pn === 5 ? "term-quarter" : "term"}">${tt}</span></span>`; })()}</div>
              </div>`;
            }).join("")}
            <div class="debug-gap"><span></span><strong></strong></div>
            <div><span>Stake</span><strong>£${calc.totalStake.toFixed(2)} ${(BET_DEFS[b.type]?.name || b.type)}</strong></div>
            <div><span>Potential</span><strong>£${Math.round(calc.potential).toLocaleString("en-GB")}</strong></div>
            <div><span>Selections</span><strong>${b.selections.length}</strong></div>
            ${b.type === "combo" ? `<div><span>Bet Types</span><strong>${(b.comboFolds || []).map(f => foldPluralLabel(f)).join(", ")}</strong></div>` : ""}
            <div class="remove-confirm">
              <button class="remove-bet" type="button" onclick="event.stopPropagation(); this.closest('.remove-confirm').classList.add('confirming');">Remove</button>
              <div class="confirm-actions">
                <span>Remove bet?</span>
                <button type="button" class="confirm-no" onclick="event.stopPropagation(); this.closest('.remove-confirm').classList.remove('confirming');">No</button>
                <button type="button" class="confirm-yes" onclick="event.stopPropagation(); removeSavedBet(${b.id});">Yes</button>
              </div>
            </div>
          </div>
        </div>`;
      }).join("");

      target.innerHTML = toolbar + bulkDeleteModal + cards;
    }

    function getComboLineStake(fold, fallbackStake) {
      if (betMakerState.calcMode !== "advanced") return Number(fallbackStake || 0);
      return Number(betMakerState.comboStakes[fold] ?? fallbackStake ?? 0);
    }

    function renderAdvancedStakeRows(activeFolds) {
      const target = document.getElementById("advancedStakeRows");
      if (!target) return;

      if (betMakerState.calcMode !== "advanced" || !activeFolds.length) {
        target.innerHTML = "";
        return;
      }

      target.innerHTML = `
        <div class="combo-mini-label">Stake Per Type</div>
        <div class="advanced-stake-grid">
          ${activeFolds.map(fold => `
            <label class="advanced-stake-pill">
              <span>${foldLabel(fold)}</span>
              <input type="text" inputmode="decimal" enterkeyhint="done" value="${Number(betMakerState.comboStakes[fold] ?? 1).toFixed(2)}" data-advanced-stake="${fold}">
            </label>
          `).join("")}
        </div>
      `;

      target.querySelectorAll("[data-advanced-stake]").forEach(input => {
        input.addEventListener("change", () => {
          betMakerState.comboStakes[input.dataset.advancedStake] = Number(input.value || 0);
          renderBetMaker();
        });
        input.addEventListener("blur", () => {
          betMakerState.comboStakes[input.dataset.advancedStake] = Number(input.value || 0);
        });
      });
    }

    function renderBetMaker() {
      const typeEl = document.getElementById("betType");
      const rowsEl = document.getElementById("betSelectionRows");
      const matrixEl = document.getElementById("betMatrix");
      const summaryEl = document.getElementById("betSummary");
      const stakeEl = document.getElementById("betStake");
      const comboPickEl = document.getElementById("comboPickCount");
      const comboOptions = document.getElementById("comboOptions");

      if (!typeEl || !rowsEl || !matrixEl || !summaryEl || !stakeEl) return;

      const isCombo = betMakerState.calcMode === "advanced";
      const comboOption = typeEl.querySelector('option[value="combo"]');
      if (isCombo) {
        if (!comboOption) typeEl.insertAdjacentHTML("beforeend", '<option value="combo">Combo Matrix</option>');
        typeEl.value = "combo";
        typeEl.disabled = true;
      } else {
        if (comboOption) comboOption.remove();
        typeEl.disabled = false;
        if (typeEl.value === "combo" || !BET_DEFS[typeEl.value]) typeEl.value = "double";
      }
      const activeType = isCombo ? "combo" : typeEl.value;
      if (activeType !== betMakerState.lastBetType) {
        if (activeType === "combo") betMakerState.comboFolds = [3];
        betMakerState.lastBetType = activeType;
      }
      const stake = Number(stakeEl.value || 0);
      let def = BET_DEFS[activeType] || BET_DEFS.trixie;

      if (isCombo) {
        const picks = Number(comboPickEl?.value || 6);
        def = { name: "Combo Matrix", picks, folds: betMakerState.comboFolds.filter(f => f <= picks) };
        renderComboControls(picks);
        renderAdvancedStakeRows(def.folds);
      } else if (comboOptions) {
        comboOptions.classList.remove("open");
        renderAdvancedStakeRows([]);
      }

      while (betMakerState.selectedKeys.length < def.picks) betMakerState.selectedKeys.push("");
      betMakerState.selectedKeys = betMakerState.selectedKeys.slice(0, def.picks);

      document.querySelectorAll(".fill-source").forEach(button => {
        button.classList.toggle("active", (button.dataset.source || "market") === betMakerState.sourceMode);
      });

      const fillContextEl = document.getElementById("betFillContext");
      if (fillContextEl) {
        if (betMakerState.sourceMode === "market") {
          const feed = getActiveFeedConfig();
          const edge = getActiveEdgeConfig();
          const shopClass = shopClassFromId(feed.shopId);
          fillContextEl.innerHTML = `<span class="shop-chip ${shopClass}">${escapeHtml(feed.shopName)}</span><span class="edge-pill">${escapeHtml(edge.edgeName)}</span>`;
        } else {
          const filters = ensureBetSavedSourceFilters();
          if (filters.activeShop) {
            fillContextEl.innerHTML = `<button type="button" class="bet-source-filter shop-chip ${filters.activeShop.shopClass}" data-filter="shop">${escapeHtml(filters.activeShop.shopName)}</button><button type="button" class="bet-source-filter edge-pill" data-filter="edge">${escapeHtml(filters.activeEdge)}</button>`;
          } else {
            fillContextEl.innerHTML = `<span class="edge-pill">No saved selections</span>`;
          }
        }
      }

      function betSelectionIsGolf(selection = {}) {
        return betSelectionIsGolfGlobal(selection);
      }

      function betShopNick(shopName = "") {
        const name = String(shopName || "").toLowerCase();
        if (name.includes("william")) return "WH Shop";
        if (name.includes("betfred")) return "BF Shop";
        if (name.includes("ladbrokes") || name.includes("lads")) return "LB Shop";
        return shopName || "Shop";
      }

      function betSelectionConflictKey(selection = {}) {
        return betConflictKey(selection);
      }

      function betSelectionMeta(selection = {}) {
        const golf = betSelectionIsGolf(selection);
        const parts = sourcePartsFromLabel(selection.sourceLabel || `${selection.shopName || "Legacy Feed"} - ${selection.edgeName || "Horses"}`);
        const shopClass = selection.shopId ? shopClassFromId(selection.shopId) : parts.shopClass;
        const edgeName = golf ? "Golf" : "Horse";
        const comp = golf ? (selection.comp || compFromSheetName(selection.sourceSheetName || selection.sheetName || "")) : "";
        const shopName = selection.shopName || parts.shopName || "Legacy Feed";
        return { golf, shopClass, shopName, shopNick: betShopNick(shopName), edgeName, comp };
      }

      function betPickDisplay(selection, slotIndex, isOption = false) {
        if (!selection) {
          return `
            <span class="bet-pick-empty-main">Selection ${slotIndex + 1}</span>
            <span class="bet-pick-empty-sub">Tap to choose</span>`;
        }
        const meta = betSelectionMeta(selection);
        const sourceBadge = meta.golf && selection.dataSource === "dg" ? `<span class="source-badge dg">DG</span>` : (meta.golf && selection.dataSource === "ex" ? `<span class="source-badge ex">EX</span>` : "");
        const matchedRaw = selection.selectionMatched || selection.matched || "";
        const placeNum = Number(selection.p || 0);
        const termsText = selection.terms || (placeNum === 5 ? "1/4" : "1/5");
        const ptStack = placeNum ? `<span class="pt-mini-stack"><span class="pill p${placeNum}">${placeNum}</span><span class="pill ${placeNum === 5 ? "term-quarter" : "term"}">${termsText}</span></span>` : "";
        const subLine = meta.golf ? `${selection.meet || ""}${placeNum ? ` - ${placeNum} places` : ""}` : `${selection.meet || ""}${selection.time ? ` · ${selection.time}` : ""}`;
        const winEv = Number(selection.winEv || 0);
        const placeEv = Number(selection.placeEv || 0);
        const totalEv = Number(selection.totalEv ?? ((winEv + placeEv) / 2));
        const sourcePills = `<span class="edge-pill">${escapeHtml(meta.edgeName)}</span>${meta.comp ? `<span class="edge-pill">${escapeHtml(meta.comp)}</span>` : ""}`;
        const matchedLine = meta.golf && matchedRaw ? `<span class="matched-under">(${escapeHtml(matchedRaw)})</span>` : "";
        return `
          <span class="bet-pick-main">
            <span class="bet-pick-name-line">${sourceBadge}<span class="bet-pick-name">${escapeHtml(selection.horse || "Selection")}</span></span>
            <span class="bet-pick-sub">${escapeHtml(subLine)}</span>
          </span>
          <span class="bet-pick-odds-col">
            <span class="bet-pick-odds">${money(selection.odds)}</span>${matchedLine}${ptStack}
          </span>
          <span class="bet-pick-meta">
            <span class="bet-pick-source-row">${sourcePills}</span>
            <span class="bet-pick-data-row"><span class="ev-chip ${evClass(placeEv)}" style="${evStyle(placeEv)}">${placeEv.toFixed(1)}%</span><span class="ev-chip ${evClass(totalEv)}" style="${evStyle(totalEv)}">${totalEv.toFixed(1)}%</span></span>
          </span>`;
      }

      rowsEl.innerHTML = betMakerState.selectedKeys.map((key, i) => {
        const taken = new Set(betMakerState.selectedKeys.filter((k, idx) => k && idx !== i));
        const sourceSelections = getBetSourceSelections();
        const current = findBetSelectionByKey(key);
        const winEv = current ? Number(current.winEv || 0) : 0;
        const placeEv = current ? Number(current.placeEv || 0) : 0;
        const totalEv = current ? (winEv + placeEv) / 2 : 0;
        const pickerOpen = betMakerState.activePickerIndex === i;
        const usedConflictKeys = new Set(
          betMakerState.selectedKeys
            .map((selectedKey, idx) => idx === i || !selectedKey ? null : findBetSelectionByKey(selectedKey))
            .filter(Boolean)
            .map(s => betSelectionConflictKey(s))
            .filter(Boolean)
        );
        const shouldAnimate = current && Array.isArray(betMakerState.animateSlots) && betMakerState.animateSlots.includes(i);
        const animateClass = `${shouldAnimate ? " slot-animate" : ""}`;
        const animateStyle = shouldAnimate ? ` style="--slot-delay:${i * 55}ms"` : "";
        const options = sourceSelections.map(s => {
          const optionKey = runnerKey(s);
          if (taken.has(optionKey) && optionKey !== key) return "";
          const conflictKey = betSelectionConflictKey(s);
          if (conflictKey && usedConflictKeys.has(conflictKey) && optionKey !== key) return "";
          const selectedClass = optionKey === key ? " selected" : "";
          return `<button type="button" class="bet-picker-option${selectedClass}" data-index="${i}" data-select-key="${encodeURIComponent(optionKey)}">${betPickDisplay(s, i, true)}</button>`;
        }).join("");

        return `
          <div class="bet-row custom-bet-row">
            <button type="button" class="bet-pick-trigger ${current ? "filled" : "empty"}${animateClass}"${animateStyle} data-picker-index="${i}">
              ${betPickDisplay(current, i)}
            </button>
            ${pickerOpen ? `
              <div class="bet-picker-panel">
                <button type="button" class="bet-picker-option clear-option" data-index="${i}" data-select-key="">Clear slot</button>
                ${options || `<div class="empty-state">No available selections.</div>`}
              </div>
            ` : ""}
          </div>`;
      }).join("");

      rowsEl.querySelectorAll(".bet-pick-trigger").forEach(button => {
        button.addEventListener("click", () => {
          const idx = Number(button.dataset.pickerIndex);
          betMakerState.activePickerIndex = betMakerState.activePickerIndex === idx ? null : idx;
          renderBetMaker();
        });
      });

      rowsEl.querySelectorAll(".bet-picker-option").forEach(button => {
        button.addEventListener("click", () => {
          const idx = Number(button.dataset.index);
          const encoded = button.dataset.selectKey || "";
          clearTimeout(betMakerState.clearTimer);
          betMakerState.activePickerIndex = null;

          betMakerState.selectedKeys[idx] = encoded ? decodeURIComponent(encoded) : "";
          betMakerState.clearingSlots = [];
          betMakerState.animateSlots = encoded ? [idx] : [];
          renderBetMaker();
        });
      });

      if (Array.isArray(betMakerState.animateSlots) && betMakerState.animateSlots.length) {
        clearTimeout(betMakerState.animateTimer);
        betMakerState.animateTimer = setTimeout(() => { betMakerState.animateSlots = []; }, 650);
      }

      const betFillContextControls = document.getElementById("betFillContext");
      if (betFillContextControls) {
        betFillContextControls.querySelectorAll(".bet-source-filter").forEach(button => {
          button.addEventListener("click", event => {
            event.preventDefault();
            event.stopPropagation();
            if (betMakerState.sourceMode !== "selections") return;
            cycleBetSavedSourceFilter(button.dataset.filter);
          });
        });
      }

      updateBetSelectionWarnings();

      const selected = betMakerState.selectedKeys.map(key => findBetSelectionByKey(key)).filter(Boolean);

      if (selected.length < def.picks || !def.folds.length) {
        summaryEl.innerHTML = `<div class="summary-tile"><strong>${selected.length}/${def.picks}</strong><span>Selected</span></div>`;
        matrixEl.innerHTML = `<div class="empty-state">Choose ${def.picks} selections${isCombo ? " and at least one bet type" : ""}.</div>`;
        return;
      }

      let combos = [];
      def.folds.forEach(fold => combinations(selected, fold).forEach(combo => combos.push({ fold, combo })));

      const invalidCount = combos.filter(c => hasSameRace(c.combo)).length;
      combos = combos.filter(c => !hasSameRace(c.combo));

      const calcRows = combos.map(c => ({ ...c, lineStake: getComboLineStake(c.fold, stake), calc: comboCalc(c.combo, getComboLineStake(c.fold, stake)) }));

      const totalBets = calcRows.length;
      const totalStake = calcRows.reduce((sum, row) => sum + (row.lineStake || stake) * (betMakerState.ew ? 2 : 1), 0);
      const potentialWin = calcRows.reduce((sum, c) => sum + c.calc.winReturn, 0);
      const potentialPlace = betMakerState.ew ? calcRows.reduce((sum, c) => sum + c.calc.placeReturn, 0) : 0;
      const potential = potentialWin + potentialPlace;

      const avgWinEv = calcRows.length ? calcRows.reduce((sum, c) => sum + c.calc.winEv, 0) / calcRows.length : 0;
      const avgPlaceEv = calcRows.length ? calcRows.reduce((sum, c) => sum + c.calc.placeEv, 0) / calcRows.length : 0;
      const unweightedTrueEv = (avgWinEv + avgPlaceEv) / 2;
      const weightedProfit = calcRows.reduce((sum, row) => sum + (row.lineStake || stake) * ((row.calc.totalEv - 100) / 100) * (betMakerState.ew ? 2 : 1), 0);
      const weightedStake = calcRows.reduce((sum, row) => sum + (row.lineStake || stake) * (betMakerState.ew ? 2 : 1), 0);
      const trueEv = betMakerState.calcMode === "advanced" && weightedStake
        ? ((weightedProfit / weightedStake) + 1) * 100
        : unweightedTrueEv;

      function foldSummaryRows(rows, field) {
        const grouped = {};
        rows.forEach(row => {
          if (!grouped[row.fold]) grouped[row.fold] = [];
          grouped[row.fold].push(row);
        });
        return Object.entries(grouped).map(([fold, arr]) => {
          const avg = arr.reduce((sum, row) => sum + row.calc[field], 0) / arr.length;
          return `<div><span>${foldLabel(Number(fold))}</span><b>${avg.toFixed(1)}%</b></div>`;
        }).join("");
      }

      function totalEvBreakdownRows(rows) {
        const grouped = {};
        rows.forEach(row => {
          if (!grouped[row.fold]) grouped[row.fold] = [];
          grouped[row.fold].push(row);
        });

        let totalStakeForAll = 0;
        let totalEvProfit = 0;

        const parts = Object.entries(grouped).map(([fold, arr]) => {
          const stakeForType = arr.reduce((sum, row) => sum + (row.lineStake || stake) * 2, 0);
          const evProfit = arr.reduce((sum, row) => sum + ((row.lineStake || stake) * 2) * ((row.calc.totalEv - 100) / 100), 0);
          const typeEv = stakeForType ? 100 + ((evProfit / stakeForType) * 100) : 0;

          totalStakeForAll += stakeForType;
          totalEvProfit += evProfit;

          return `
            <div><span>${foldLabel(Number(fold))}</span><b>${arr.length * 2} lines</b></div>
            <div><span>Stake</span><b>£${stakeForType.toFixed(2)}</b></div>
            <div><span>EV</span><b>${typeEv.toFixed(1)}%</b></div>
            <div><span>EV Profit</span><b>${signedMoney(evProfit)}</b></div>
            <div class="debug-gap"><span></span><b></b></div>
          `;
        }).join("");

        const trueSlipEv = totalStakeForAll ? 100 + ((totalEvProfit / totalStakeForAll) * 100) : 0;

        return parts
          + `<div><span>Total Stake</span><b>£${totalStakeForAll.toFixed(2)}</b></div>`
          + `<div><span>Weighted EV Profit</span><b>${signedMoney(totalEvProfit)}</b></div>`
          + `<div><span>True Slip EV</span><b>${trueSlipEv.toFixed(1)}%</b></div>`;
      }

      function lineBreakdownRows(rows, ew) {
        const grouped = {};
        rows.forEach(row => {
          if (!grouped[row.fold]) grouped[row.fold] = 0;
          grouped[row.fold] += 1;
        });
        const win = Object.entries(grouped).map(([fold, count]) => `<div><span>Win ${foldLabel(Number(fold))}</span><b>${count}</b></div>`).join("");
        const place = ew ? Object.entries(grouped).map(([fold, count]) => `<div><span>Place ${foldLabel(Number(fold))}</span><b>${count}</b></div>`).join("") : "";
        return win + (place ? `<div class="debug-gap"><span></span><b></b></div>` + place : "");
      }

      function stakeBreakdownRows(rows, stake, ew) {
        const grouped = {};
        rows.forEach(row => {
          if (!grouped[row.fold]) grouped[row.fold] = 0;
          grouped[row.fold] += 1;
        });

        const winRows = Object.entries(grouped).map(([fold, count]) => {
          const rowStake = (rows.find(r => Number(r.fold) === Number(fold))?.lineStake) || stake;
          const total = count * rowStake;
          return `<div><span>Win ${foldLabel(Number(fold))}</span><b>${count} × £${rowStake.toFixed(2)} = £${total.toFixed(2)}</b></div>`;
        }).join("");

        const placeRows = ew ? Object.entries(grouped).map(([fold, count]) => {
          const rowStake = (rows.find(r => Number(r.fold) === Number(fold))?.lineStake) || stake;
          const total = count * rowStake;
          return `<div><span>Place ${foldLabel(Number(fold))}</span><b>${count} × £${rowStake.toFixed(2)} = £${total.toFixed(2)}</b></div>`;
        }).join("") : "";

        const totalStake = rows.reduce((sum, row) => sum + ((row.lineStake || stake) * (ew ? 2 : 1)), 0);
        return winRows
          + (placeRows ? `<div class="debug-gap"><span></span><b></b></div>` + placeRows : "")
          + `<div class="debug-gap"><span></span><b></b></div>`
          + `<div><span>Total</span><b>£${totalStake.toFixed(2)}</b></div>`;
      }

      const byFold = {};
      calcRows.forEach(row => {
        if (!byFold[row.fold]) byFold[row.fold] = [];
        byFold[row.fold].push(row);
      });

      const foldSummaries = Object.entries(byFold).map(([fold, rows]) => {
        const avgWin = rows.reduce((s, r) => s + r.calc.winEv, 0) / rows.length;
        const avgPlace = rows.reduce((s, r) => s + r.calc.placeEv, 0) / rows.length;
        const avgTotal = (avgWin + avgPlace) / 2;
        return { fold: Number(fold), rows, avgWin, avgPlace, avgTotal };
      });

      summaryEl.innerHTML = `
        <div class="summary-tile expandable-summary" onclick="this.classList.toggle('open')">
          <strong>${betMakerState.ew ? totalBets * 2 : totalBets}</strong><span>Lines ▾</span>
          <div class="summary-breakdown">${lineBreakdownRows(calcRows, betMakerState.ew)}</div>
        </div>
        <div class="summary-tile expandable-summary" onclick="this.classList.toggle('open')">
          <strong>£${totalStake.toFixed(2)}</strong><span>Stake ▾</span>
          <div class="summary-breakdown">${stakeBreakdownRows(calcRows, stake, betMakerState.ew)}</div>
        </div>
        <div class="summary-tile potential-tile" onclick="this.classList.toggle('open')">
          <strong>£${Math.round(potential).toLocaleString('en-GB')}</strong>
          <span>Potential ▾</span>
          <div class="potential-breakdown">
            <div><span>Win</span><b>£${Math.round(potentialWin).toLocaleString('en-GB')}</b></div>
            <div><span>Place</span><b>£${Math.round(potentialPlace).toLocaleString('en-GB')}</b></div>
            <div><span>Total</span><b>£${Math.round(potential).toLocaleString('en-GB')}</b></div>
          </div>
        </div>
        <div class="summary-tile ${evClass(avgWinEv)}"><strong>${avgWinEv.toFixed(1)}%</strong><span>Win EV</span></div>
        <div class="summary-tile ${evClass(avgPlaceEv)}"><strong>${avgPlaceEv.toFixed(1)}%</strong><span>Place EV</span></div>
        <div class="summary-tile ${evClass(trueEv)} expandable-summary" onclick="this.classList.toggle('open')">
          <strong>${trueEv.toFixed(1)}%</strong><span>Total EV ▾</span>
          <div class="summary-breakdown">${totalEvBreakdownRows(calcRows)}</div>
        </div>
      `;

      const warning = invalidCount ? `<div class="same-race-warning">⚠ ${invalidCount} same race/event combo${invalidCount === 1 ? "" : "s"} removed.</div>` : "";

      if (!calcRows.length) {
        matrixEl.innerHTML = warning + `<div class="empty-state">No valid combinations available.</div>`;
        return;
      }

      matrixEl.innerHTML = warning + foldSummaries.map(group => `
        <div class="fold-group">
          <div class="fold-header" onclick="this.closest('.fold-group').classList.toggle('open')">
            <strong><span class="expand-mark">▾</span>${foldLabel(group.fold)}</strong>
            <span>${group.rows.length} bets</span>
            <span class="ev-chip ${evClass(group.avgTotal)}">${group.avgTotal.toFixed(1)}%</span>
          </div>
          <div class="fold-content">
            ${group.rows.map(({ fold, combo, calc, lineStake }) => {
              const names = combo.map(r => r.horse).join(" + ");
              return `
                <div class="combo-card">
                  <div class="combo-top" onclick="this.closest('.combo-card').classList.toggle('open')">
                    <span class="combo-name"><span class="expand-mark">▾</span>${names}</span>
                    <span>${calc.winOdds.toFixed(2)}</span>
                    <span class="ev-chip ${evClass(calc.totalEv)}">${calc.totalEv.toFixed(1)}%</span>
                  </div>
                  <div class="meet-under">Win £${calc.winReturn.toFixed(2)}${betMakerState.ew ? ` · Place £${calc.placeReturn.toFixed(2)}` : ""}</div>
                  ${renderBreakdown(combo, calc, lineStake)}
                </div>
              `;
            }).join("")}
          </div>
        </div>
      `).join("");
    }


    document.addEventListener("pointerdown", event => {
      const button = event.target.closest(".save-horse, .save-star-button");
      if (!button) return;
      event.stopPropagation();
    }, true);

    document.addEventListener("click", event => {
      const button = event.target.closest(".save-horse, .save-star-button");
      if (!button) return;

      event.preventDefault();
      event.stopPropagation();

      if (button.dataset.key) {
        toggleSelection(button.dataset.key);
      }
    }, true);

    document.addEventListener("click", event => {
      const details = event.target.closest(".more.market-more, .more.selection-more");
      if (!details || event.target.closest(".remove-confirm, .selection-edit-actions, .selection-edit-panel, .selection-edit-btn, .selection-save-btn, .selection-cancel-btn, .selection-reset-btn")) return;
      const card = details.closest(".card");
      if (!card) return;
      event.preventDefault();
      event.stopPropagation();
      card.classList.toggle("open");
    }, true);

    document.getElementById("filtersToggle").addEventListener("click", () => {
      filters.classList.toggle("open");
    });

    const sortByDirectFixV4 = document.getElementById("sortBy");
    if (sortByDirectFixV4) {
      sortByDirectFixV4.addEventListener("change", event => {
        event.stopPropagation();
        render();
      });
      sortByDirectFixV4.addEventListener("click", event => event.stopPropagation());
    }

    const sortByDirectListener = document.getElementById("sortBy");
    if (sortByDirectListener) {
      sortByDirectListener.addEventListener("change", event => {
        event.stopPropagation();
        applyMarketSort();
      });
    }

    const sortByChangeListener = document.getElementById("sortBy");
    if (sortByChangeListener) sortByChangeListener.addEventListener("change", render);

    const shopSelect = document.getElementById("shopSelect");
    if (shopSelect) {
      shopSelect.value = activeShopId;
      shopSelect.addEventListener("change", () => {
        activeShopId = shopSelect.value;
        updateShopVisualState();
        const feed = getActiveFeedConfig();
        activeEdgeId = feed.edges[0].edgeId;
        activeMixedFilterTab = "horse";
        runners = [];
        renderedSheetHash = "";
        latestSheetHash = "";
        pendingCsvText = "";
        populateEdgeSelect();
        applyDefaultSortForEdge();
        populateFilters();
        render();
        renderBetMaker();
        if (document.getElementById("shopSelect")) document.getElementById("shopSelect").value = activeShopId;
    updateShopVisualState();
    populateEdgeSelect();
    loadLiveData();
      });
    }

    const edgeSelectListener = document.getElementById("edgeSelect");
    if (edgeSelectListener) {
      populateEdgeSelect();
      edgeSelectListener.addEventListener("change", () => {
        activeEdgeId = edgeSelectListener.value;
        activeMixedFilterTab = "horse";
        applyDefaultSortForEdge();
        runners = [];
        renderedSheetHash = "";
        latestSheetHash = "";
        pendingCsvText = "";
        populateFilters();
        render();
        renderBetMaker();
        if (document.getElementById("shopSelect")) document.getElementById("shopSelect").value = activeShopId;
    updateShopVisualState();
    populateEdgeSelect();
    loadLiveData();
      });
    }


    document.querySelectorAll("[data-mixed-filter-tab]").forEach(button => {
      button.addEventListener("click", event => {
        event.stopPropagation();
        activeMixedFilterTab = button.dataset.mixedFilterTab || "horse";
        populateFilters();
      });
    });

    const filterInputIds = ["meetFilter", "timeFilter", "minOdds", "maxOdds", "minEv", "minPlaceEv", "dgFilter"];
    filterInputIds.forEach(id => {
      const el = document.getElementById(id);
      if (!el) return;
      const markFiltersDirty = event => {
        if (event) event.stopPropagation();
        workingFilterState[currentFilterKey()] = readFilterInputs();
        syncApplyButton();
      };
      el.addEventListener("input", markFiltersDirty);
      el.addEventListener("change", markFiltersDirty);
    });

    const dgToggle = document.getElementById("dgFilter");
    if (dgToggle) {
      updateDgToggleUI(dgToggle);
      dgToggle.addEventListener("click", event => {
        event.stopPropagation();
        dgToggle.value = String(dgToggle.value || "on") === "on" ? "off" : "on";
        updateDgToggleUI(dgToggle);
        dgToggle.dispatchEvent(new Event("change", { bubbles: true }));
      });
    }

    const applyFiltersBtn = document.getElementById("applyFilters");
    if (applyFiltersBtn) {
      applyFiltersBtn.addEventListener("click", async event => {
        event.stopPropagation();
        const key = currentFilterKey();
        const before = normaliseFilters(savedFilterState[key] || currentFilterDefaults(), isMixedEdge() ? activeMixedFilterTab : null);
        const next = normaliseFilters(workingFilterState[key] || readFilterInputs(), isMixedEdge() ? activeMixedFilterTab : null);
        savedFilterState[key] = next;
        localStorage.setItem("marketFilterStateByEdgeType", JSON.stringify(savedFilterState));
        syncApplyButton();
        if ((isGolfEdge() || isMixedEdge()) && String(before.dg || "on") !== String(next.dg || "on")) {
          await loadLiveData();
          return;
        }
        render();
        renderBetMaker();
      });
    }

    document.getElementById("clearFilters").addEventListener("click", event => {
      event.stopPropagation();
      const key = currentFilterKey();
      workingFilterState[key] = normaliseFilters(currentFilterDefaults(), isMixedEdge() ? activeMixedFilterTab : null);
      writeFilterInputs(workingFilterState[key]);
      syncApplyButton();
    });

    const refreshBtn = document.getElementById("refreshBtn");
    if (refreshBtn) {
      updateRefreshButton();
      refreshBtn.addEventListener("click", async () => {
        if (Date.now() < refreshCooldownUntil || refreshBtn.disabled) return;
        startRefreshCooldown(60000);
        await loadLiveData();
      });
    }

    document.querySelectorAll(".nav-item").forEach(item => {
      item.addEventListener("click", () => showScreen(item.dataset.nav));
    });

    document.querySelectorAll("[data-calc-mode]").forEach(button => {
      button.addEventListener("click", () => {
        betMakerState.calcMode = button.dataset.calcMode || "basic";
        document.querySelectorAll("[data-calc-mode]").forEach(btn => btn.classList.toggle("active", btn === button));
        const betType = document.getElementById("betType");
        if (betType) betType.value = betMakerState.calcMode === "advanced" ? "combo" : "double";
        betMakerState.selectedKeys = [];
        betMakerState.activePickerIndex = null;
        renderBetMaker();
      });
    });

    const betTypeChangeListener = document.getElementById("betType");
    if (betTypeChangeListener) betTypeChangeListener.addEventListener("change", renderBetMaker);

    const betStakeChangeListener = document.getElementById("betStake");
    if (betStakeChangeListener) {
      betStakeChangeListener.addEventListener("input", renderBetMaker);
      betStakeChangeListener.addEventListener("change", renderBetMaker);
    }

    const comboPickCountListener = document.getElementById("comboPickCount");
    if (comboPickCountListener) comboPickCountListener.addEventListener("change", renderBetMaker);

    document.querySelectorAll("[data-autofill]").forEach(button => {
      button.addEventListener("click", () => autoFillBetMaker(button.dataset.autofill));
    });

    document.querySelectorAll(".fill-source").forEach(button => {
      button.addEventListener("click", () => {
        betMakerState.sourceMode = button.dataset.source || "market";
        if (betMakerState.sourceMode === "selections") ensureBetSavedSourceFilters();
        document.querySelectorAll(".fill-source").forEach(b => b.classList.toggle("active", b === button));
        betMakerState.selectedKeys = [];
        renderBetMaker();
      });
    });

    const clearBetMakerListener = document.getElementById("clearBetMaker");
    if (clearBetMakerListener) {
      clearBetMakerListener.addEventListener("click", () => {
        clearTimeout(betMakerState.clearTimer);
        betMakerState.selectedKeys = [];
        betMakerState.clearingSlots = [];
        betMakerState.animateSlots = [];
        renderBetMaker();
      });
    }

    const saveBetSlipListener = document.getElementById("saveBetSlip");
    if (saveBetSlipListener) {
      saveBetSlipListener.addEventListener("click", () => {
        saveCurrentBetSlip();
      });
    }

    const ewToggle = document.getElementById("ewToggle");
    if (ewToggle) {
      betMakerState.ew = true;
      ewToggle.classList.add("on");
      ewToggle.textContent = "E/W ON";
      ewToggle.setAttribute("aria-disabled", "true");
    }

    if (document.getElementById("shopSelect")) document.getElementById("shopSelect").value = activeShopId;
    updateShopVisualState();
    populateEdgeSelect();
    loadLiveData();
    renderSelections();
    updateSelectionNavState();
    updateMyBetsNavState();
  


  const APP_VERSION = "1.4.0";

  function compareVersions(a, b) {
    const pa = String(a || "0").split('.').map(n => parseInt(n, 10) || 0);
    const pb = String(b || "0").split('.').map(n => parseInt(n, 10) || 0);
    const len = Math.max(pa.length, pb.length);
    for (let i = 0; i < len; i++) {
      if ((pb[i] || 0) > (pa[i] || 0)) return 1;
      if ((pb[i] || 0) < (pa[i] || 0)) return -1;
    }
    return 0;
  }

  function showUpdateBanner(latestVersion, latestName) {
    let banner = document.getElementById('appUpdateBanner');
    if (!banner) {
      banner = document.createElement('div');
      banner.id = 'appUpdateBanner';
      banner.className = 'app-update-banner';
      banner.innerHTML = `
        <div class="app-update-copy">
          <strong>Update available</strong>
          <span></span>
        </div>
        <div class="app-update-actions">
          <button type="button" class="later">Later</button>
          <button type="button" class="reload">Reload</button>
        </div>`;
      document.body.appendChild(banner);
      banner.querySelector('.later').addEventListener('click', () => banner.classList.remove('show'));
      banner.querySelector('.reload').addEventListener('click', () => window.location.reload());
    }
    const label = latestName ? `v${latestVersion} · ${latestName}` : `v${latestVersion}`;
    banner.querySelector('.app-update-copy span').textContent = `${label} is ready.`;
    banner.classList.add('show');
  }

  async function checkForAppUpdate() {
    try {
      const res = await fetch(`./version.json?v=${Date.now()}`, { cache: 'no-store' });
      if (!res.ok) return;
      const data = await res.json();
      if (data && data.version && compareVersions(APP_VERSION, data.version) > 0) {
        showUpdateBanner(data.version, data.name || '');
      }
    } catch (err) {}
  }

  window.addEventListener('load', () => {
    checkForAppUpdate();
    setInterval(checkForAppUpdate, 5 * 60 * 1000);
  });
  document.addEventListener('visibilitychange', () => {
    if (!document.hidden) checkForAppUpdate();
  });



    if ('serviceWorker' in navigator) {
      window.addEventListener('load', () => {
        navigator.serviceWorker.register('./service-worker.js').catch(() => {});
      });
    }
