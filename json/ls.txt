/*!
 * @name 合并音源
 * @description 星海 + 全豆要 + fish_music 三合一，支持 wy/tx/kg/kw/mg 全平台
 * @version v1.0.0
 * @note 本脚本整合三套音源，按平台自动 fallback；不保证长期可用
 */

const { EVENT_NAMES, request, on, send, env, utils } = globalThis.lx;

// ==================== 用户配置区域 ====================

// 酷我代理解密（可选，用于解密酷我加密无损格式 mflac/mgg）
const KW_DECRYPT_PROXY = {
    url: '',
    allowEncryptedLossless: false,
    urlParamName: 'url',
    ekeyParamName: 'ekey',
};

// ChKSz API（可选，填入后网易/QQ 优先走 SVIP 接口）
const CHKSZ_CONFIG = {
    apikey: '',
    enableNetease: true,
    enableQQ: true,
};

// fish_music 服务端
const FISH_API_URL = "https://m-api.ceseet.me";
const FISH_API_KEY = "";

// Huibq（可选，留空则跳过）
const HUIBQ_API = "";
const HUIBQ_REQUEST_KEY = "";

// 聆川（可选，留空则跳过）
const LINGCHUAN_API = "";

// ====================================================

// ==================== 常量配置 ====================

const SCRIPT_VERSION = 'v1.0.0';
const SCRIPT_NAME = 'MergedMusicSource';
const HTTP_URL_REGEX = /^https?:\/\//i;
const CACHE_TTL_MS = 6 * 60 * 60 * 1000;
const CACHE_MAX_SIZE = 500;
const TOKEN_TTL = 5 * 60 * 1000;

// 平台名
const PLATFORM_NAMES = {
    wy: '网易云音乐',
    tx: 'QQ音乐',
    kw: '酷我音乐',
    kg: '酷狗音乐',
    mg: '咪咕音乐'
};

// 各平台音质
const PLATFORM_QUALITIES = {
    wy: ['128k','320k','flac','hires','atmos','master'],
    tx: ['128k','192k','320k','flac','hires','atmos','atmos_plus','master'],
    kw: ['128k','320k','flac','hires','atmos','master'],
    kg: ['128k','320k','flac','hires','atmos','master'],
    mg: ['128k','320k','flac']
};

// 平台映射
const SOURCE_MAP = { tx: 'qq', mg: 'migu', kw: 'kw', kg: 'kg' };
const PLATFORM_TO_XINGHAI = { wy: 'netease', tx: 'tencent', kw: 'kuwo', kg: 'kugou', mg: 'migu' };

// 星海 URL 配置
const URL_CONFIG = {
    domains: {
        primary: 'yy.zddyr.top',
        fallback: 'zrcdy.dpdns.org',
        gdStudio: 'music-api.gdstudio.xyz',
        chkszNew: 'api.chksz.com'
    },
    paths: {
        backend: '/lx/api/',
        version: '/lx/versionh2.php',
        update: '/lx/vers.php',
        ip: '/ip.php',
        gdApi: '/api.php',
        chkszNetease: '/api/163_music',
        chkszQQ: '/api/qq_music'
    },
    gdParams: 'use_xbridge3=true&loader_name=forest&need_sec_link=1&sec_link_scene=im&theme=light'
};

const buildUrl = (domainKey, pathKey, extraQuery = '') => {
    const domain = URL_CONFIG.domains[domainKey];
    const path = URL_CONFIG.paths[pathKey];
    if (!domain || !path) throw new Error(`URL配置错误: ${domainKey} / ${pathKey}`);
    let url = `https://${domain}${path}`;
    if (extraQuery) {
        if (extraQuery.startsWith('&') && !path.includes('?')) url += '?' + extraQuery.substring(1);
        else url += extraQuery;
    }
    return url;
};

// 星海主 API
const XINGHAI_MAIN_API = "https://music-api.gdstudio.xyz/api.php?use_xbridge3=true&loader_name=forest&need_sec_link=1&sec_link_scene=im&theme=light";

// 溯音 API
const SUYIN_QQ_API = "https://oiapi.net/api/QQ_Music";
const SUYIN_QQ_KEY = "oiapi-ef6133b7-ac2f-dc7d-878c-d3e207a82575";
const SUYIN_163_API = "https://oiapi.net/api/Music_163";
const SUYIN_KUWO_API = "https://oiapi.net/api/Kuwo";
const SUYIN_MIGU_API = "https://api.xcvts.cn/api/music/migu";

// ChKSz 映射
const CHKSZ_NETEASE_LEVEL_MAP = {
    '128k': 'standard', '320k': 'exhigh', 'flac': 'lossless',
    'hires': 'hires', 'atmos': 'jymaster', 'master': 'jymaster'
};
const CHKSZ_QQ_SIZE_MAP = {
    '128k': '128k', '192k': '320k', '320k': '320k',
    'flac': 'flac', 'hires': 'hires',
    'atmos': 'master', 'atmos_plus': 'master', 'master': 'master'
};

// GD 映射
const GD_BR_MAP = { '128k':'128', '320k':'320', 'flac':'740', 'hires':'999' };
const GD_SUPPORTED_QUALITIES = new Set(['128k','320k','flac','hires']);

// 星海主 br
const QUALITY_TO_BR = {
    "128k": "128", "192k": "192", "320k": "320",
    flac: "740", flac24bit: "999", "24bit": "999"
};

