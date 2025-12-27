(() => {
    // --- SITE DETECTION ---
    const isMangaPark = window.location.hostname.includes('mangapark');
    const isHistoryPage = window.location.pathname === '/my/history';
    const isListPage = window.location.pathname.includes('/batolists') || window.location.pathname.includes('/mplists');
    
    const SITE_CONFIG = isMangaPark ? {
        name: 'MangaPark',
        apiEndpoint: '/apo/',
        listType: 'mplist',
        listUrlPart: 'mplists',
        buttonSelector: 'a[href$="/mplists/liked"]',
        baseUrl: 'https://mangapark.io'
    } : {
        name: 'Bato',
        apiEndpoint: '/ap2/',
        listType: 'batolist',
        listUrlPart: 'batolists',
        buttonSelector: 'a[href$="/batolists/liked"]',
        baseUrl: 'https://bato.to'
    };

    // --- CONFIGURATION ---
    const API_ENDPOINT = SITE_CONFIG.apiEndpoint;
    const COMIC_FETCH_LIMIT = 5000; 
    const HISTORY_FETCH_LIMIT = 300;
    const HISTORY_EXPORT_LIMIT = 100; // Items per page for full history export

    // --- GRAPHQL QUERIES ---
    const LIST_QUERY = `
    query get_user_mylistList($select: MylistList_Select) {
        get_user_mylistList(select: $select) {
            paging { total pages page init size skip limit prev next }
            items {
                id
                data {
                    name
                    isPublic
                    comicNodes(amount: ${COMIC_FETCH_LIMIT}) {
                        data {
                            id
                            name
                            urlPath
                            urlCover600
                            genres
                            authors
                            score_avg
                            uploadStatus
                            originalStatus
                            origLang
                            dateUpdate
                            dateModify
                            chaps_normal
                        }
                    }
                }
            }
        }
    }`;

    const HISTORY_QUERY = `
    query get_sser_myHistory($select: Sser_MyHistory_Select) {
        get_sser_myHistory(select: $select) {
            reqLimit
            newStart
            items {
                date
                comicNode { id }
                chapterNode {
                    data {
                        dname
                        title
                        order
                    }
                }
            }
        }
    }`;

    // Full history query with complete comic data for history page export
    // Note: Bato uses chapterNodes_last, MangaPark uses last_chapterNodes
    const HISTORY_FULL_QUERY_BATO = `
    query get_sser_myHistory($select: Sser_MyHistory_Select) {
        get_sser_myHistory(select: $select) {
            reqLimit
            newStart
            items {
                date
                comicNode {
                    id
                    data {
                        id
                        name
                        urlPath
                        urlCover600
                        genres
                        authors
                        artists
                        score_avg
                        uploadStatus
                        originalStatus
                        origLang
                        dateUpdate
                        dateModify
                        chaps_normal
                        chapterNodes_last(amount: 1) {
                            id
                            data {
                                dname
                                title
                                serial
                            }
                        }
                    }
                }
                chapterNode {
                    id
                    data {
                        dname
                        title
                        serial
                        urlPath
                    }
                }
            }
        }
    }`;

    const HISTORY_FULL_QUERY_MANGAPARK = `
    query get_sser_myHistory($select: Sser_MyHistory_Select) {
        get_sser_myHistory(select: $select) {
            reqLimit
            newStart
            items {
                date
                comicNode {
                    id
                    data {
                        id
                        name
                        urlPath
                        urlCover600
                        genres
                        authors
                        artists
                        score_avg
                        uploadStatus
                        originalStatus
                        origLang
                        dateUpdate
                        dateModify
                        chaps_normal
                        last_chapterNodes(amount: 1) {
                            id
                            data {
                                dname
                                title
                                serial
                            }
                        }
                    }
                }
                chapterNode {
                    id
                    data {
                        dname
                        title
                        serial
                        urlPath
                    }
                }
            }
        }
    }`;

    const HISTORY_FULL_QUERY = isMangaPark ? HISTORY_FULL_QUERY_MANGAPARK : HISTORY_FULL_QUERY_BATO;

    // --- UTILITIES ---
    function getUserId() {
        const match = window.location.pathname.match(/\/u\/(\d+)/);
        return match ? match[1] : null;
    }

    function downloadFile(content, filename, mimeType) {
        const blob = new Blob([content], { type: mimeType });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = filename;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
    }

    function timeAgo(timestamp) {
        if (!timestamp) return 'Unknown';
        const seconds = Math.floor((new Date() - new Date(timestamp)) / 1000);
        
        let interval = seconds / 31536000;
        if (interval > 1) return Math.floor(interval) + "y ago";
        interval = seconds / 2592000;
        if (interval > 1) return Math.floor(interval) + "mo ago";
        interval = seconds / 86400;
        if (interval > 1) return Math.floor(interval) + "d ago";
        interval = seconds / 3600;
        if (interval > 1) return Math.floor(interval) + "h ago";
        interval = seconds / 60;
        if (interval > 1) return Math.floor(interval) + "m ago";
        return "Just now";
    }

    function formatLang(code) {
        const map = {
            'ko': 'Manhwa',
            'ja': 'Manga',
            'zh': 'Manhua',
            'en': 'Comic'
        };
        return map[code] || 'Comic';
    }

    // Sanitize objects to remove XrayWrappers in Firefox
    function cleanObject(obj) {
        if (!obj) return obj;
        return JSON.parse(JSON.stringify(obj));
    }

    // --- IMAGE PROCESSOR ---
    async function fetchImageAsBase64(url) {
        try {
            const response = await fetch(url);
            const blob = await response.blob();
            return new Promise((resolve) => {
                const reader = new FileReader();
                reader.onloadend = () => resolve(reader.result);
                reader.onerror = () => resolve(null);
                reader.readAsDataURL(blob);
            });
        } catch (e) {
            return null;
        }
    }

    async function processCoversWithQueue(lists, updateStatusFn) {
        const queue = [];
        lists.forEach(list => {
            if (list.data.comicNodes) {
                list.data.comicNodes.forEach(node => {
                    if (node.data.urlCover600) {
                        queue.push(node.data);
                    }
                });
            }
        });

        const total = queue.length;
        let processed = 0;
        const CONCURRENCY = 5; 

        updateStatusFn(`Downloading covers: 0/${total}`);

        async function worker() {
            while (queue.length > 0) {
                const comicData = queue.shift();
                const absUrl = window.location.origin + comicData.urlCover600;
                comicData.base64Cover = await fetchImageAsBase64(absUrl);
                processed++;
                if (processed % 5 === 0 || processed === total) {
                    updateStatusFn(`Downloading covers: ${processed}/${total}`);
                }
            }
        }

        const workers = Array(CONCURRENCY).fill(null).map(() => worker());
        await Promise.all(workers);
    }

    // Process covers for history items (flat array)
    async function processHistoryCovers(items, updateStatusFn) {
        const queue = items.filter(item => item.urlCover600);
        const total = queue.length;
        let processed = 0;
        const CONCURRENCY = 5;

        updateStatusFn(`Downloading covers: 0/${total}`);

        async function worker() {
            while (queue.length > 0) {
                const item = queue.shift();
                const absUrl = window.location.origin + item.urlCover600;
                item.base64Cover = await fetchImageAsBase64(absUrl);
                processed++;
                if (processed % 5 === 0 || processed === total) {
                    updateStatusFn(`Downloading covers: ${processed}/${total}`);
                }
            }
        }

        const workers = Array(CONCURRENCY).fill(null).map(() => worker());
        await Promise.all(workers);
    }


    // --- FORMAT 1: HTML (Web / Visual) ---
    function generateHTML(lists, username, date, editable = false) {
        // Collect unique values for filters
        const allStatuses = new Set();
        const allLangs = new Set();
        const allTags = new Set();
        const allListNames = [];
        
        lists.forEach(list => {
            if (list.data.comicNodes && list.data.comicNodes.length > 0) {
                allListNames.push(list.data.name);
                list.data.comicNodes.forEach(node => {
                    const nodeStatus = node.data.uploadStatus || node.data.originalStatus;
                    if (nodeStatus) allStatuses.add(nodeStatus);
                    if (node.data.origLang) allLangs.add(node.data.origLang);
                    (node.data.genres || []).forEach(g => allTags.add(g.toLowerCase()));
                });
            }
        });

        const langOptions = [...allLangs].sort().map(l => {
            const label = formatLang(l).replace(/[^\w\s]/g, '').trim();
            return '<option value="' + l + '">' + label + '</option>';
        }).join('');
        
        const tagOptions = [...allTags].sort().map(t => {
            return '<option value="' + t + '">' + t + '</option>';
        }).join('');

        let html = `<!DOCTYPE html>
<html lang="en" data-theme="mdark">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>${username}'s ${SITE_CONFIG.name} Lists</title>
<style>
    :root { --b1: #161616; --b2: #1c1c1c; --b3: #252525; --bc: #eee; --pc: #fff; --p: #00bcd4; --er: #f87171; --su: #4ade80; --wa: #facc15; }
    * { box-sizing: border-box; }
    body { background-color: var(--b1); color: var(--bc); font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen, Ubuntu, sans-serif; margin: 0; padding: 20px; font-size: 14px; line-height: 1.5; }
    a { text-decoration: none; color: inherit; }
    .container { max-width: 1000px; margin: 0 auto; }
    .header-main { border-bottom: 1px solid #333; padding-bottom: 15px; margin-bottom: 20px; }
    h1 { color: var(--p); margin: 0; font-size: 24px; }
    
    .filter-bar { background: var(--b2); border: 1px solid #333; border-radius: 8px; padding: 16px; margin-bottom: 24px; display: flex; flex-wrap: wrap; gap: 12px; align-items: center; }
    .search-box { flex: 1; min-width: 200px; position: relative; }
    .search-box input { width: 100%; padding: 10px 14px 10px 38px; border: 1px solid #444; border-radius: 6px; background: var(--b3); color: #fff; font-size: 14px; box-sizing: border-box; }
    .search-box input:focus { outline: none; border-color: var(--p); }
    .search-box input::placeholder { color: #666; }
    .search-icon { position: absolute; left: 12px; top: 50%; transform: translateY(-50%); width: 16px; height: 16px; color: #666; }
    .filter-select { padding: 10px 12px; border: 1px solid #444; border-radius: 6px; background: var(--b3); color: #fff; font-size: 14px; font-family: inherit; cursor: pointer; min-width: 120px; }
    .filter-select:focus { outline: none; border-color: var(--p); }
    .filter-select option { background: var(--b2); }
    .clear-btn { padding: 10px 16px; border: 1px solid #555; border-radius: 6px; background: transparent; color: #aaa; font-size: 14px; font-family: inherit; cursor: pointer; transition: all 0.2s; }
    .clear-btn:hover { border-color: var(--er); color: var(--er); }
    .results-count { font-size: 14px; color: #888; padding: 8px 0; }
    .list-block { background: var(--b2); border: 1px solid #333; border-radius: 8px; margin-bottom: 40px; overflow: hidden; }
    .list-block.hidden { display: none; }
    .list-head { background: #222; padding: 15px; border-bottom: 1px solid #333; display: flex; justify-content: space-between; align-items: center; }
    .list-name { font-size: 18px; font-weight: bold; color: #fff; }
    .list-meta { font-size: 12px; color: #888; background: #111; padding: 4px 10px; border-radius: 4px; text-transform: uppercase; letter-spacing: 0.5px; }
    .list-count { margin-left: 8px; }
    .item { display: flex; padding: 12px; border-bottom: 1px solid #2a2a2a; transition: background 0.2s; }
    .item.hidden { display: none; }
    .item:last-child { border-bottom: none; }
    .item:hover { background: var(--b3); }
    .cover-box { width: 90px; flex-shrink: 0; margin-right: 15px; position: relative; border-radius: 4px; overflow: hidden; background: #000; aspect-ratio: 2/3; }
    .cover-img { width: 100%; height: 100%; object-fit: cover; }
    .details { flex-grow: 1; display: flex; flex-direction: column; justify-content: flex-start; overflow: hidden; }
    .comic-title { font-weight: bold; font-size: 16px; color: #fff; margin-bottom: 4px; display: block; }
    .stats-row { display: flex; align-items: center; gap: 12px; font-size: 14px; color: #aaa; margin-bottom: 6px; }
    .star-val { color: var(--wa); font-weight: bold; }
    .status-ongoing { color: var(--su); }
    .status-completed { color: #60a5fa; }
    .meta-row { font-size: 13px; line-height: 1.5; margin-bottom: 6px; color: #999; }
    .lang-tag { font-weight: bold; color: #fff; margin-right: 6px; }
    .tags-row { display: flex; flex-wrap: wrap; font-size: 12px; color: rgba(255,255,255,0.7); margin-bottom: 8px; line-height: 1.6; }
    .tag { margin-right: 2px; }
    .tag::after { content: ','; }
    .tag:last-child::after { content: ''; }
    .tag-bold { font-weight: bold; }
    .tag-warning { font-weight: bold; border-bottom: 1px solid var(--wa); }
    .tag-primary { font-weight: bold; border-bottom: 1px solid var(--p); }
    .chapter-row { margin-top: auto; font-size: 14px; border-top: 1px dashed #333; padding-top: 8px; display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
    .chap-info { display: flex; flex-direction: column; gap: 2px; }
    .chap-label { font-size: 11px; color: #666; text-transform: uppercase; font-weight: 600; letter-spacing: 0.5px; }
    .chap-val { color: var(--p); font-weight: 600; }
    .chap-date { color: #666; font-size: 12px; }
    .history-box { color: var(--wa); }
    .no-image { display: flex; justify-content: center; align-items: center; height: 100%; color: #444; font-size: 12px; text-align: center; }
    .no-results { text-align: center; padding: 40px; color: #666; font-size: 16px; }
    
    @media (max-width: 600px) {
        .filter-bar { flex-direction: column; }
        .search-box { width: 100%; }
        .filter-select { width: 100%; }
    }
</style>
${editable ? `<style>
    .action-bar { display: flex; justify-content: space-between; align-items: center; background: var(--b2); border: 1px solid #333; border-radius: 8px; padding: 12px 16px; margin-bottom: 16px; flex-wrap: wrap; gap: 12px; }
    .action-buttons { display: flex; gap: 10px; flex-wrap: wrap; }
    .action-btn { display: inline-flex; align-items: center; gap: 6px; background: var(--b3); border: 1px solid #444; color: #ccc; padding: 8px 14px; border-radius: 6px; cursor: pointer; font-size: 13px; font-family: inherit; transition: all 0.2s; }
    .action-btn svg { width: 16px; height: 16px; }
    .action-btn:hover { border-color: var(--p); color: var(--p); }
    .action-btn-primary { border-color: var(--su); color: var(--su); }
    .action-btn-primary:hover { background: var(--su); color: #000; }
    .action-btn-save { border-color: var(--p); color: var(--p); }
    .action-btn-save:hover { background: var(--p); color: #000; }
    .edit-notice { display: flex; align-items: center; gap: 8px; font-size: 13px; color: #888; }
    .notice-icon { width: 16px; height: 16px; color: var(--p); }
    .unsaved-badge { display: none; background: var(--wa); color: #000; font-size: 11px; padding: 2px 6px; border-radius: 4px; font-weight: bold; }
    .unsaved-badge.show { display: inline-block; }
    .edit-btn, .delete-btn { background: transparent; border: 1px solid #444; color: #aaa; padding: 4px 8px; border-radius: 4px; cursor: pointer; font-size: 12px; transition: all 0.2s; }
    .edit-btn:hover { border-color: var(--p); color: var(--p); }
    .delete-btn:hover { border-color: var(--er); color: var(--er); }
    .item-actions { display: flex; gap: 6px; margin-top: 8px; }
    .modal-overlay { position: fixed; top: 0; left: 0; right: 0; bottom: 0; background: rgba(0,0,0,0.8); display: none; justify-content: center; align-items: center; z-index: 1000; }
    .modal-overlay.open { display: flex; }
    .modal-box { background: var(--b2); border: 1px solid #444; border-radius: 12px; padding: 24px; width: 90%; max-width: 500px; max-height: 90vh; overflow-y: auto; }
    .modal-title { font-size: 18px; font-weight: bold; margin-bottom: 16px; color: #fff; }
    .form-group { margin-bottom: 14px; }
    .form-label { display: block; font-size: 12px; color: #888; margin-bottom: 4px; text-transform: uppercase; }
    .form-input, .form-select { width: 100%; padding: 10px; border: 1px solid #444; border-radius: 6px; background: var(--b3); color: #fff; font-size: 14px; box-sizing: border-box; }
    .form-input:focus, .form-select:focus { outline: none; border-color: var(--p); }
    .form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
    .modal-actions { display: flex; gap: 10px; justify-content: flex-end; margin-top: 20px; }
    .btn-save { background: var(--p); color: #000; border: none; padding: 10px 20px; border-radius: 6px; cursor: pointer; font-weight: bold; }
    .btn-cancel { background: transparent; border: 1px solid #555; color: #aaa; padding: 10px 20px; border-radius: 6px; cursor: pointer; }
    @media (max-width: 600px) { .action-bar { flex-direction: column; align-items: stretch; } .action-buttons { justify-content: center; } }
</style>` : ''}
</head>
<body>
<div class="container">
    <div class="header-main">
        <h1>${username}'s ${SITE_CONFIG.name} Collection</h1>
        <div style="color:#777; font-size:12px">Exported: ${date}</div>
    </div>
    
    <div class="filter-bar">
        <div class="search-box">
            <svg class="search-icon" viewBox="0 0 512 512" fill="currentColor"><path d="M416 208c0 45.9-14.9 88.3-40 122.7L502.6 457.4c12.5 12.5 12.5 32.8 0 45.3s-32.8 12.5-45.3 0L330.7 376c-34.4 25.2-76.8 40-122.7 40C93.1 416 0 322.9 0 208S93.1 0 208 0S416 93.1 416 208zM208 352a144 144 0 1 0 0-288 144 144 0 1 0 0 288z"/></svg>
            <input type="text" id="searchInput" placeholder="Search by title or author...">
        </div>
        <select class="filter-select" id="listFilter">
            <option value="">All Lists</option>
            ${allListNames.map(name => '<option value="' + name + '">' + name + '</option>').join('')}
        </select>
        <select class="filter-select" id="statusFilter">
            <option value="">All Status</option>
            ${[...allStatuses].sort().map(s => '<option value="' + s + '">' + s + '</option>').join('')}
        </select>
        <select class="filter-select" id="langFilter">
            <option value="">All Types</option>
            ${langOptions}
        </select>
        <select class="filter-select" id="tagFilter">
            <option value="">All Tags</option>
            ${tagOptions}
        </select>
        <select class="filter-select" id="sortBy">
            <option value="">Default Order</option>
            <option value="title">Title A-Z</option>
            <option value="title-desc">Title Z-A</option>
            <option value="score">Highest Score</option>
            <option value="chapters">Most Chapters</option>
            <option value="lastread">Last Read (Recent)</option>
            <option value="updated">Recently Updated</option>
        </select>
        <button class="clear-btn" id="clearFilters">Clear</button>
    </div>
${editable ? `<div class="action-bar">
        <div class="edit-notice"><svg class="notice-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7"/><path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z"/></svg>Editable mode<span class="unsaved-badge" id="unsavedBadge">Unsaved changes</span></div>
        <div class="action-buttons">
            <button class="action-btn action-btn-primary" id="addEntryBtn"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><line x1="12" y1="5" x2="12" y2="19"/><line x1="5" y1="12" x2="19" y2="12"/></svg>Add Entry</button>
            <button class="action-btn action-btn-save" id="saveChangesBtn"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M19 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h11l5 5v11a2 2 0 0 1-2 2z"/><polyline points="17 21 17 13 7 13 7 21"/><polyline points="7 3 7 8 15 8"/></svg>Save Changes</button>
            <button class="action-btn" id="exportJsonBtn"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="7 10 12 15 17 10"/><line x1="12" y1="15" x2="12" y2="3"/></svg>Export JSON</button>
        </div>
    </div>` : ''}
    <div class="results-count" id="resultsCount"></div>`;

        lists.forEach(list => {
            const items = list.data.comicNodes || [];
            if (items.length === 0) return;

            html += `<div class="list-block" data-list="${list.data.name}"><div class="list-head"><span class="list-name">${list.data.name}</span><span class="list-meta">${list.data.isPublic ? 'Public' : 'Private'} &bull; <span class="list-count">${items.length}</span> items</span></div>`;

            items.forEach(node => {
                const c = node.data;
                const score = c.score_avg ? c.score_avg.toFixed(1) : '-';
                const status = c.uploadStatus || c.originalStatus || 'Unknown';
                const statusClass = status.toLowerCase() === 'completed' ? 'status-completed' : 'status-ongoing';
                const updateDate = c.dateUpdate || c.dateModify;
                const latestDate = timeAgo(updateDate);
                const latestChap = c.chaps_normal ? `Ch.${c.chaps_normal}` : 'Unknown';
                const authors = (c.authors||[]).join(', ');
                const escapedTitle = c.name.replace(/"/g, '&quot;').toLowerCase();
                const escapedAuthors = authors.replace(/"/g, '&quot;').toLowerCase();
                
                // Format tags with special styling like Bato
                const warningTags = ['adult', 'mature', 'smut', 'gore', 'sexual violence', 'hentai'];
                const demographicTags = ['josei', 'josei(w)', 'seinen', 'shoujo', 'shounen', 'kodomo'];
                const langName = { 'ko': 'Manhwa', 'ja': 'Manga', 'zh': 'Manhua', 'en': 'Comic' };
                
                let tagsHtml = '';
                if (c.origLang && langName[c.origLang]) {
                    tagsHtml += `<span class="tag tag-bold">${langName[c.origLang]}</span>`;
                }
                (c.genres || []).forEach(g => {
                    const gLower = g.toLowerCase();
                    let tagClass = 'tag';
                    if (warningTags.includes(gLower)) tagClass = 'tag tag-warning';
                    else if (demographicTags.includes(gLower)) tagClass = 'tag tag-primary';
                    tagsHtml += `<span class="${tagClass}">${g}</span>`;
                });
                
                let historyHtml = '<span class="chap-val">-</span>';
                if (c.history) {
                    historyHtml = `<span class="chap-val" style="color:#facc15"> ${c.history.chapterName}</span><span class="chap-date">${timeAgo(c.history.readDate)}</span>`;
                }

                const imgSrc = c.base64Cover; // Only use embedded covers, not URLs
                const coverHtml = imgSrc ? `<div class="cover-box"><img class="cover-img" src="${imgSrc}" loading="lazy"></div>` : '';

                const tagsLower = (c.genres || []).map(g => g.toLowerCase()).join(',');
                const scoreNum = c.score_avg || 0;
                const chaptersNum = c.chaps_normal || 0;
                const lastReadDate = c.history ? c.history.readDate : 0;
                const updateDateNum = updateDate || 0;
                const itemId = c.id || Math.random().toString(36).substr(2, 9);
                const actionsHtml = editable ? `<div class="item-actions"><button class="edit-btn" data-id="${itemId}">Edit</button><button class="delete-btn" data-id="${itemId}">Delete</button></div>` : '';

                html += `<div class="item" data-id="${itemId}" data-list="${list.data.name}" data-title="${escapedTitle}" data-authors="${escapedAuthors}" data-status="${status}" data-lang="${c.origLang || ''}" data-tags="${tagsLower}" data-score="${scoreNum}" data-chapters="${chaptersNum}" data-lastread="${lastReadDate}" data-updated="${updateDateNum}">${coverHtml}<div class="details"><div class="comic-title">${c.name}</div><div class="stats-row"><span>&#9733; <span class="star-val">${score}</span></span><span class="${statusClass}">${status}</span></div><div class="meta-row"><span>${authors}</span></div><div class="tags-row">${tagsHtml}</div><div class="chapter-row"><div class="chap-info history-box"><span class="chap-label">Last Read</span>${historyHtml}</div><div class="chap-info"><span class="chap-label">Latest Release (As of ${date})</span><span class="chap-val">${latestChap}</span><span class="chap-date">${latestDate}</span></div></div>${actionsHtml}</div></div>`;
            });
            html += `</div>`;
        });
        
        html += `
    <div class="no-results" id="noResults" style="display:none;">No comics match your filters</div>
</div>`;

        // Add filter script
        html += generateFilterScript();
        
        // Add editable script if needed
        if (editable) {
            html += generateEditableScript(lists);
        }
        
        html += `</body></html>`;
        return html;
    }


    // Generate filter script for HTML export
    function generateFilterScript() {
        return `
<script>
(function() {
    var searchInput = document.getElementById('searchInput');
    var listFilter = document.getElementById('listFilter');
    var statusFilter = document.getElementById('statusFilter');
    var langFilter = document.getElementById('langFilter');
    var tagFilter = document.getElementById('tagFilter');
    var sortBy = document.getElementById('sortBy');
    var clearBtn = document.getElementById('clearFilters');
    var resultsCount = document.getElementById('resultsCount');
    var noResults = document.getElementById('noResults');
    
    function applyFilters() {
        var search = searchInput.value.toLowerCase().trim();
        var listVal = listFilter.value;
        var statusVal = statusFilter.value;
        var langVal = langFilter.value;
        var tagVal = tagFilter.value.toLowerCase();
        var sortVal = sortBy.value;
        
        var listBlocks = document.querySelectorAll('.list-block');
        var totalVisible = 0;
        
        listBlocks.forEach(function(block) {
            var listName = block.dataset.list;
            var items = Array.from(block.querySelectorAll('.item'));
            var visibleInList = 0;
            var listMatch = !listVal || listName === listVal;
            
            if (sortVal) {
                items.sort(function(a, b) {
                    switch(sortVal) {
                        case 'title': return a.dataset.title.localeCompare(b.dataset.title);
                        case 'title-desc': return b.dataset.title.localeCompare(a.dataset.title);
                        case 'score': return parseFloat(b.dataset.score || 0) - parseFloat(a.dataset.score || 0);
                        case 'chapters': return parseInt(b.dataset.chapters || 0) - parseInt(a.dataset.chapters || 0);
                        case 'lastread': return parseInt(b.dataset.lastread || 0) - parseInt(a.dataset.lastread || 0);
                        case 'updated': return parseInt(b.dataset.updated || 0) - parseInt(a.dataset.updated || 0);
                        default: return 0;
                    }
                });
                var parent = items[0] ? items[0].parentNode : null;
                if (parent) { items.forEach(function(item) { parent.appendChild(item); }); }
            }
            
            items.forEach(function(item) {
                var title = item.dataset.title;
                var authors = item.dataset.authors;
                var status = item.dataset.status;
                var lang = item.dataset.lang;
                var tags = item.dataset.tags || '';
                
                var searchMatch = !search || title.indexOf(search) !== -1 || authors.indexOf(search) !== -1;
                var statusMatch = !statusVal || status === statusVal;
                var langMatch = !langVal || lang === langVal;
                var tagMatch = !tagVal || tags.indexOf(tagVal) !== -1;
                
                if (listMatch && searchMatch && statusMatch && langMatch && tagMatch) {
                    item.classList.remove('hidden');
                    visibleInList++;
                    totalVisible++;
                } else {
                    item.classList.add('hidden');
                }
            });
            
            var countEl = block.querySelector('.list-count');
            if (countEl) countEl.textContent = visibleInList;
            
            if (visibleInList === 0 || !listMatch) { block.classList.add('hidden'); }
            else { block.classList.remove('hidden'); }
        });
        
        var hasFilters = search || listVal || statusVal || langVal || tagVal || sortVal;
        resultsCount.textContent = hasFilters ? totalVisible + ' comic' + (totalVisible !== 1 ? 's' : '') + ' found' : '';
        noResults.style.display = totalVisible === 0 && hasFilters ? 'block' : 'none';
    }
    
    searchInput.addEventListener('input', applyFilters);
    listFilter.addEventListener('change', applyFilters);
    statusFilter.addEventListener('change', applyFilters);
    langFilter.addEventListener('change', applyFilters);
    tagFilter.addEventListener('change', applyFilters);
    sortBy.addEventListener('change', applyFilters);
    
    clearBtn.addEventListener('click', function() {
        searchInput.value = '';
        listFilter.value = '';
        statusFilter.value = '';
        langFilter.value = '';
        tagFilter.value = '';
        sortBy.value = '';
        applyFilters();
    });
})();
</script>`;
    }

    // Generate editable script for HTML export
    function generateEditableScript(lists) {
        const dataJson = JSON.stringify(lists.map(list => ({
            name: list.data.name,
            isPublic: list.data.isPublic,
            items: (list.data.comicNodes || []).map(node => {
                const c = node.data;
                return {
                    id: c.id || Math.random().toString(36).substr(2, 9),
                    title: c.name,
                    score: c.score_avg || 0,
                    status: c.uploadStatus || c.originalStatus || 'ongoing',
                    lang: c.origLang || 'ko',
                    chapters: c.chaps_normal || 0,
                    authors: (c.authors || []).join(', '),
                    tags: (c.genres || []).join(', '),
                    lastRead: c.history ? c.history.chapterName : '',
                    lastReadDate: c.history ? c.history.readDate : 0,
                    dateUpdate: c.dateUpdate || c.dateModify || 0,
                    cover: c.base64Cover || ''
                };
            })
        })));

        return `
<script id="listData" type="application/json">${dataJson}</script>
<div class="modal-overlay" id="editModal">
    <div class="modal-box">
        <div class="modal-title" id="modalTitle">Add New Entry</div>
        <input type="hidden" id="editId">
        <input type="hidden" id="editListIdx">
        <div class="form-group">
            <label class="form-label">Title *</label>
            <input type="text" class="form-input" id="formTitle" required>
        </div>
        <div class="form-row">
            <div class="form-group">
                <label class="form-label">Score (0-10)</label>
                <input type="number" class="form-input" id="formScore" min="0" max="10" step="0.1">
            </div>
            <div class="form-group">
                <label class="form-label">Status</label>
                <select class="form-select" id="formStatus">
                    <option value="ongoing">Ongoing</option>
                    <option value="completed">Completed</option>
                    <option value="hiatus">Hiatus</option>
                    <option value="cancelled">Cancelled</option>
                </select>
            </div>
        </div>
        <div class="form-row">
            <div class="form-group">
                <label class="form-label">Type</label>
                <select class="form-select" id="formLang">
                    <option value="ko">Manhwa</option>
                    <option value="ja">Manga</option>
                    <option value="zh">Manhua</option>
                    <option value="en">Comic</option>
                </select>
            </div>
            <div class="form-group">
                <label class="form-label">Chapters</label>
                <input type="number" class="form-input" id="formChapters" min="0">
            </div>
        </div>
        <div class="form-group">
            <label class="form-label">Authors (comma separated)</label>
            <input type="text" class="form-input" id="formAuthors">
        </div>
        <div class="form-group">
            <label class="form-label">Tags (comma separated)</label>
            <input type="text" class="form-input" id="formTags" placeholder="action, romance, fantasy">
        </div>
        <div class="form-group">
            <label class="form-label">Last Read Chapter</label>
            <input type="text" class="form-input" id="formLastRead" placeholder="Chapter 50">
        </div>
        <div class="form-group">
            <label class="form-label">Cover Image</label>
            <div style="display: flex; gap: 10px; align-items: center;">
                <div id="coverPreview" style="width: 60px; height: 80px; background: #333; border-radius: 4px; overflow: hidden; flex-shrink: 0;">
                    <img id="coverPreviewImg" src="" style="width: 100%; height: 100%; object-fit: cover; display: none;">
                </div>
                <div style="flex: 1;">
                    <input type="file" id="formCover" accept="image/*" style="display: none;">
                    <button type="button" class="btn-cancel" id="uploadCoverBtn" style="width: 100%; margin-bottom: 6px;">Upload Image</button>
                    <button type="button" class="btn-cancel" id="clearCoverBtn" style="width: 100%; font-size: 11px; padding: 6px;">Clear Cover</button>
                </div>
            </div>
            <input type="hidden" id="formCoverData">
        </div>
        <div class="form-group">
            <label class="form-label">Add to List</label>
            <select class="form-select" id="formList"></select>
        </div>
        <div class="modal-actions">
            <button class="btn-cancel" id="modalCancel">Cancel</button>
            <button class="btn-save" id="modalSave">Save</button>
        </div>
    </div>
</div>
<script>
(function() {
    var DATA = JSON.parse(document.getElementById('listData').textContent);
    var hasChanges = false;
    var modal = document.getElementById('editModal');
    var addBtn = document.getElementById('addEntryBtn');
    var saveBtn = document.getElementById('saveChangesBtn');
    var exportJsonBtn = document.getElementById('exportJsonBtn');
    var unsavedBadge = document.getElementById('unsavedBadge');
    var modalTitle = document.getElementById('modalTitle');
    var modalCancel = document.getElementById('modalCancel');
    var modalSave = document.getElementById('modalSave');
    var formCover = document.getElementById('formCover');
    var uploadCoverBtn = document.getElementById('uploadCoverBtn');
    var clearCoverBtn = document.getElementById('clearCoverBtn');
    var coverPreviewImg = document.getElementById('coverPreviewImg');
    var formCoverData = document.getElementById('formCoverData');
    
    uploadCoverBtn.addEventListener('click', function() { formCover.click(); });
    formCover.addEventListener('change', function(e) {
        var file = e.target.files[0];
        if (file) {
            var reader = new FileReader();
            reader.onload = function(ev) {
                formCoverData.value = ev.target.result;
                coverPreviewImg.src = ev.target.result;
                coverPreviewImg.style.display = 'block';
            };
            reader.readAsDataURL(file);
        }
    });
    clearCoverBtn.addEventListener('click', function() {
        formCover.value = '';
        formCoverData.value = '';
        coverPreviewImg.src = '';
        coverPreviewImg.style.display = 'none';
    });
    
    var formList = document.getElementById('formList');
    DATA.forEach(function(list, idx) {
        var opt = document.createElement('option');
        opt.value = idx;
        opt.textContent = list.name;
        formList.appendChild(opt);
    });
    
    function markChanged() {
        hasChanges = true;
        unsavedBadge.classList.add('show');
    }
    
    function findItemInData(id) {
        for (var i = 0; i < DATA.length; i++) {
            for (var j = 0; j < DATA[i].items.length; j++) {
                if (DATA[i].items[j].id === id) return { listIdx: i, itemIdx: j, item: DATA[i].items[j] };
            }
        }
        return null;
    }
    
    function updateItemDisplay(item, data) {
        item.querySelector('.comic-title').textContent = data.title;
        item.querySelector('.star-val').textContent = data.score || '-';
        var statusEl = item.querySelector('.status-ongoing, .status-completed');
        if (statusEl) {
            statusEl.textContent = data.status;
            statusEl.className = data.status === 'completed' ? 'status-completed' : 'status-ongoing';
        }
        item.querySelector('.meta-row span').textContent = data.authors;
        item.dataset.title = data.title.toLowerCase();
        item.dataset.authors = data.authors.toLowerCase();
        item.dataset.status = data.status;
        item.dataset.lang = data.lang;
        item.dataset.score = data.score || 0;
        item.dataset.chapters = data.chapters || 0;
        item.dataset.tags = data.tags;
        if (data.cover) {
            var coverBox = item.querySelector('.cover-box');
            if (!coverBox) {
                var details = item.querySelector('.details');
                coverBox = document.createElement('div');
                coverBox.className = 'cover-box';
                var img = document.createElement('img');
                img.className = 'cover-img';
                img.loading = 'lazy';
                coverBox.appendChild(img);
                item.insertBefore(coverBox, details);
            }
            coverBox.querySelector('.cover-img').src = data.cover;
        } else {
            var cb = item.querySelector('.cover-box');
            if (cb) cb.remove();
        }
        var tagsRow = item.querySelector('.tags-row');
        if (tagsRow) {
            var langMap = { ko: 'Manhwa', ja: 'Manga', zh: 'Manhua', en: 'Comic' };
            var warningTags = ['adult', 'mature', 'smut', 'gore', 'sexual violence', 'hentai'];
            var demographicTags = ['josei', 'josei(w)', 'seinen', 'shoujo', 'shounen', 'kodomo'];
            var html = '';
            if (data.lang && langMap[data.lang]) {
                html += '<span class="tag tag-bold">' + langMap[data.lang] + '</span>';
            }
            data.tags.split(',').filter(function(t) { return t.trim(); }).forEach(function(t) {
                var tLower = t.trim().toLowerCase();
                var cls = 'tag';
                if (warningTags.indexOf(tLower) !== -1) cls = 'tag tag-warning';
                else if (demographicTags.indexOf(tLower) !== -1) cls = 'tag tag-primary';
                html += '<span class="' + cls + '">' + t.trim() + '</span>';
            });
            tagsRow.textContent = '';
            var parser = new DOMParser();
            var tagDoc = parser.parseFromString('<div>' + html + '</div>', 'text/html');
            var tagNodes = tagDoc.body.firstChild.childNodes;
            while (tagNodes.length) tagsRow.appendChild(tagNodes[0]);
        }
        var historyBox = item.querySelector('.history-box');
        if (historyBox) {
            historyBox.textContent = '';
            var label = document.createElement('span');
            label.className = 'chap-label';
            label.textContent = 'Last Read';
            var val = document.createElement('span');
            val.className = 'chap-val';
            val.style.color = '#facc15';
            val.textContent = data.lastRead || '-';
            historyBox.appendChild(label);
            historyBox.appendChild(val);
        }
    }
    
    function addItemToDOM(data, listName) {
        var listBlock = document.querySelector('.list-block[data-list="' + listName + '"]');
        if (!listBlock) {
            var container = document.querySelector('.container');
            var noResults = document.getElementById('noResults');
            listBlock = document.createElement('div');
            listBlock.className = 'list-block';
            listBlock.dataset.list = listName;
            var listHead = document.createElement('div');
            listHead.className = 'list-head';
            var nameSpan = document.createElement('span');
            nameSpan.className = 'list-name';
            nameSpan.textContent = listName;
            var metaSpan = document.createElement('span');
            metaSpan.className = 'list-meta';
            metaSpan.textContent = 'Custom ';
            var countSpan = document.createElement('span');
            countSpan.className = 'list-count';
            countSpan.textContent = '0';
            metaSpan.appendChild(countSpan);
            metaSpan.appendChild(document.createTextNode(' items'));
            listHead.appendChild(nameSpan);
            listHead.appendChild(metaSpan);
            listBlock.appendChild(listHead);
            container.insertBefore(listBlock, noResults);
        }
        var langMap = { ko: 'Manhwa', ja: 'Manga', zh: 'Manhua', en: 'Comic' };
        var warningTags = ['adult', 'mature', 'smut', 'gore', 'sexual violence', 'hentai'];
        var demographicTags = ['josei', 'josei(w)', 'seinen', 'shoujo', 'shounen', 'kodomo'];
        var tagsHtml = '';
        if (data.lang && langMap[data.lang]) {
            tagsHtml += '<span class="tag tag-bold">' + langMap[data.lang] + '</span>';
        }
        data.tags.split(',').filter(function(t) { return t.trim(); }).forEach(function(t) {
            var tLower = t.trim().toLowerCase();
            var cls = 'tag';
            if (warningTags.indexOf(tLower) !== -1) cls = 'tag tag-warning';
            else if (demographicTags.indexOf(tLower) !== -1) cls = 'tag tag-primary';
            tagsHtml += '<span class="' + cls + '">' + t.trim() + '</span>';
        });
        var statusClass = data.status === 'completed' ? 'status-completed' : 'status-ongoing';
        var historyHtml = data.lastRead ? '<span class="chap-val" style="color:#facc15">' + data.lastRead + '</span>' : '<span class="chap-val">-</span>';
        var coverHtml = data.cover ? '<div class="cover-box"><img class="cover-img" src="' + data.cover + '" loading="lazy"></div>' : '';
        var itemHtml = '<div class="item" data-id="' + data.id + '" data-list="' + listName + '" data-title="' + data.title.toLowerCase() + '" data-authors="' + data.authors.toLowerCase() + '" data-status="' + data.status + '" data-lang="' + data.lang + '" data-tags="' + data.tags.toLowerCase() + '" data-score="' + (data.score || 0) + '" data-chapters="' + (data.chapters || 0) + '" data-lastread="' + (data.lastReadDate || 0) + '" data-updated="' + (data.dateUpdate || 0) + '">' + coverHtml + '<div class="details"><div class="comic-title">' + data.title + '</div><div class="stats-row"><span>★ <span class="star-val">' + (data.score || '-') + '</span></span><span class="' + statusClass + '">' + data.status + '</span></div><div class="meta-row"><span>' + data.authors + '</span></div><div class="tags-row">' + tagsHtml + '</div><div class="chapter-row"><div class="chap-info history-box"><span class="chap-label">Last Read</span>' + historyHtml + '</div><div class="chap-info"><span class="chap-label">Chapters</span><span class="chap-val">' + (data.chapters || '-') + '</span></div></div><div class="item-actions"><button class="edit-btn" data-id="' + data.id + '">Edit</button><button class="delete-btn" data-id="' + data.id + '">Delete</button></div></div></div>';
        var itemParser = new DOMParser();
        var itemDoc = itemParser.parseFromString(itemHtml, 'text/html');
        listBlock.appendChild(itemDoc.body.firstChild);
    }
    
    function updateListCounts() {
        document.querySelectorAll('.list-block').forEach(function(block) {
            var count = block.querySelectorAll('.item:not(.hidden)').length;
            var countEl = block.querySelector('.list-count');
            if (countEl) countEl.textContent = count;
            if (count === 0) block.classList.add('hidden');
            else block.classList.remove('hidden');
        });
    }
    
    function openModal(isEdit, itemData, listIdx) {
        modalTitle.textContent = isEdit ? 'Edit Entry' : 'Add New Entry';
        document.getElementById('editId').value = itemData.id || '';
        document.getElementById('editListIdx').value = listIdx !== undefined ? listIdx : '';
        document.getElementById('formTitle').value = itemData.title || '';
        document.getElementById('formScore').value = itemData.score || '';
        document.getElementById('formStatus').value = itemData.status || 'ongoing';
        document.getElementById('formLang').value = itemData.lang || 'ko';
        document.getElementById('formChapters').value = itemData.chapters || '';
        document.getElementById('formAuthors').value = itemData.authors || '';
        document.getElementById('formTags').value = itemData.tags || '';
        document.getElementById('formLastRead').value = itemData.lastRead || '';
        formList.value = listIdx !== undefined ? listIdx : 0;
        formCoverData.value = itemData.cover || '';
        coverPreviewImg.src = itemData.cover || '';
        coverPreviewImg.style.display = itemData.cover ? 'block' : 'none';
        formCover.value = '';
        modal.classList.add('open');
    }
    
    function closeModal() { modal.classList.remove('open'); }
    
    if (addBtn) { addBtn.addEventListener('click', function() { openModal(false, {}, 0); }); }
    
    document.addEventListener('click', function(e) {
        if (e.target.classList.contains('edit-btn')) {
            var found = findItemInData(e.target.dataset.id);
            if (found) openModal(true, found.item, found.listIdx);
        }
        if (e.target.classList.contains('delete-btn')) {
            if (confirm('Delete this entry?')) {
                var found = findItemInData(e.target.dataset.id);
                if (found) {
                    DATA[found.listIdx].items.splice(found.itemIdx, 1);
                    var item = document.querySelector('.item[data-id="' + e.target.dataset.id + '"]');
                    if (item) item.remove();
                    updateListCounts();
                    markChanged();
                }
            }
        }
    });
    
    modalCancel.addEventListener('click', closeModal);
    modal.addEventListener('click', function(e) { if (e.target === modal) closeModal(); });
    
    modalSave.addEventListener('click', function() {
        var title = document.getElementById('formTitle').value.trim();
        if (!title) { alert('Title is required'); return; }
        
        var editId = document.getElementById('editId').value;
        var isEdit = !!editId;
        var newListIdx = parseInt(formList.value);
        
        var itemData = {
            id: editId || 'custom_' + Date.now(),
            title: title,
            score: parseFloat(document.getElementById('formScore').value) || 0,
            status: document.getElementById('formStatus').value,
            lang: document.getElementById('formLang').value,
            chapters: parseInt(document.getElementById('formChapters').value) || 0,
            authors: document.getElementById('formAuthors').value,
            tags: document.getElementById('formTags').value,
            lastRead: document.getElementById('formLastRead').value,
            lastReadDate: Date.now(),
            dateUpdate: Date.now(),
            cover: formCoverData.value
        };
        
        if (isEdit) {
            var found = findItemInData(editId);
            if (found) {
                var oldListIdx = found.listIdx;
                Object.assign(DATA[oldListIdx].items[found.itemIdx], itemData);
                var domItem = document.querySelector('.item[data-id="' + editId + '"]');
                if (domItem) {
                    if (oldListIdx !== newListIdx) {
                        DATA[oldListIdx].items.splice(found.itemIdx, 1);
                        DATA[newListIdx].items.push(itemData);
                        domItem.remove();
                        addItemToDOM(itemData, DATA[newListIdx].name);
                    } else {
                        updateItemDisplay(domItem, itemData);
                    }
                }
            }
        } else {
            DATA[newListIdx].items.push(itemData);
            addItemToDOM(itemData, DATA[newListIdx].name);
        }
        
        updateListCounts();
        markChanged();
        closeModal();
    });
    
    if (saveBtn) {
        saveBtn.addEventListener('click', function() {
            document.getElementById('listData').textContent = JSON.stringify(DATA);
            unsavedBadge.classList.remove('show');
            hasChanges = false;
            var html = '<!DOCTYPE html>\\n' + document.documentElement.outerHTML;
            var blob = new Blob([html], { type: 'text/html' });
            var a = document.createElement('a');
            a.href = URL.createObjectURL(blob);
            a.download = 'manga_list_editable.html';
            a.click();
        });
    }
    
    if (exportJsonBtn) {
        exportJsonBtn.addEventListener('click', function() {
            var blob = new Blob([JSON.stringify(DATA, null, 2)], { type: 'application/json' });
            var a = document.createElement('a');
            a.href = URL.createObjectURL(blob);
            a.download = 'manga_list_export.json';
            a.click();
        });
    }
    
    window.addEventListener('beforeunload', function(e) {
        if (hasChanges) { e.preventDefault(); e.returnValue = ''; }
    });
})();
</script>`;
    }


    // --- HISTORY HTML GENERATOR ---
    function generateHistoryHTML(items, username, date) {
        const allStatuses = new Set();
        const allLangs = new Set();
        const allTags = new Set();
        
        items.forEach(item => {
            const status = item.uploadStatus || item.originalStatus;
            if (status) allStatuses.add(status);
            if (item.origLang) allLangs.add(item.origLang);
            (item.genres || []).forEach(g => allTags.add(g.toLowerCase()));
        });

        const langOptions = [...allLangs].sort().map(l => {
            const label = formatLang(l).replace(/[^\w\s]/g, '').trim();
            return '<option value="' + l + '">' + label + '</option>';
        }).join('');
        
        const tagOptions = [...allTags].sort().map(t => {
            return '<option value="' + t + '">' + t + '</option>';
        }).join('');

        let html = `<!DOCTYPE html>
<html lang="en" data-theme="mdark">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>${username}'s ${SITE_CONFIG.name} Reading History</title>
<style>
    :root { --b1: #161616; --b2: #1c1c1c; --b3: #252525; --bc: #eee; --pc: #fff; --p: #00bcd4; --er: #f87171; --su: #4ade80; --wa: #facc15; }
    * { box-sizing: border-box; }
    body { background-color: var(--b1); color: var(--bc); font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Oxygen, Ubuntu, sans-serif; margin: 0; padding: 20px; font-size: 14px; line-height: 1.5; }
    a { text-decoration: none; color: inherit; }
    .container { max-width: 1000px; margin: 0 auto; }
    .header-main { border-bottom: 1px solid #333; padding-bottom: 15px; margin-bottom: 20px; }
    h1 { color: var(--p); margin: 0; font-size: 24px; }
    
    .filter-bar { background: var(--b2); border: 1px solid #333; border-radius: 8px; padding: 16px; margin-bottom: 24px; display: flex; flex-wrap: wrap; gap: 12px; align-items: center; }
    .search-box { flex: 1; min-width: 200px; position: relative; }
    .search-box input { width: 100%; padding: 10px 14px 10px 38px; border: 1px solid #444; border-radius: 6px; background: var(--b3); color: #fff; font-size: 14px; box-sizing: border-box; }
    .search-box input:focus { outline: none; border-color: var(--p); }
    .search-box input::placeholder { color: #666; }
    .search-icon { position: absolute; left: 12px; top: 50%; transform: translateY(-50%); width: 16px; height: 16px; color: #666; }
    .filter-select { padding: 10px 12px; border: 1px solid #444; border-radius: 6px; background: var(--b3); color: #fff; font-size: 14px; font-family: inherit; cursor: pointer; min-width: 120px; }
    .filter-select:focus { outline: none; border-color: var(--p); }
    .filter-select option { background: var(--b2); }
    .clear-btn { padding: 10px 16px; border: 1px solid #555; border-radius: 6px; background: transparent; color: #aaa; font-size: 14px; font-family: inherit; cursor: pointer; transition: all 0.2s; }
    .clear-btn:hover { border-color: var(--er); color: var(--er); }
    .results-count { font-size: 14px; color: #888; padding: 8px 0; }
    .items-container { background: var(--b2); border: 1px solid #333; border-radius: 8px; overflow: hidden; }
    .item { display: flex; padding: 12px; border-bottom: 1px solid #2a2a2a; transition: background 0.2s; }
    .item.hidden { display: none; }
    .item:last-child { border-bottom: none; }
    .item:hover { background: var(--b3); }
    .cover-box { width: 90px; flex-shrink: 0; margin-right: 15px; position: relative; border-radius: 4px; overflow: hidden; background: #000; aspect-ratio: 2/3; }
    .cover-img { width: 100%; height: 100%; object-fit: cover; }
    .details { flex-grow: 1; display: flex; flex-direction: column; justify-content: flex-start; overflow: hidden; }
    .comic-title { font-weight: bold; font-size: 16px; color: #fff; margin-bottom: 4px; display: block; }
    .stats-row { display: flex; align-items: center; gap: 12px; font-size: 14px; color: #aaa; margin-bottom: 6px; }
    .star-val { color: var(--wa); font-weight: bold; }
    .status-ongoing { color: var(--su); }
    .status-completed { color: #60a5fa; }
    .meta-row { font-size: 13px; line-height: 1.5; margin-bottom: 6px; color: #999; }
    .tags-row { display: flex; flex-wrap: wrap; font-size: 12px; color: rgba(255,255,255,0.7); margin-bottom: 8px; line-height: 1.6; }
    .tag { margin-right: 2px; }
    .tag::after { content: ','; }
    .tag:last-child::after { content: ''; }
    .tag-bold { font-weight: bold; }
    .tag-warning { font-weight: bold; border-bottom: 1px solid var(--wa); }
    .tag-primary { font-weight: bold; border-bottom: 1px solid var(--p); }
    .chapter-row { margin-top: auto; font-size: 14px; border-top: 1px dashed #333; padding-top: 8px; display: grid; grid-template-columns: 1fr 1fr; gap: 12px; }
    .chap-info { display: flex; flex-direction: column; gap: 2px; }
    .chap-label { font-size: 11px; color: #666; text-transform: uppercase; font-weight: 600; letter-spacing: 0.5px; }
    .chap-val { color: var(--p); font-weight: 600; }
    .chap-date { color: #666; font-size: 12px; }
    .history-box { color: var(--wa); }
    .no-results { text-align: center; padding: 40px; color: #666; font-size: 16px; }
    
    @media (max-width: 600px) {
        .filter-bar { flex-direction: column; }
        .search-box { width: 100%; }
        .filter-select { width: 100%; }
    }
</style>
</head>
<body>
<div class="container">
    <div class="header-main">
        <h1>${username}'s Reading History</h1>
        <div style="color:#777; font-size:12px">Exported: ${date} | ${items.length} entries</div>
    </div>
    
    <div class="filter-bar">
        <div class="search-box">
            <svg class="search-icon" viewBox="0 0 512 512" fill="currentColor"><path d="M416 208c0 45.9-14.9 88.3-40 122.7L502.6 457.4c12.5 12.5 12.5 32.8 0 45.3s-32.8 12.5-45.3 0L330.7 376c-34.4 25.2-76.8 40-122.7 40C93.1 416 0 322.9 0 208S93.1 0 208 0S416 93.1 416 208zM208 352a144 144 0 1 0 0-288 144 144 0 1 0 0 288z"/></svg>
            <input type="text" id="searchInput" placeholder="Search by title or author...">
        </div>
        <select class="filter-select" id="statusFilter">
            <option value="">All Status</option>
            ${[...allStatuses].sort().map(s => '<option value="' + s + '">' + s + '</option>').join('')}
        </select>
        <select class="filter-select" id="langFilter">
            <option value="">All Types</option>
            ${langOptions}
        </select>
        <select class="filter-select" id="tagFilter">
            <option value="">All Tags</option>
            ${tagOptions}
        </select>
        <select class="filter-select" id="sortBy">
            <option value="">Recently Read</option>
            <option value="title">Title A-Z</option>
            <option value="title-desc">Title Z-A</option>
            <option value="score">Highest Score</option>
            <option value="chapters">Most Chapters</option>
        </select>
        <button class="clear-btn" id="clearFilters">Clear</button>
    </div>
    <div class="results-count" id="resultsCount"></div>
    <div class="items-container">`;

        items.forEach(item => {
            const score = item.score_avg ? item.score_avg.toFixed(1) : '-';
            const status = item.uploadStatus || item.originalStatus || 'Unknown';
            const statusClass = status.toLowerCase() === 'completed' ? 'status-completed' : 'status-ongoing';
            const latestChap = item.chaps_normal ? `Ch.${item.chaps_normal}` : 'Unknown';
            const authors = (item.authors || []).join(', ');
            const escapedTitle = (item.name || '').replace(/"/g, '&quot;').toLowerCase();
            const escapedAuthors = authors.replace(/"/g, '&quot;').toLowerCase();
            
            const warningTags = ['adult', 'mature', 'smut', 'gore', 'sexual violence', 'hentai'];
            const demographicTags = ['josei', 'josei(w)', 'seinen', 'shoujo', 'shounen', 'kodomo'];
            const langName = { 'ko': 'Manhwa', 'ja': 'Manga', 'zh': 'Manhua', 'en': 'Comic' };
            
            let tagsHtml = '';
            if (item.origLang && langName[item.origLang]) {
                tagsHtml += `<span class="tag tag-bold">${langName[item.origLang]}</span>`;
            }
            (item.genres || []).forEach(g => {
                const gLower = g.toLowerCase();
                let tagClass = 'tag';
                if (warningTags.includes(gLower)) tagClass = 'tag tag-warning';
                else if (demographicTags.includes(gLower)) tagClass = 'tag tag-primary';
                tagsHtml += `<span class="${tagClass}">${g}</span>`;
            });

            const imgSrc = item.base64Cover;
            const coverHtml = imgSrc ? `<div class="cover-box"><img class="cover-img" src="${imgSrc}" loading="lazy"></div>` : '';

            const tagsLower = (item.genres || []).map(g => g.toLowerCase()).join(',');
            const scoreNum = item.score_avg || 0;
            const chaptersNum = item.chaps_normal || 0;

            html += `<div class="item" data-title="${escapedTitle}" data-authors="${escapedAuthors}" data-status="${status}" data-lang="${item.origLang || ''}" data-tags="${tagsLower}" data-score="${scoreNum}" data-chapters="${chaptersNum}" data-lastread="${item.readDate || 0}">${coverHtml}<div class="details"><div class="comic-title">${item.name}</div><div class="stats-row"><span>&#9733; <span class="star-val">${score}</span></span><span class="${statusClass}">${status}</span></div><div class="meta-row"><span>${authors}</span></div><div class="tags-row">${tagsHtml}</div><div class="chapter-row"><div class="chap-info history-box"><span class="chap-label">Last Read</span><span class="chap-val" style="color:#facc15">${item.lastReadChapter || '-'}</span><span class="chap-date">${timeAgo(item.readDate)}</span></div><div class="chap-info"><span class="chap-label">Latest Chapter</span><span class="chap-val">${latestChap}</span></div></div></div></div>`;
        });
        
        html += `</div>
    <div class="no-results" id="noResults" style="display:none;">No comics match your filters</div>
</div>
<script>
(function() {
    var searchInput = document.getElementById('searchInput');
    var statusFilter = document.getElementById('statusFilter');
    var langFilter = document.getElementById('langFilter');
    var tagFilter = document.getElementById('tagFilter');
    var sortBy = document.getElementById('sortBy');
    var clearBtn = document.getElementById('clearFilters');
    var resultsCount = document.getElementById('resultsCount');
    var noResults = document.getElementById('noResults');
    
    function applyFilters() {
        var search = searchInput.value.toLowerCase().trim();
        var statusVal = statusFilter.value;
        var langVal = langFilter.value;
        var tagVal = tagFilter.value.toLowerCase();
        var sortVal = sortBy.value;
        
        var items = Array.from(document.querySelectorAll('.item'));
        var totalVisible = 0;
        
        if (sortVal) {
            items.sort(function(a, b) {
                switch(sortVal) {
                    case 'title': return a.dataset.title.localeCompare(b.dataset.title);
                    case 'title-desc': return b.dataset.title.localeCompare(a.dataset.title);
                    case 'score': return parseFloat(b.dataset.score || 0) - parseFloat(a.dataset.score || 0);
                    case 'chapters': return parseInt(b.dataset.chapters || 0) - parseInt(a.dataset.chapters || 0);
                    default: return parseInt(b.dataset.lastread || 0) - parseInt(a.dataset.lastread || 0);
                }
            });
            var parent = items[0] ? items[0].parentNode : null;
            if (parent) { items.forEach(function(item) { parent.appendChild(item); }); }
        }
        
        items.forEach(function(item) {
            var title = item.dataset.title;
            var authors = item.dataset.authors;
            var status = item.dataset.status;
            var lang = item.dataset.lang;
            var tags = item.dataset.tags || '';
            
            var searchMatch = !search || title.indexOf(search) !== -1 || authors.indexOf(search) !== -1;
            var statusMatch = !statusVal || status === statusVal;
            var langMatch = !langVal || lang === langVal;
            var tagMatch = !tagVal || tags.indexOf(tagVal) !== -1;
            
            if (searchMatch && statusMatch && langMatch && tagMatch) {
                item.classList.remove('hidden');
                totalVisible++;
            } else {
                item.classList.add('hidden');
            }
        });
        
        var hasFilters = search || statusVal || langVal || tagVal;
        resultsCount.textContent = hasFilters ? totalVisible + ' comic' + (totalVisible !== 1 ? 's' : '') + ' found' : '';
        noResults.style.display = totalVisible === 0 ? 'block' : 'none';
    }
    
    searchInput.addEventListener('input', applyFilters);
    statusFilter.addEventListener('change', applyFilters);
    langFilter.addEventListener('change', applyFilters);
    tagFilter.addEventListener('change', applyFilters);
    sortBy.addEventListener('change', applyFilters);
    
    clearBtn.addEventListener('click', function() {
        searchInput.value = '';
        statusFilter.value = '';
        langFilter.value = '';
        tagFilter.value = '';
        sortBy.value = '';
        applyFilters();
    });
})();
</script>
</body></html>`;
        return html;
    }


    // --- FORMAT 2: PDF Generation using jsPDF ---
    async function generatePDF(lists, username, date, includeCovers = false) {
        const jsPDF = globalThis.jspdf?.jsPDF;
        if (!jsPDF) {
            throw new Error('jsPDF library not loaded. Please reload the extension.');
        }
        
        const doc = new jsPDF({ orientation: 'portrait', unit: 'mm', format: 'a4' });
        const fontName = globalThis.BATO_PDF_FONT || 'helvetica';
        doc.setFont(fontName, 'normal');
        
        const pageWidth = doc.internal.pageSize.getWidth();
        const pageHeight = doc.internal.pageSize.getHeight();
        const margin = 15;
        const contentWidth = pageWidth - (margin * 2);
        let y = margin;
        
        const C = {
            dark: [51, 51, 51],
            text: [68, 68, 68],
            muted: [136, 136, 136],
            border: [230, 230, 230],
            headerBg: [245, 247, 250]
        };
        
        const allComics = [];
        let completedCount = 0;
        let ongoingCount = 0;
        
        lists.forEach(list => {
            const items = list.data.comicNodes || [];
            items.forEach(node => {
                allComics.push({ ...node.data, listName: list.data.name });
                const status = (node.data.uploadStatus || node.data.originalStatus || '').toLowerCase();
                if (status === 'completed') completedCount++;
                else ongoingCount++;
            });
        });
        
        const totalCount = allComics.length;
        
        function checkPage(needed = 15) {
            if (y + needed > pageHeight - 15) {
                doc.addPage();
                y = margin;
                return true;
            }
            return false;
        }
        
        // Header
        doc.setFontSize(24);
        doc.setFont(fontName, 'bold');
        doc.setTextColor(...C.dark);
        doc.text(`${SITE_CONFIG.name} List Export`, pageWidth / 2, y + 5, { align: 'center' });
        
        y += 14;
        doc.setFontSize(11);
        doc.setFont(fontName, 'normal');
        doc.setTextColor(...C.text);
        doc.text(username, pageWidth / 2, y, { align: 'center' });
        
        y += 6;
        doc.setFontSize(10);
        doc.setTextColor(...C.muted);
        doc.text(`Exported: ${date}`, pageWidth / 2, y, { align: 'center' });
        
        y += 8;
        doc.setDrawColor(...C.border);
        doc.setLineWidth(0.5);
        doc.line(margin + 20, y, pageWidth - margin - 20, y);
        
        y += 10;
        doc.setFontSize(10);
        doc.setTextColor(...C.text);
        const statsText = `Total Series: ${totalCount}  |  Completed: ${completedCount}  |  Ongoing: ${ongoingCount}`;
        doc.text(statsText, pageWidth / 2, y, { align: 'center' });
        
        y += 12;
        
        const baseRowHeight = includeCovers ? 18 : 12;
        const coverWidth = includeCovers ? 12 : 0;
        const coverColWidth = includeCovers ? 15 : 0;
        const lineHeight = 4;
        
        const col = {
            num: margin,
            cover: margin + 10,
            title: margin + 10 + coverColWidth,
            progress: margin + (includeCovers ? 115 : 125),
            tags: margin + (includeCovers ? 140 : 150)
        };
        
        const titleMaxWidth = col.progress - col.title - 5;
        const tagsMaxWidth = pageWidth - margin - col.tags - 2;
        
        doc.setFillColor(...C.headerBg);
        doc.rect(margin, y - 4, contentWidth, 10, 'F');
        
        doc.setFontSize(8);
        doc.setFont(fontName, 'bold');
        doc.setTextColor(...C.muted);
        
        doc.text('#', col.num + 2, y + 2);
        doc.text('Title', col.title, y + 2);
        doc.text('Progress', col.progress, y + 2);
        doc.text('Tags', col.tags, y + 2);
        
        y += 10;
        
        doc.setDrawColor(...C.border);
        doc.setLineWidth(0.3);
        doc.line(margin, y - 2, pageWidth - margin, y - 2);
        
        for (let i = 0; i < allComics.length; i++) {
            const c = allComics[i];
            
            doc.setFontSize(10);
            doc.setFont(fontName, 'bold');
            const titleLines = doc.splitTextToSize(c.name || '', titleMaxWidth);
            
            doc.setFontSize(8);
            doc.setFont(fontName, 'normal');
            const genres = (c.genres || []).join(', ');
            const tagLines = doc.splitTextToSize(genres, tagsMaxWidth);
            
            const titleHeight = titleLines.length * lineHeight;
            const tagHeight = tagLines.length * lineHeight;
            const textHeight = Math.max(titleHeight, tagHeight, baseRowHeight);
            const rowHeight = Math.max(textHeight + 4, baseRowHeight);
            
            checkPage(rowHeight + 2);
            
            const rowY = y;
            const textStartY = rowY + 3;
            
            doc.setFontSize(9);
            doc.setFont(fontName, 'normal');
            doc.setTextColor(...C.muted);
            doc.text(String(i + 1), col.num + 2, textStartY);
            
            if (includeCovers) {
                doc.setFillColor(240, 240, 240);
                doc.setDrawColor(...C.border);
                doc.rect(col.cover, rowY - 1, coverWidth, Math.min(rowHeight - 2, 16), 'FD');
                
                if (c.base64Cover) {
                    try {
                        doc.addImage(c.base64Cover, 'JPEG', col.cover, rowY - 1, coverWidth, Math.min(rowHeight - 2, 16));
                    } catch (e) {}
                }
            }
            
            doc.setFontSize(10);
            doc.setFont(fontName, 'bold');
            doc.setTextColor(...C.dark);
            titleLines.forEach((line, idx) => {
                doc.text(line, col.title, textStartY + (idx * lineHeight));
            });
            
            doc.setFontSize(9);
            doc.setFont(fontName, 'normal');
            doc.setTextColor(...C.text);
            let progressText = '-';
            if (c.history && c.chaps_normal) {
                const readMatch = c.history.chapterName.match(/\d+/);
                const readNum = readMatch ? readMatch[0] : '?';
                progressText = `${readNum} / ${c.chaps_normal}`;
            } else if (c.chaps_normal) {
                progressText = `- / ${c.chaps_normal}`;
            } else if (c.history) {
                const readMatch = c.history.chapterName.match(/\d+/);
                const readNum = readMatch ? readMatch[0] : '?';
                progressText = `${readNum} / ???`;
            }
            doc.text(progressText, col.progress, textStartY);
            
            doc.setFontSize(8);
            doc.setTextColor(...C.muted);
            tagLines.forEach((line, idx) => {
                doc.text(line, col.tags, textStartY + (idx * lineHeight));
            });
            
            y += rowHeight;
            
            doc.setDrawColor(...C.border);
            doc.setLineWidth(0.1);
            doc.line(margin, y - 2, pageWidth - margin, y - 2);
        }
        
        const totalPages = doc.internal.getNumberOfPages();
        for (let i = 1; i <= totalPages; i++) {
            doc.setPage(i);
            doc.setFontSize(8);
            doc.setTextColor(...C.muted);
            doc.text(`Page ${i} of ${totalPages}`, pageWidth / 2, pageHeight - 8, { align: 'center' });
        }
        
        return doc.output('blob');
    }

    // --- HISTORY PDF GENERATOR ---
    async function generateHistoryPDF(items, username, date, includeCovers = false) {
        const jsPDF = globalThis.jspdf?.jsPDF;
        if (!jsPDF) {
            throw new Error('jsPDF library not loaded. Please reload the extension.');
        }
        
        const doc = new jsPDF({ orientation: 'portrait', unit: 'mm', format: 'a4' });
        const fontName = globalThis.BATO_PDF_FONT || 'helvetica';
        doc.setFont(fontName, 'normal');
        
        const pageWidth = doc.internal.pageSize.getWidth();
        const pageHeight = doc.internal.pageSize.getHeight();
        const margin = 15;
        const contentWidth = pageWidth - (margin * 2);
        let y = margin;
        
        const C = {
            dark: [51, 51, 51],
            text: [68, 68, 68],
            muted: [136, 136, 136],
            border: [230, 230, 230],
            headerBg: [245, 247, 250]
        };
        
        function checkPage(needed = 15) {
            if (y + needed > pageHeight - 15) {
                doc.addPage();
                y = margin;
                return true;
            }
            return false;
        }
        
        // Header
        doc.setFontSize(24);
        doc.setFont(fontName, 'bold');
        doc.setTextColor(...C.dark);
        doc.text(`${SITE_CONFIG.name} Reading History`, pageWidth / 2, y + 5, { align: 'center' });
        
        y += 14;
        doc.setFontSize(11);
        doc.setFont(fontName, 'normal');
        doc.setTextColor(...C.text);
        doc.text(username, pageWidth / 2, y, { align: 'center' });
        
        y += 6;
        doc.setFontSize(10);
        doc.setTextColor(...C.muted);
        doc.text(`Exported: ${date} | ${items.length} entries`, pageWidth / 2, y, { align: 'center' });
        
        y += 8;
        doc.setDrawColor(...C.border);
        doc.setLineWidth(0.5);
        doc.line(margin + 20, y, pageWidth - margin - 20, y);
        
        y += 12;
        
        const baseRowHeight = includeCovers ? 18 : 12;
        const coverWidth = includeCovers ? 12 : 0;
        const coverColWidth = includeCovers ? 15 : 0;
        const lineHeight = 4;
        
        const col = {
            num: margin,
            cover: margin + 10,
            title: margin + 10 + coverColWidth,
            lastRead: margin + (includeCovers ? 100 : 110),
            tags: margin + (includeCovers ? 130 : 140)
        };
        
        const titleMaxWidth = col.lastRead - col.title - 5;
        const tagsMaxWidth = pageWidth - margin - col.tags - 2;
        
        doc.setFillColor(...C.headerBg);
        doc.rect(margin, y - 4, contentWidth, 10, 'F');
        
        doc.setFontSize(8);
        doc.setFont(fontName, 'bold');
        doc.setTextColor(...C.muted);
        
        doc.text('#', col.num + 2, y + 2);
        doc.text('Title', col.title, y + 2);
        doc.text('Last Read', col.lastRead, y + 2);
        doc.text('Tags', col.tags, y + 2);
        
        y += 10;
        
        doc.setDrawColor(...C.border);
        doc.setLineWidth(0.3);
        doc.line(margin, y - 2, pageWidth - margin, y - 2);
        
        for (let i = 0; i < items.length; i++) {
            const item = items[i];
            
            doc.setFontSize(10);
            doc.setFont(fontName, 'bold');
            const titleLines = doc.splitTextToSize(item.name || '', titleMaxWidth);
            
            doc.setFontSize(8);
            doc.setFont(fontName, 'normal');
            const genres = (item.genres || []).join(', ');
            const tagLines = doc.splitTextToSize(genres, tagsMaxWidth);
            
            const titleHeight = titleLines.length * lineHeight;
            const tagHeight = tagLines.length * lineHeight;
            const textHeight = Math.max(titleHeight, tagHeight, baseRowHeight);
            const rowHeight = Math.max(textHeight + 4, baseRowHeight);
            
            checkPage(rowHeight + 2);
            
            const rowY = y;
            const textStartY = rowY + 3;
            
            doc.setFontSize(9);
            doc.setFont(fontName, 'normal');
            doc.setTextColor(...C.muted);
            doc.text(String(i + 1), col.num + 2, textStartY);
            
            if (includeCovers) {
                doc.setFillColor(240, 240, 240);
                doc.setDrawColor(...C.border);
                doc.rect(col.cover, rowY - 1, coverWidth, Math.min(rowHeight - 2, 16), 'FD');
                
                if (item.base64Cover) {
                    try {
                        doc.addImage(item.base64Cover, 'JPEG', col.cover, rowY - 1, coverWidth, Math.min(rowHeight - 2, 16));
                    } catch (e) {}
                }
            }
            
            doc.setFontSize(10);
            doc.setFont(fontName, 'bold');
            doc.setTextColor(...C.dark);
            titleLines.forEach((line, idx) => {
                doc.text(line, col.title, textStartY + (idx * lineHeight));
            });
            
            doc.setFontSize(9);
            doc.setFont(fontName, 'normal');
            doc.setTextColor(...C.text);
            const lastReadText = item.lastReadChapter || '-';
            const readDateText = item.readDate ? timeAgo(item.readDate) : '';
            doc.text(lastReadText, col.lastRead, textStartY);
            if (readDateText) {
                doc.setFontSize(7);
                doc.setTextColor(...C.muted);
                doc.text(readDateText, col.lastRead, textStartY + lineHeight);
            }
            
            doc.setFontSize(8);
            doc.setTextColor(...C.muted);
            tagLines.forEach((line, idx) => {
                doc.text(line, col.tags, textStartY + (idx * lineHeight));
            });
            
            y += rowHeight;
            
            doc.setDrawColor(...C.border);
            doc.setLineWidth(0.1);
            doc.line(margin, y - 2, pageWidth - margin, y - 2);
        }
        
        const totalPages = doc.internal.getNumberOfPages();
        for (let i = 1; i <= totalPages; i++) {
            doc.setPage(i);
            doc.setFontSize(8);
            doc.setTextColor(...C.muted);
            doc.text(`Page ${i} of ${totalPages}`, pageWidth / 2, pageHeight - 8, { align: 'center' });
        }
        
        return doc.output('blob');
    }


    // --- CSV Generator ---
    function generateCSV(lists, date) {
        const rows = [];
        rows.push(['List Name', 'Privacy', 'Comic Name', 'Lang', 'Score', 'Status', 'Last Read Ch', 'Last Read Date', `Latest Ch (As of ${date})`, 'Updated', 'Genres', 'URL']);
        
        lists.forEach(list => {
            if (!list.data.comicNodes) return;
            list.data.comicNodes.forEach(node => {
                const c = node.data;
                const safe = (s) => `"${(s||'').replace(/"/g, '""')}"`;
                const lastReadCh = c.history ? c.history.chapterName : '-';
                const lastReadDate = c.history ? new Date(c.history.readDate).toISOString().slice(0,10) : '-';

                rows.push([
                    safe(list.data.name),
                    list.data.isPublic ? 'Public' : 'Private',
                    safe(c.name),
                    formatLang(c.origLang),
                    c.score_avg,
                    c.uploadStatus || c.originalStatus,
                    safe(lastReadCh),
                    lastReadDate,
                    c.chaps_normal,
                    timeAgo(c.dateUpdate),
                    safe((c.genres||[]).join(', ')),
                    SITE_CONFIG.baseUrl + c.urlPath
                ]);
            });
        });
        return rows.map(r => r.join(',')).join('\n');
    }

    // --- History CSV Generator ---
    function generateHistoryCSV(items, date) {
        const rows = [];
        rows.push(['Comic Name', 'Type', 'Score', 'Status', 'Last Read Chapter', 'Read Date', 'Total Chapters', 'Genres', 'Authors', 'URL']);
        
        items.forEach(item => {
            const safe = (s) => `"${(s||'').replace(/"/g, '""')}"`;
            const readDate = item.readDate ? new Date(item.readDate).toISOString().slice(0,10) : '-';

            rows.push([
                safe(item.name),
                formatLang(item.origLang),
                item.score_avg || '-',
                item.uploadStatus || item.originalStatus || '-',
                safe(item.lastReadChapter || '-'),
                readDate,
                item.chaps_normal || '-',
                safe((item.genres||[]).join(', ')),
                safe((item.authors||[]).join(', ')),
                SITE_CONFIG.baseUrl + item.urlPath
            ]);
        });
        return rows.map(r => r.join(',')).join('\n');
    }

    // --- JSON Generator ---
    function generateJSON(lists) {
        return JSON.stringify(lists.map(l => ({
            id: l.id,
            data: {
                name: l.data.name,
                isPublic: l.data.isPublic,
                comicNodes: (l.data.comicNodes || []).map(n => ({
                    data: {
                        id: n.data.id,
                        name: n.data.name,
                        urlPath: n.data.urlPath,
                        urlCover600: n.data.urlCover600,
                        genres: n.data.genres,
                        authors: n.data.authors,
                        score_avg: n.data.score_avg,
                        status: n.data.uploadStatus || n.data.originalStatus,
                        origLang: n.data.origLang,
                        dateUpdate: n.data.dateUpdate,
                        chaps_normal: n.data.chaps_normal,
                        history: n.data.history || null
                    }
                }))
            }
        })), null, 2);
    }

    // --- History JSON Generator ---
    function generateHistoryJSON(items) {
        return JSON.stringify(items.map(item => ({
            id: item.id,
            name: item.name,
            urlPath: item.urlPath,
            urlCover600: item.urlCover600,
            genres: item.genres,
            authors: item.authors,
            score_avg: item.score_avg,
            status: item.uploadStatus || item.originalStatus,
            origLang: item.origLang,
            chaps_normal: item.chaps_normal,
            lastReadChapter: item.lastReadChapter,
            readDate: item.readDate
        })), null, 2);
    }

    // --- UI MANAGER ---
    const UI = {
        modal: null,
        
        create(isHistory = false) {
            if (document.getElementById('be-modal')) return;
            const title = isHistory ? 'Export Reading History' : 'Export Options';
            const html = `
            <div class="be-modal-overlay" id="be-overlay">
                <div class="be-modal" id="be-modal">
                    <div class="be-header">
                        <h3 class="be-title">${title}</h3>
                        <button class="be-close" id="be-close">&times;</button>
                    </div>
                    <div class="be-option-group">
                        
                        <!-- Visual List with Sub-option -->
                        <div class="be-option" style="flex-direction: column; align-items: flex-start; cursor: default;">
                            <label style="display: flex; align-items: flex-start; width: 100%; cursor: pointer;">
                                <input type="checkbox" id="fmt-html" checked>
                                <div>
                                    <span class="be-label">Visual List (HTML)</span>
                                    <span class="be-desc">Self-contained viewer. Best for reading on PC/Mobile.</span>
                                </div>
                            </label>
                            
                            <div id="sub-opt-container" style="width: 100%; margin-top: 10px; padding-top: 10px; border-top: 1px solid #333; margin-left: 4px;">
                                <label style="display: flex; align-items: center; cursor: pointer; margin-bottom: 8px;">
                                    <input type="checkbox" id="opt-covers" style="width: 14px; height: 14px; margin-top: 0; margin-right: 10px;">
                                    <span style="font-size: 13px; font-weight: bold; color: #ccc;">Embed Cover Images</span>
                                </label>
                                <div style="font-size: 11px; color: #888; margin-left: 24px; margin-top: 2px; margin-bottom: 10px;">
                                    Makes file fully offline but significantly larger (slower export).
                                </div>
                                ${!isHistory ? `<label style="display: flex; align-items: center; cursor: pointer;">
                                    <input type="checkbox" id="opt-editable" style="width: 14px; height: 14px; margin-top: 0; margin-right: 10px;">
                                    <span style="font-size: 13px; font-weight: bold; color: #ccc;">Editable Version</span>
                                </label>
                                <div style="font-size: 11px; color: #888; margin-left: 24px; margin-top: 2px;">
                                    Add/edit/delete entries. Changes saved within the HTML file.
                                </div>` : ''}
                            </div>
                        </div>

                        <label class="be-option">
                            <input type="checkbox" id="fmt-pdf">
                            <div>
                                <span class="be-label">PDF Document</span>
                                <span class="be-desc">Professional formatted PDF with table layout.</span>
                            </div>
                        </label>
                        
                        <div id="pdf-opt-container" style="width: 100%; margin-top: -8px; margin-bottom: 4px; padding: 10px 16px; background: #252525; border-radius: 0 0 8px 8px; display: none;">
                            <label style="display: flex; align-items: center; cursor: pointer;">
                                <input type="checkbox" id="opt-pdf-covers" style="width: 14px; height: 14px; margin-top: 0; margin-right: 10px;">
                                <span style="font-size: 13px; font-weight: bold; color: #ccc;">Include Cover Images</span>
                            </label>
                            <div style="font-size: 11px; color: #888; margin-left: 24px; margin-top: 2px;">
                                Adds thumbnail covers (requires downloading images first).
                            </div>
                        </div>
                        <label class="be-option">
                            <input type="checkbox" id="fmt-csv">
                            <div><span class="be-label">Spreadsheet (CSV)</span></div>
                        </label>
                        <label class="be-option">
                            <input type="checkbox" id="fmt-json">
                            <div><span class="be-label">JSON Data</span></div>
                        </label>
                    </div>
                    <div class="be-progress-area" id="be-progress">Ready</div>
                    <div class="be-footer">
                        <button class="be-btn be-btn-secondary" id="be-cancel">Cancel</button>
                        <button class="be-btn be-btn-primary" id="be-start">Start Export</button>
                    </div>
                </div>
            </div>`;
            const parser = new DOMParser();
            const doc = parser.parseFromString(html, 'text/html');
            document.body.appendChild(doc.body.firstElementChild);

            this.modal = document.getElementById('be-overlay');
            
            const htmlCheck = document.getElementById('fmt-html');
            const coversCheck = document.getElementById('opt-covers');
            const editableCheck = document.getElementById('opt-editable');
            const subContainer = document.getElementById('sub-opt-container');

            const toggleCovers = () => {
                if (htmlCheck.checked) {
                    coversCheck.disabled = false;
                    if (editableCheck) editableCheck.disabled = false;
                    subContainer.style.opacity = '1';
                    subContainer.style.pointerEvents = 'auto';
                } else {
                    coversCheck.disabled = true;
                    if (editableCheck) editableCheck.disabled = true;
                    subContainer.style.opacity = '0.4';
                    subContainer.style.pointerEvents = 'none';
                }
            };
            htmlCheck.onchange = toggleCovers;
            toggleCovers();
            
            const pdfCheck = document.getElementById('fmt-pdf');
            const pdfCoversCheck = document.getElementById('opt-pdf-covers');
            const pdfOptContainer = document.getElementById('pdf-opt-container');
            
            const togglePdfCovers = () => {
                if (pdfCheck.checked) {
                    pdfOptContainer.style.display = 'block';
                } else {
                    pdfOptContainer.style.display = 'none';
                    pdfCoversCheck.checked = false;
                }
            };
            pdfCheck.onchange = togglePdfCovers;
            togglePdfCovers();

            document.getElementById('be-close').onclick = () => this.hide();
            document.getElementById('be-cancel').onclick = () => this.hide();
            document.getElementById('be-start').onclick = () => {
                if (isHistory) {
                    historyLogic.startExport();
                } else {
                    logic.startExport();
                }
            };
            this.modal.onclick = (e) => { if(e.target === this.modal) this.hide(); };
        },
        show(isHistory = false) { 
            this.create(isHistory); 
            this.modal.classList.add('open'); 
        },
        hide() { if(this.modal) { this.modal.classList.remove('open'); this.modal.remove(); this.modal = null; } },
        updateStatus(msg) {
            const el = document.getElementById('be-progress');
            el.style.display = 'block';
            el.innerText = msg;
        },
        setLoading(loading) {
            const btn = document.getElementById('be-start');
            btn.disabled = loading;
            btn.innerText = loading ? "Processing..." : "Start Export";
        }
    };


    // --- MAIN LOGIC FOR LISTS ---
    const logic = {
        async startExport() {
            const userId = getUserId();
            const doHtml = document.getElementById('fmt-html').checked;
            const doPdf = document.getElementById('fmt-pdf').checked;
            const doCsv = document.getElementById('fmt-csv').checked;
            const doJson = document.getElementById('fmt-json').checked;
            const embedCovers = document.getElementById('opt-covers').checked;
            const editableCheck = document.getElementById('opt-editable');
            const editableHtml = editableCheck ? editableCheck.checked : false;
            const pdfCovers = document.getElementById('opt-pdf-covers').checked;

            if (!doHtml && !doCsv && !doJson && !doPdf) return alert("Select a format.");

            UI.setLoading(true);
            UI.updateStatus(`Connecting to ${SITE_CONFIG.name}...`);

            try {
                const lists = await this.fetchLists(userId);
                if (!lists.length) {
                    UI.updateStatus("No lists found.");
                    UI.setLoading(false);
                    return;
                }

                UI.updateStatus("Fetching reading history...");
                const historyMap = await this.fetchHistory();
                
                lists.forEach(list => {
                    if (list.data.comicNodes) {
                        list.data.comicNodes.forEach(node => {
                            const comicId = node.data.id;
                            if (historyMap.has(comicId)) {
                                node.data.history = cleanObject(historyMap.get(comicId));
                            }
                        });
                    }
                });

                if ((doHtml && embedCovers) || (doPdf && pdfCovers)) {
                    await processCoversWithQueue(lists, UI.updateStatus);
                }

                UI.updateStatus("Generating files...");
                const user = document.title.split("'")[0] || "User";
                const date = new Date().toISOString().slice(0, 10);
                const base = `${SITE_CONFIG.listType}_export_${user}_${date}`;

                if (doHtml) downloadFile(generateHTML(lists, user, date, editableHtml), base+'.html', 'text/html');
                if (doPdf) {
                    UI.updateStatus("Generating PDF...");
                    const pdfBlob = await generatePDF(lists, user, date, pdfCovers);
                    downloadFile(pdfBlob, base+'.pdf', 'application/pdf');
                }
                if (doCsv) downloadFile("\uFEFF"+generateCSV(lists, date), base+'.csv', 'text/csv;charset=utf-8;');
                if (doJson) downloadFile(generateJSON(lists), base+'.json', 'application/json');

                UI.updateStatus("Done! Check your downloads.");
                UI.setLoading(false);
            } catch (e) {
                console.error(e);
                UI.updateStatus("Error: " + e.message);
                UI.setLoading(false);
            }
        },

        async fetchLists(userId) {
            const allLists = [];
            let page = 1, hasMore = true;
            while (hasMore) {
                UI.updateStatus(`Fetching lists page ${page}...`);
                const res = await fetch(API_ENDPOINT, {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify({
                        query: LIST_QUERY,
                        variables: { select: { page, size: 20, sortby: "update", userId } }
                    })
                });
                const json = await res.json();
                const data = cleanObject(json.data?.get_user_mylistList);
                if (!data?.items) break;
                allLists.push(...data.items);
                if (page >= data.paging.pages) hasMore = false; else page++;
            }
            return allLists;
        },

        async fetchHistory() {
            const historyMap = new Map(); 
            let startCursor = null;
            let page = 1;
            let hasMore = true;
            const MAX_HISTORY_PAGES = 20; 

            while (hasMore && page <= MAX_HISTORY_PAGES) {
                UI.updateStatus(`Fetching history page ${page}...`);
                const res = await fetch(API_ENDPOINT, {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify({
                        query: HISTORY_QUERY,
                        variables: { select: { limit: HISTORY_FETCH_LIMIT, start: startCursor } }
                    })
                });
                const json = await res.json();
                const data = cleanObject(json.data?.get_sser_myHistory);
                
                if (!data || !data.items || data.items.length === 0) break;

                data.items.forEach(item => {
                    const comicId = item.comicNode?.id;
                    const chapterData = item.chapterNode?.data;
                    if (comicId && chapterData) {
                        if (!historyMap.has(comicId)) {
                            historyMap.set(comicId, cleanObject({
                                chapterName: chapterData.dname || chapterData.title || `Ch.${chapterData.order}`,
                                readDate: item.date
                            }));
                        }
                    }
                });
                startCursor = data.newStart;
                if (!startCursor) hasMore = false;
                page++;
            }
            return historyMap;
        }
    };

    // --- HISTORY PAGE LOGIC ---
    const historyLogic = {
        async startExport() {
            const doHtml = document.getElementById('fmt-html').checked;
            const doPdf = document.getElementById('fmt-pdf').checked;
            const doCsv = document.getElementById('fmt-csv').checked;
            const doJson = document.getElementById('fmt-json').checked;
            const embedCovers = document.getElementById('opt-covers').checked;
            const pdfCovers = document.getElementById('opt-pdf-covers').checked;

            if (!doHtml && !doCsv && !doJson && !doPdf) return alert("Select a format.");

            UI.setLoading(true);
            UI.updateStatus(`Fetching reading history from ${SITE_CONFIG.name}...`);

            try {
                const items = await this.fetchFullHistory();
                if (!items.length) {
                    UI.updateStatus("No history found.");
                    UI.setLoading(false);
                    return;
                }

                if ((doHtml && embedCovers) || (doPdf && pdfCovers)) {
                    await processHistoryCovers(items, UI.updateStatus);
                }

                UI.updateStatus("Generating files...");
                const user = "My";
                const date = new Date().toISOString().slice(0, 10);
                const base = `${SITE_CONFIG.name.toLowerCase()}_history_${date}`;

                if (doHtml) downloadFile(generateHistoryHTML(items, user, date), base+'.html', 'text/html');
                if (doPdf) {
                    UI.updateStatus("Generating PDF...");
                    const pdfBlob = await generateHistoryPDF(items, user, date, pdfCovers);
                    downloadFile(pdfBlob, base+'.pdf', 'application/pdf');
                }
                if (doCsv) downloadFile("\uFEFF"+generateHistoryCSV(items, date), base+'.csv', 'text/csv;charset=utf-8;');
                if (doJson) downloadFile(generateHistoryJSON(items), base+'.json', 'application/json');

                UI.updateStatus("Done! Check your downloads.");
                UI.setLoading(false);
            } catch (e) {
                console.error(e);
                UI.updateStatus("Error: " + e.message);
                UI.setLoading(false);
            }
        },

        async fetchFullHistory() {
            const items = [];
            const seenComicIds = new Set();
            let startCursor = null;
            let page = 1;
            let hasMore = true;
            const MAX_PAGES = 50;

            while (hasMore && page <= MAX_PAGES) {
                UI.updateStatus(`Fetching history page ${page}...`);
                const res = await fetch(API_ENDPOINT, {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify({
                        query: HISTORY_FULL_QUERY,
                        variables: { select: { limit: HISTORY_EXPORT_LIMIT, start: startCursor } }
                    })
                });
                const json = await res.json();
                const data = cleanObject(json.data?.get_sser_myHistory);
                
                if (!data || !data.items || data.items.length === 0) break;

                data.items.forEach(item => {
                    const comic = item.comicNode?.data;
                    const chapter = item.chapterNode?.data;
                    if (comic && !seenComicIds.has(comic.id)) {
                        seenComicIds.add(comic.id);
                        
                        // Get latest chapter info - handle both Bato and MangaPark field names
                        const latestChapterNodes = comic.last_chapterNodes || comic.chapterNodes_last;
                        const latestChapter = latestChapterNodes?.[0]?.data;
                        
                        items.push({
                            id: comic.id,
                            name: comic.name,
                            urlPath: comic.urlPath,
                            urlCover600: comic.urlCover600,
                            genres: comic.genres,
                            authors: comic.authors || comic.artists || [],
                            score_avg: comic.score_avg,
                            uploadStatus: comic.uploadStatus,
                            originalStatus: comic.originalStatus,
                            origLang: comic.origLang,
                            chaps_normal: comic.chaps_normal,
                            dateUpdate: comic.dateUpdate || comic.dateModify,
                            lastReadChapter: chapter?.dname || chapter?.title || `Ch.${chapter?.serial}`,
                            readDate: item.date,
                            latestChapter: latestChapter?.dname || latestChapter?.title || null
                        });
                    }
                });
                
                startCursor = data.newStart;
                if (!startCursor) hasMore = false;
                page++;
            }
            return items;
        }
    };

    // --- INITIALIZATION ---
    function initListPage() {
        if (document.getElementById('bato-csv-btn')) return;
        const likedBtn = document.querySelector(SITE_CONFIG.buttonSelector);
        if (!likedBtn || !likedBtn.parentElement) return;
        
        const btn = document.createElement('button');
        btn.id = 'bato-csv-btn';
        btn.className = 'btn btn-xs rounded btn-success ml-2 font-bold text-white';
        btn.innerText = 'Export';
        btn.onclick = () => UI.show(false);
        likedBtn.parentElement.appendChild(btn);
    }

    function initHistoryPage() {
        if (document.getElementById('history-export-btn')) return;
        
        // Look for the history page title "My History by Profile"
        const pageTitle = document.querySelector('h2.text-sky-500');
        if (pageTitle && pageTitle.textContent.includes('History')) {
            const btn = document.createElement('button');
            btn.id = 'history-export-btn';
            btn.className = 'btn btn-xs rounded btn-success ml-3 font-bold text-white';
            btn.innerText = 'Export';
            btn.onclick = () => UI.show(true);
            pageTitle.appendChild(btn);
            return;
        }
        
        // Fallback: look for any h2 element
        const h2 = document.querySelector('main h2');
        if (h2) {
            const btn = document.createElement('button');
            btn.id = 'history-export-btn';
            btn.className = 'btn btn-xs rounded btn-success ml-3 font-bold text-white';
            btn.innerText = 'Export';
            btn.onclick = () => UI.show(true);
            h2.appendChild(btn);
            return;
        }
        
        // Last fallback: floating button
        const btn = document.createElement('button');
        btn.id = 'history-export-btn';
        btn.className = 'btn btn-success';
        btn.style.cssText = 'position: fixed; top: 80px; right: 20px; z-index: 9999; padding: 10px 20px; font-weight: bold;';
        btn.innerText = 'Export History';
        btn.onclick = () => UI.show(true);
        document.body.appendChild(btn);
    }

    function tryInit(attempts = 0) {
        if (isHistoryPage) {
            if (document.getElementById('history-export-btn')) return;
            initHistoryPage();
            if (!document.getElementById('history-export-btn') && attempts < 10) {
                setTimeout(() => tryInit(attempts + 1), 500);
            }
        } else if (isListPage) {
            if (document.getElementById('bato-csv-btn')) return;
            initListPage();
            if (!document.getElementById('bato-csv-btn') && attempts < 10) {
                setTimeout(() => tryInit(attempts + 1), 500);
            }
        }
    }

    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', () => tryInit());
    } else {
        tryInit();
    }
})();