// 音质集合
const HIRES_QUALITY_SET = new Set(["24bit", "flac", "flac24bit", "hires", "master", "atmos"]);
const QUALITY_PRIORITY = ["flac24bit", "flac", "320k", "192k", "128k"];

// 长青/念心模板
const CHANGQING_URL_TEMPLATES = {
    tx: "http://175.27.166.236/kgqq/qq.php?type=mp3&id={id}&level={level}",
    wy: "http://175.27.166.236/wy/wy.php?type=mp3&id={id}&level={level}",
    kw: "https://musicapi.haitangw.net/music/kw.php?type=mp3&id={id}&level={level}",
    kg: "https://music.haitangw.cc/kgqq/kg.php?type=mp3&id={id}&level={level}",
    mg: "https://music.haitangw.cc/musicapi/mg.php?type=mp3&id={id}&level={level}"
};
const NIANXIN_URL_TEMPLATES = {
    tx: "https://music.nxinxz.com/kgqq/tx.php?id={id}&level={level}&type=mp3",
    wy: "http://music.nxinxz.com/wy.php?id={id}&level={level}&type=mp3",
    kw: "http://music.nxinxz.com/kw.php?id={id}&level={level}&type=mp3",
    kg: "https://music.nxinxz.com/kgqq/kg.php?id={id}&level={level}&type=mp3",
    mg: "http://music.nxinxz.com/mg.php?id={id}&level={level}&type=mp3"
};

// 溯音码率
const QUALITY_TO_SUYIN_QQ_BR = {
    "128k": 7, "320k": 5, flac: 4, hires: 3, atmos: 2, master: 1, "24bit": 1
};
const QUALITY_TO_KUWO_BR = { flac: 1, "320k": 5, "128k": 7, "24bit": 1 };

// 状态
let userIp = null;
let userToken = '';
let tokenTimestamp = 0;
let clientHeader = '';
let deviceId = '';
let backendAggBlocked = false;
const urlCache = new Map();
const extraCache = new Map();

// ==================== 工具函数 ====================

function isBuffer(obj) {
    return obj && typeof obj === 'object' &&
        ((typeof Buffer !== 'undefined' && Buffer.isBuffer(obj)) ||
        (typeof obj.constructor === 'function' && obj.constructor.name === 'Buffer'));
}

function safeParseBody(body) {
    if (typeof body === 'string') {
        const trimmed = body.trim();
        if (/^[{["]/.test(trimmed)) { try { return JSON.parse(trimmed); } catch (e) {} }
        return body;
    }
    if (typeof body === 'object' && body !== null) {
        try { if (typeof body.toString === 'function' && body.toString() !== '[object Object]') body = body.toString('utf-8'); } catch (e) {}
        if (typeof body === 'object' && !isBuffer(body)) return body;
    }
    try {
        if (isBuffer(body)) {
            if (globalThis.lx?.utils?.buffer?.bufToString) body = globalThis.lx.utils.buffer.bufToString(body, 'utf-8');
            else if (typeof Buffer !== 'undefined') body = Buffer.from(body).toString('utf-8');
            else body = String(body);
        }
    } catch (e) {}
    if (typeof body === 'string') {
        const trimmed = body.trim();
        if (/^[{["]/.test(trimmed)) { try { return JSON.parse(trimmed); } catch (e) {} }
    }
    return body;
}

function safeBase64Encode(str) {
    try {
        if (globalThis.lx?.utils?.buffer?.from) {
            const buf = globalThis.lx.utils.buffer.from(str, 'utf-8');
            return globalThis.lx.utils.buffer.bufToString(buf, 'base64');
        }
        if (typeof Buffer !== 'undefined') return Buffer.from(str, 'utf-8').toString('base64');
        return btoa(unescape(encodeURIComponent(str)));
    } catch (e) {
        return str;
    }
}

function simpleGetQueryParam(url, key) {
    if (typeof url !== 'string' || !url) return null;
    const qIdx = url.indexOf('?');
    if (qIdx < 0) return null;
    let query = url.substring(qIdx + 1);
    const hashIdx = query.indexOf('#');
    if (hashIdx >= 0) query = query.substring(0, hashIdx);
    for (const p of query.split('&')) {
        const eq = p.indexOf('=');
        if (eq < 0) continue;
        if (p.substring(0, eq) === key) {
            try { return decodeURIComponent(p.substring(eq + 1)); } catch (e) { return p.substring(eq + 1); }
        }
    }
    return null;
}

function generateDeviceId() {
    return 'lx-merged-' + Math.random().toString(36).substring(2, 8) + Date.now().toString(36).slice(-4);
}

function buildClientHeader() {
    let deviceType = 'unknown';
    try {
        const p = (env?.platform || '').toLowerCase();
        if (p.includes('android')) deviceType = 'Android';
        else if (p.includes('ios')) deviceType = 'iOS';
        else if (p.includes('win')) deviceType = 'Windows';
        else if (p.includes('mac')) deviceType = 'macOS';
        else if (p.includes('linux')) deviceType = 'Linux';
    } catch (e) {}
    return `${SCRIPT_NAME}/${SCRIPT_VERSION} (${deviceType})`;
}

function generateToken(ip) {
    if (!deviceId) deviceId = generateDeviceId();
    tokenTimestamp = Date.now();
    return safeBase64Encode(JSON.stringify({
        device_id: deviceId,
        ip: ip || '0.0.0.0',
        timestamp: Math.floor(Date.now() / 1000),
        random: Math.random().toString(36).substring(2, 12)
    }));
}

function ensureTokenFresh() {
    if (!userToken || (Date.now() - tokenTimestamp) > TOKEN_TTL) {
        userToken = generateToken(userIp);
    }
}

// 星海风格 httpFetch
const httpFetch = (url, options = {}) => new Promise((resolve, reject) => {
    if (!options.noAuth) ensureTokenFresh();
    const headers = { ...(options.headers || {}) };
    if (!options.noAuth) {
        if (userToken) headers['X-Token'] = userToken;
        if (clientHeader) headers['X-Client'] = clientHeader;
    }
    if (!headers['User-Agent']) headers['User-Agent'] = 'lx-music';
    request(url, { ...options, headers }, (err, resp) => {
        if (err) return reject(err);
        resolve({ body: safeParseBody(resp.body), statusCode: resp.statusCode, headers: resp.headers || {} });
    });
});

// 全豆要风格 httpRequest
function httpRequest(url, options = { method: "GET" }) {
    return new Promise((resolve, reject) => {
        request(url, { timeout: 2000, ...options }, (err, res) => {
            if (err) return reject(new Error(`请求错误: ${err.message}`));
            let body = res?.body;
            if (typeof body === "string") {
                const trimmed = body.trim();
                if (trimmed.startsWith("{") || trimmed.startsWith("[") || trimmed.startsWith('"')) {
                    try { body = JSON.parse(trimmed); } catch (e) {}
                }
            }
            resolve({ statusCode: res?.statusCode ?? 0, headers: res?.headers || {}, body });
        });
    });
}

async function httpGet(url, params = {}) {
    const queryStr = Object.entries(params)
        .filter(([, v]) => v !== undefined && v !== null)
        .map(([k, v]) => `${encodeURIComponent(k)}=${encodeURIComponent(v)}`)
        .join("&");
    const fullUrl = url + (queryStr ? (url.includes("?") ? "&" : "?") + queryStr : "");
    const res = await httpRequest(fullUrl, { method: "GET", timeout: 2000 });
    if (res.statusCode >= 400) throw new Error(`HTTP错误: ${res.statusCode}`);
    return res.body;
}

function validateUrl(url, sourceName) {
    if (!url || typeof url !== "string") throw new Error(`${sourceName}返回空URL`);
    if (!HTTP_URL_REGEX.test(url.trim())) throw new Error(`${sourceName}非法URL格式`);
    return url.trim();
}

function selectQuality(requestedQuality, supportedQualities) {
    const requested = String(requestedQuality || "128k").toLowerCase();
    if (supportedQualities.includes(requested)) return requested;
    const idx = QUALITY_PRIORITY.indexOf(requested);
    const start = idx >= 0 ? idx : QUALITY_PRIORITY.length - 1;
    for (let i = start; i < QUALITY_PRIORITY.length; i++) {
        if (supportedQualities.includes(QUALITY_PRIORITY[i])) return QUALITY_PRIORITY[i];
    }
    for (let i = QUALITY_PRIORITY.length - 1; i >= 0; i--) {
        if (supportedQualities.includes(QUALITY_PRIORITY[i])) return QUALITY_PRIORITY[i];
    }
    return supportedQualities[0] || "128k";
}

function qualityToNetease(quality) {
    const q = String(quality || "128k").toLowerCase();
    if (q === "flac" || q === "flac24bit" || q === "hires" || q === "master" || q === "atmos") return "lossless";
    if (q === "320k" || q === "192k") return "exhigh";
    return "standard";
}

function qualityToSuyinQQ(quality) {
    const q = String(quality || "128k").toLowerCase();
    if (q === "flac24bit") return "hires";
    if (q === "192k") return "320k";
    return QUALITY_TO_SUYIN_QQ_BR[q] ? q : "128k";
}

function mapQuality(target, avail) {
    const pm = { '臻品母带': 'jymaster', '臻品音质2.0': 'sky', '臻品音质AI': 'jyeffect', '臻品音质': 'jyeffect', 'Hires 无损24-Bit': 'hires', 'Hi-Res': 'hires', 'FLAC': 'flac', '320k': '320k', '192k': '192k', '128k': '128k' };
    if (avail.includes(target)) return target;
    const m = pm[target]; if (m && avail.includes(m)) return m;
    const order = ['jymaster', 'sky', 'jyeffect', 'hires', 'flac24bit', 'master', 'flac', '320k', '192k', '128k'];
    for (const q of order) if (avail.includes(q)) return q;
    return avail[0] || '128k';
}

function getMobileUserAgent() {
    return "Mozilla/5.0 (iPhone; CPU iPhone OS 16_6 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/16.6 Mobile/15E148 Safari/604.1";
}

function getHashOrMid(songInfo) {
    return songInfo?.hash ?? songInfo?.songmid ?? songInfo?.id ?? null;
}

function getSongId(songInfo) {
    return (songInfo?.id || songInfo?.songmid || songInfo?.songId || songInfo?.hash || songInfo?.rid || songInfo?.mid || songInfo?.strMediaMid || songInfo?.mediaId || "").toString();
}

function buildCacheKey(prefix, songInfo, quality = "") {
    const name = songInfo?.name || "";
    const singer = songInfo?.singer || "";
    const album = songInfo?.albumName || songInfo?.album || "";
    return `${prefix}_${name}_${singer}_${album}_${quality}`;
}

function getCachedUrl(key) {
    const entry = urlCache.get(key);
    if (!entry) return null;
    if (Date.now() - entry.timestamp >= CACHE_TTL_MS) { urlCache.delete(key); return null; }
    return entry.url;
}

function setCachedUrl(key, url) {
    urlCache.set(key, { url, timestamp: Date.now() });
    if (urlCache.size > CACHE_MAX_SIZE) {
        const oldestKey = urlCache.keys().next().value;
        if (oldestKey !== undefined) urlCache.delete(oldestKey);
    }
}

function normalizeKeyword(keyword) {
    if (!keyword) return "";
    return String(keyword)
        .replace(/\(\s*Live\s*\)/gi, "")
        .replace(/\([^)]*\)/g, "")
        .replace(/\s+/g, "")
        .replace(/[^\w\u4e00-\u9fa5]/g, "")
        .trim()
        .toLowerCase();
}

function titleMatch(a, b) {
    const na = normalizeKeyword(a), nb = normalizeKeyword(b);
    if (!na || !nb) return true;
    return na.includes(nb) || nb.includes(na);
}

function buildSearchKeywords(songInfo) {
    const keywords = [];
    const name = songInfo?.name || "";
    const album = songInfo?.albumName || songInfo?.album || "";
    const singer = songInfo?.singer || "";
    if (name && album) {
        const kw = normalizeKeyword(name + album);
        if (kw) keywords.push({ keyword: kw, strict: true });
    }
    if (name && singer) {
        const kw = normalizeKeyword(name + singer);
        if (kw) keywords.push({ keyword: kw, strict: true });
    }
    if (name) {
        const kw = normalizeKeyword(name);
        if (kw) keywords.push({ keyword: kw, strict: false });
    }
    return keywords;
}

function songInfoMatch(responseData, songInfo) {
    const song = responseData?.song || responseData?.data?.song || "";
    const singer = responseData?.singer || responseData?.data?.singer || "";
    const album = responseData?.album || responseData?.data?.album || "";
    if (!titleMatch(song, songInfo?.name || "")) return false;
    if (songInfo?.singer && singer && !titleMatch(singer, songInfo.singer)) return false;
    if ((songInfo?.albumName || songInfo?.album) && album && !titleMatch(album, songInfo.albumName || songInfo.album)) return false;
    return true;
}

function songTitleMatch(responseData, songInfo) {
    if (!titleMatch(responseData?.title || "", songInfo?.name || "")) return false;
    if (songInfo?.singer && responseData?.artist && !titleMatch(responseData.artist, songInfo.singer)) return false;
    if ((songInfo?.albumName || songInfo?.album) && responseData?.album && !titleMatch(responseData.album, songInfo.albumName || songInfo.album)) return false;
    return true;
}

function parseMessageSongInfo(message) {
    if (!message) return null;
    const result = {};
    for (const line of String(message).split("\n")) {
        if (line.startsWith("歌名：")) result.song = line.replace("歌名：", "").trim();
        if (line.startsWith("歌手：")) result.singer = line.replace("歌手：", "").trim();
        if (line.startsWith("专辑：")) result.album = line.replace("专辑：", "").trim();
    }
    return result.song ? result : null;
}

function getPlatformSongId(platform, songInfo) {
    if (platform === "kg") return songInfo?.hash || songInfo?.songmid || songInfo?.id || songInfo?.rid || songInfo?.mid || null;
    if (platform === "tx") {
        const mid = songInfo?.meta?.qq?.mid || songInfo?.meta?.mid || songInfo?.songmid ||
            (typeof songInfo?.id === "string" && !/^\d+$/.test(songInfo.id) ? songInfo.id : null);
        if (mid) return mid;
        const songid = songInfo?.meta?.qq?.songid || songInfo?.meta?.songid ||
            (typeof songInfo?.id === "number" ? songInfo.id :
             (typeof songInfo?.id === "string" && /^\d+$/.test(songInfo.id) ? Number(songInfo.id) : null));
        if (songid) return songid;
    }
    return songInfo?.songmid || songInfo?.id || songInfo?.songId || songInfo?.rid || songInfo?.hash || null;
}

function getQQSongId(songInfo) {
    const mid = songInfo?.meta?.qq?.mid || songInfo?.meta?.mid || songInfo?.songmid ||
        (typeof songInfo?.id === "string" && !/^\d+$/.test(songInfo.id) ? songInfo.id : null);
    if (mid) return { type: "mid", value: mid };
    const songid = songInfo?.meta?.qq?.songid || songInfo?.meta?.songid ||
        (typeof songInfo?.id === "number" ? songInfo.id :
         (typeof songInfo?.id === "string" && /^\d+$/.test(songInfo.id) ? Number(songInfo.id) : null));
    if (songid) return { type: "songid", value: songid };
    return null;
}

// ==================== 星海源实现 ====================

// 酷我加密链接处理
function processKwEncryptedUrl(data, source) {
    if (source !== 'kw' || !KW_DECRYPT_PROXY.allowEncryptedLossless) return data?.url || '';
    let ekey = data?.ekey ? String(data.ekey).trim() : null;
    if (!ekey && data?.url && typeof data.url === 'string') ekey = simpleGetQueryParam(data.url, 'ekey');
    if (!ekey || !KW_DECRYPT_PROXY.url) return data?.url || '';
    const rawUrl = typeof data.url === 'string' ? data.url : String(data.url);
    try {
        return `${KW_DECRYPT_PROXY.url}?${KW_DECRYPT_PROXY.urlParamName}=${encodeURIComponent(rawUrl)}&${KW_DECRYPT_PROXY.ekeyParamName}=${encodeURIComponent(ekey)}`;
    } catch (e) { return rawUrl; }
}

async function fetchIp() {
    try {
        const r = await httpFetch(buildUrl('primary', 'ip'), { timeout: 3000 });
        if (r.body?.ip) { userIp = r.body.ip; userToken = generateToken(userIp); }
    } catch (e) {}
}

async function starSeaChkszWy(id, quality) {
    const level = CHKSZ_NETEASE_LEVEL_MAP[quality];
    if (!level) throw new Error('chksz不支持该品质');
    const url = `https://${URL_CONFIG.domains.chkszNew}${URL_CONFIG.paths.chkszNetease}?id=${id}&level=${level}&apikey=${encodeURIComponent(CHKSZ_CONFIG.apikey)}`;
    const resp = await httpFetch(url, { headers: { 'User-Agent': 'LX-Music-Mobile' }, timeout: 8000, noAuth: true });
    if (resp.statusCode !== 200 || resp.body.code !== 200 || !resp.body.data?.url) {
        throw new Error(`chksz网易失败(${resp.statusCode}): ${resp.body?.msg || '未返回url'}`);
    }
    return { url: resp.body.data.url, lyric: null, cover: resp.body.data.picUrl || null };
}

async function starSeaChkszQQ(musicInfo, quality) {
    const size = CHKSZ_QQ_SIZE_MAP[quality];
    if (!size) throw new Error('chksz不支持该品质');
    const mid = musicInfo.songmid || musicInfo.id;
    if (!mid) throw new Error('缺少QQ mid');
    const url = `https://${URL_CONFIG.domains.chkszNew}${URL_CONFIG.paths.chkszQQ}?mid=${mid}&size=${size}&type=json&apikey=${encodeURIComponent(CHKSZ_CONFIG.apikey)}`;
    const resp = await httpFetch(url, { headers: { 'User-Agent': 'LX-Music-Mobile' }, timeout: 8000, noAuth: true });
    if (resp.statusCode !== 200 || resp.body.code !== 200 || !resp.body.url) {
        throw new Error(`chksz QQ失败(${resp.statusCode}): ${resp.body?.msg || '未返回url'}`);
    }
    return { url: resp.body.url, lyric: resp.body.lrc || null, cover: resp.body.cover || null };
}

async function starSeaGDWy(id, q) {
    const br = GD_BR_MAP[q] || '320';
    const url = buildUrl('gdStudio', 'gdApi', `&${URL_CONFIG.gdParams}&types=url&source=netease&id=${id}&br=${br}`);
    let resp = await httpFetch(url, { headers: { 'User-Agent': 'LX-Music-Mobile' }, timeout: 8000, noAuth: true });
    if (q === 'hires' && (resp.statusCode !== 200 || !resp.body.url)) {
        const fallbackUrl = buildUrl('gdStudio', 'gdApi', `&${URL_CONFIG.gdParams}&types=url&source=netease&id=${id}&br=740`);
        resp = await httpFetch(fallbackUrl, { headers: { 'User-Agent': 'LX-Music-Mobile' }, timeout: 8000, noAuth: true });
    }
    if (resp.statusCode !== 200 || !resp.body.url) throw new Error(`GD接口状态${resp.statusCode}，未返回音频`);
    return { url: resp.body.url, lyric: null, cover: null };
}

async function starSeaBackend(source, musicInfo, quality) {
    const backendSource = SOURCE_MAP[source] || source;
    const baseUrl = buildUrl('primary', 'backend');
    const params = {};
    if (backendSource === 'kg') {
        const types = musicInfo._types || {};
        params.source = 'kg';
        params.quality = quality || '';
        params.songmid = musicInfo.songmid || musicInfo.id || '';
        params.albumId = musicInfo.albumId || '';
        params.mainHash = musicInfo.hash || '';
        if (types[quality]?.hash) params.hash = types[quality].hash;
    } else {
        params.source = backendSource;
        params.name = musicInfo.name || '';
        params.singer = musicInfo.singer || '';
        params.songmid = musicInfo.songmid || musicInfo.id || '';
        params.interval = musicInfo.interval || '';
        params.albumName = musicInfo.albumName || musicInfo.album || '';
        params.quality = quality || '';
    }
    const query = Object.keys(params).map(k => `${encodeURIComponent(k)}=${encodeURIComponent(params[k])}`).join('&');
    const url = `${baseUrl}?${query}`;
    const resp = await httpFetch(url, { method: 'GET', timeout: 8000 });
    if (resp.statusCode === 403) { backendAggBlocked = true; throw new Error('后端聚合接口返回403，已屏蔽'); }
    if (resp.statusCode !== 200) throw new Error(`后端接口状态${resp.statusCode}`);
    const data = resp.body;
    if (data.code !== 200 || !data.url) throw new Error(data.msg || '后端无可用链接');
    const finalUrl = processKwEncryptedUrl(data, backendSource);
    return { url: finalUrl, lyric: data.lrc || null, cover: data.picture || null };
}

// 星海主链路（按原逻辑）
async function starSeaFetch(source, musicInfo, quality) {
    const id = musicInfo.hash ?? musicInfo.songmid ?? musicInfo.id;
    if (!id) throw new Error('缺少 songId');
    let actualQuality = mapQuality(quality, PLATFORM_QUALITIES[source] || ['128k','320k','flac']);
    if (source === 'kw' && !KW_DECRYPT_PROXY.allowEncryptedLossless) {
        actualQuality = mapQuality(quality, ['128k','320k','flac']);
    }

    let result = { url: '', lyric: null, cover: null };
    let lastError = '';
    const chkszEnabled = !!(CHKSZ_CONFIG.apikey && CHKSZ_CONFIG.apikey.trim());

    if (source === 'wy') {
        if (chkszEnabled && CHKSZ_CONFIG.enableNetease) {
            try { result = await starSeaChkszWy(id, actualQuality); }
            catch (e) { lastError = `chksz网易失败: ${e.message}`; }
        }
        if (!result.url && !backendAggBlocked) {
            try { result = await starSeaBackend('wy', musicInfo, actualQuality); }
            catch (e) { lastError = `后端聚合失败: ${e.message}`; }
        }
        if (!result.url && GD_SUPPORTED_QUALITIES.has(actualQuality)) {
            try { result = await starSeaGDWy(id, actualQuality); }
            catch (e) { lastError = `GD接口失败: ${e.message}`; }
        }
    } else if (source === 'tx') {
        if (chkszEnabled && CHKSZ_CONFIG.enableQQ) {
            try { result = await starSeaChkszQQ(musicInfo, actualQuality); }
            catch (e) { lastError = `chksz QQ失败: ${e.message}`; }
        }
        if (!result.url) {
            try { result = await starSeaBackend('tx', musicInfo, actualQuality); }
            catch (e) { lastError = `后端失败: ${e.message}`; }
        }
    } else {
        try { result = await starSeaBackend(source, musicInfo, actualQuality); }
        catch (e) { lastError = `后端失败: ${e.message}`; }
    }

    extraCache.set(id, { lyric: result.lyric, cover: result.cover });
    const trimmedUrl = typeof result.url === 'string' ? result.url.trim() : '';
    if (typeof result.url !== 'string' || trimmedUrl.length < 1 || !trimmedUrl.match(/^https?:\/\//i)) {
        throw new Error(lastError || '星海链路获取失败');
    }
    return trimmedUrl;
}

// ==================== 全豆要源实现 ====================

async function qdyXinghaiMain(platform, songId, quality, songInfo) {
    const source = PLATFORM_TO_XINGHAI[platform];
    if (!source) throw new Error("星海主API不支持该平台");
    const id = songId ?? getHashOrMid(songInfo);
    if (!id) throw new Error("缺少songId");
    const selectedQuality = selectQuality(quality, ["128k", "192k", "320k", "flac", "flac24bit"]);
    const br = QUALITY_TO_BR[selectedQuality];
    if (!br) throw new Error("星海主API音质映射失败");
    const url = `${XINGHAI_MAIN_API}&types=url&source=${encodeURIComponent(source)}&id=${encodeURIComponent(id)}&br=${br}`;
    const res = await httpRequest(url, { method: "GET", headers: { "User-Agent": "LX-Music-Mobile", Accept: "application/json" } });
    const body = res.body;
    if (!body || typeof body !== "object" || !body.url) throw new Error(body?.message || "星海主API未返回可用URL");
    return body.url;
}

async function qdyHuibq(platform, songId, quality, songInfo) {
    if (!HUIBQ_API || !HUIBQ_REQUEST_KEY) throw new Error("Huibq未配置");
    const hashOrMid = songInfo?.hash ?? songInfo?.songmid;
    if (!hashOrMid) throw new Error("Huibq缺少hash/songmid");
    const selectedQuality = selectQuality(quality, ["320k", "128k"]);
    const url = `${HUIBQ_API}/url/${platform}/${encodeURIComponent(hashOrMid)}/${encodeURIComponent(selectedQuality)}`;
    const res = await httpRequest(url, {
        method: "GET",
        headers: { "Content-Type": "application/json", "User-Agent": getMobileUserAgent(), "X-Request-Key": HUIBQ_REQUEST_KEY }
    });
    const body = res.body;
    if (!body || typeof body !== "object" || Number.isNaN(Number(body.code))) throw new Error("Huibq返回无效");
    if (Number(body.code) === 0 && body.url) return body.url;
    throw new Error(body.message || `Huibq code=${body.code}`);
}

async function qdyLingchuan(platform, songId, quality, songInfo) {
    if (!LINGCHUAN_API) throw new Error("聆川未配置");
    const hashOrMid = songInfo?.hash ?? songInfo?.songmid;
    if (!hashOrMid) throw new Error("聆川缺少hash/songmid");
    const selectedQuality = selectQuality(quality, ["320k", "128k"]);
    const url = `${LINGCHUAN_API}/url?source=${encodeURIComponent(platform)}&songId=${encodeURIComponent(hashOrMid)}&quality=${encodeURIComponent(selectedQuality)}`;
    const res = await httpRequest(url, {
        method: "GET",
        headers: { "Content-Type": "application/json", "User-Agent": getMobileUserAgent() },
        follow_max: 5
    });
    const body = res.body;
    if (!body || typeof body !== "object" || Number.isNaN(Number(body.code))) throw new Error("聆川返回无效");
    if (Number(body.code) === 200 && body.url) return body.url;
    throw new Error(body.message || `聆川 code=${body.code}`);
}

async function qdySuyin163(songInfo) {
    const id = songInfo?.songmid || songInfo?.id;
    if (!id) throw new Error("溯音163缺少songmid/id");
    const res = await httpGet(SUYIN_163_API, { id });
    if (res?.code === 0 && res?.data) {
        const item = Array.isArray(res.data) ? res.data[0] : res.data;
        if (item?.url) return item.url;
    }
    throw new Error("溯音163获取失败");
}

async function qdySuyinQQ(songInfo, quality) {
    const qqId = getQQSongId(songInfo);
    if (!qqId) throw new Error("溯音QQ缺少songmid/id");
    const normalizedQuality = qualityToSuyinQQ(quality);
    const startBr = QUALITY_TO_SUYIN_QQ_BR[normalizedQuality] || QUALITY_TO_SUYIN_QQ_BR["128k"];
    const brList = [startBr, 4, 5, 7]
        .filter((val, idx, arr) => arr.indexOf(val) === idx && val >= startBr)
        .sort((a, b) => a - b);
    let lastError = null;
    for (const br of brList) {
        try {
            const reqParams = { key: SUYIN_QQ_KEY, type: "json", br, n: 1 };
            if (qqId.type === "mid") reqParams.mid = qqId.value;
            else reqParams.songid = qqId.value;
            const res = await httpGet(SUYIN_QQ_API, reqParams);
            if (res?.music) return res.music;
            if (res?.url) return res.url;
            if (res?.message) {
                const match = String(res.message).match(/音频链接[：:](.+?)(?:\n|$)/);
                if (match && match[1]) return match[1].trim();
            }
            throw new Error("溯音QQ未找到音频链接");
        } catch (e) { lastError = e; }
    }
    throw new Error(`溯音QQ全部音质尝试失败: ${lastError?.message || "unknown"}`);
}

async function qdySuyinKuwoSearch(keyword, br, songInfo = null) {
    const res = await httpGet(SUYIN_KUWO_API, { msg: keyword, n: 1, br });
    if (res?.data?.url) {
        if (songInfo && !songInfoMatch(res, songInfo)) throw new Error("溯音酷我歌曲信息不匹配");
        return res.data.url;
    }
    if (res?.message) {
        const match = String(res.message).match(/音乐链接[：:](\S+)/);
        if (match && match[1]) {
            if (songInfo) {
                const parsed = parseMessageSongInfo(res.message);
                if (parsed && !songInfoMatch(parsed, songInfo)) throw new Error("溯音酷我歌曲信息不匹配");
            }
            return match[1];
        }
    }
    throw new Error("溯音酷我未找到链接");
}

async function qdySuyinKuwo(songInfo, quality) {
    if (!songInfo?.name) throw new Error("溯音酷我需要歌曲名");
    const cacheKey = buildCacheKey("kw", songInfo, quality);
    const cached = getCachedUrl(cacheKey);
    if (cached) return cached;
    const selectedQuality = selectQuality(quality, ["flac", "320k", "128k"]);
    const br = QUALITY_TO_KUWO_BR[selectedQuality] || 1;
    const keywords = buildSearchKeywords(songInfo);
    let lastError = null;
    for (const item of keywords) {
        try {
            const url = await qdySuyinKuwoSearch(item.keyword, br, item.strict ? songInfo : null);
            if (url) { setCachedUrl(cacheKey, url); return url; }
        } catch (e) { lastError = e; }
    }
    throw new Error(`溯音酷我失败: ${lastError?.message || "unknown"}`);
}

async function qdySuyinMigu(songInfo) {
    if (!songInfo?.name) throw new Error("溯音咪咕需要歌曲名");
    const cacheKey = buildCacheKey("mg", songInfo);
    const cached = getCachedUrl(cacheKey);
    if (cached) return cached;
    const keywords = buildSearchKeywords(songInfo);
    let lastError = null;
    for (const item of keywords) {
        try {
            const res = await httpGet(SUYIN_MIGU_API, { gm: item.keyword, n: 1, num: 1, type: "json" });
            if (res?.code === 200 && res?.musicInfo) {
                if (item.strict && !songTitleMatch(res, songInfo)) throw new Error("溯音咪咕歌曲信息不匹配");
                setCachedUrl(cacheKey, res.musicInfo);
                return res.musicInfo;
            }
        } catch (e) { lastError = e; }
    }
    throw new Error(`溯音咪咕失败: ${lastError?.message || "unknown"}`);
}

async function qdyChangqing(platform, songId, quality, songInfo) {
    const template = CHANGQING_URL_TEMPLATES[platform];
    if (!template) throw new Error("长青SVIP不支持该平台");
    const id = getPlatformSongId(platform, songInfo);
    if (!id) throw new Error("长青SVIP缺少songId");
    const level = qualityToNetease(quality);
    return template.replace("{id}", encodeURIComponent(String(id))).replace("{level}", encodeURIComponent(level));
}

async function qdyNianxin(platform, songId, quality, songInfo) {
    const template = NIANXIN_URL_TEMPLATES[platform];
    if (!template) throw new Error("念心SVIP不支持该平台");
    const id = getPlatformSongId(platform, songInfo);
    if (!id) throw new Error("念心SVIP缺少songId");
    const level = qualityToNetease(quality);
    return template.replace("{id}", encodeURIComponent(String(id))).replace("{level}", encodeURIComponent(level));
}

// 全豆要链路
function buildQDYChain(platform) {
    const chain = [
        { name: "星海主", fn: qdyXinghaiMain },
        { name: "Huibq", fn: qdyHuibq },
    ];
    if (platform === "wy") chain.push({ name: "溯音163", fn: (p, id, q, info) => qdySuyin163(info) });
    if (platform === "tx") chain.push({ name: "溯音QQ", fn: (p, id, q, info) => qdySuyinQQ(info, q) });
    if (platform === "kw") chain.push({ name: "溯音酷我", fn: (p, id, q, info) => qdySuyinKuwo(info, q) });
    if (platform === "mg") chain.push({ name: "溯音咪咕", fn: (p, id, q, info) => qdySuyinMigu(info) });
    chain.push({ name: "聆川", fn: qdyLingchuan });
    chain.push({ name: "长青SVIP", fn: qdyChangqing });
    chain.push({ name: "念心SVIP", fn: qdyNianxin });
    return chain;
}

// ==================== fish_music 实现 ====================

async function fishFetch(source, musicInfo, quality) {
    const songId = musicInfo.hash ?? musicInfo.songmid ?? musicInfo.id;
    if (!songId) throw new Error("fish: 缺少 songId");
    const url = `${FISH_API_URL}/url/${source}/${songId}/${quality}`;
    const resp = await httpFetch(url, {
        method: 'GET',
        headers: {
            'Content-Type': 'application/json',
            'User-Agent': `${env ? `lx-music-${env}/${SCRIPT_VERSION}` : `lx-music-request/${SCRIPT_VERSION}`}`,
            'X-Request-Key': FISH_API_KEY,
        },
        follow_max: 5,
    });
    const body = resp.body;
    if (!body || isNaN(Number(body.code))) throw new Error('fish: 返回无效');
    if (Number(body.code) === 0 && body.data) return body.data;
    throw new Error(body.msg || `fish code=${body.code}`);
}

// ==================== 链路调度 ====================

function buildQDYFallback(platform, quality, songInfo) {
    const chain = buildQDYChain(platform);
    const songId = getHashOrMid(songInfo) || getSongId(songInfo);
    return chain.map(item => ({
        name: item.name,
        fn: () => item.fn(platform, songId, quality, songInfo)
    }));
}

// 通用多链路 fallback：每个链路一个 try，逐个尝试，返回第一个可用 URL
async function fetchWithFallback(source, musicInfo, quality) {
    const errors = [];
    const id = musicInfo.hash ?? musicInfo.songmid ?? musicInfo.id;

    // 1. 星海（自带内部分支）
    try {
        const url = await starSeaFetch(source, musicInfo, quality);
        return url;
    } catch (e) { errors.push(`星海: ${e.message}`); }

    // 2. 全豆要链路（逐条 fallback）
    const qdyChain = buildQDYFallback(source, quality, musicInfo);
    for (const item of qdyChain) {
        try {
            const url = await item.fn();
            if (url && /^https?:\/\//i.test(url.trim())) return url.trim();
            throw new Error("返回空URL");
        } catch (e) { errors.push(`全豆要-${item.name}: ${e.message}`); }
    }

    // 3. fish_music 兜底
    try {
        const url = await fishFetch(source, musicInfo, quality);
        if (url && /^https?:\/\//i.test(url.trim())) return url.trim();
    } catch (e) { errors.push(`fish: ${e.message}`); }

    throw new Error(`所有链路均失败: ${errors.join(' | ')}`);
}

// ==================== 事件处理 ====================

on(EVENT_NAMES.request, async ({ action, source, info }) => {
    if (!source || !PLATFORM_QUALITIES[source]) {
        throw new Error(`不支持的音乐源: ${source}`);
    }
    if (action === 'musicUrl') {
        if (!info?.musicInfo || !info.type) throw new Error('参数不完整');
        return fetchWithFallback(source, info.musicInfo, info.type);
    }

    const id = info?.musicInfo?.hash ?? info?.musicInfo?.songmid ?? info?.musicInfo?.id;
    const cached = extraCache.get(id);
    if (action === 'lyric') return cached?.lyric ? { lyric: cached.lyric, tlyric: '' } : null;
    if (action === 'pic') return cached?.cover || null;
    throw new Error(`不支持的操作: ${action}`);
});

// ==================== 启动 ====================

(async () => {
    deviceId = generateDeviceId();
    clientHeader = buildClientHeader();
    userToken = generateToken(null);

    const sources = {};
    Object.keys(PLATFORM_QUALITIES).forEach(p => {
        sources[p] = {
            name: PLATFORM_NAMES[p],
            type: 'music',
            actions: ['musicUrl', 'lyric', 'pic'],
            qualitys: PLATFORM_QUALITIES[p]
        };
    });

    send(EVENT_NAMES.inited, { openDevTools: false, status: true, sources });
    fetchIp();
})();