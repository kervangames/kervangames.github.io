import { writeFile, mkdir, readFile } from "node:fs/promises";
import { existsSync } from "node:fs";
import path from "node:path";
import { fileURLToPath } from "node:url";
import gplay from "google-play-scraper";

// ---- Tek yapılandırma noktası ----
// devId, https://play.google.com/store/apps/developer?id=KERVAN bağlantısındaki
// "id" parametresinden alınmıştır.
const CONFIG = {
    devId: "KERVAN",
    lang: "tr",
    country: "tr",
    num: 60,
    fullDetail: false
};

const MAX_ATTEMPTS = 3;
const RETRY_DELAY_MS = 3000;

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
const REPO_ROOT = path.resolve(__dirname, "..");
const DATA_DIR = path.join(REPO_ROOT, "data");
const OUTPUT_FILE = path.join(DATA_DIR, "playstore-apps.json");

function sleep(ms) {
    return new Promise((resolve) => setTimeout(resolve, ms));
}

function toSafeString(value) {
    return typeof value === "string" && value.trim().length > 0 ? value.trim() : null;
}

function toSafeNumber(value) {
    const num = Number(value);
    return Number.isFinite(num) ? num : null;
}

function toIsoDate(value) {
    if (!value) {
        return null;
    }
    const date = new Date(value);
    return Number.isNaN(date.getTime()) ? null : date.toISOString();
}

function normalizeApp(rawApp) {
    const appId = toSafeString(rawApp && rawApp.appId);
    const title = toSafeString(rawApp && rawApp.title);

    if (!appId || !title) {
        return null;
    }

    const priceValue = toSafeNumber(rawApp.price);
    const free = typeof rawApp.free === "boolean" ? rawApp.free : priceValue === 0;

    return {
        appId,
        title,
        summary: toSafeString(rawApp.summary) || toSafeString(rawApp.description) || null,
        icon: toSafeString(rawApp.icon),
        url: toSafeString(rawApp.url) || `https://play.google.com/store/apps/details?id=${appId}`,
        developer: toSafeString(rawApp.developer),
        developerId: toSafeString(rawApp.developerId),
        genre: toSafeString(rawApp.genre),
        score: toSafeNumber(rawApp.score),
        scoreText: toSafeString(rawApp.scoreText),
        ratings: toSafeNumber(rawApp.ratings),
        price: priceValue,
        priceText: toSafeString(rawApp.priceText) || (free ? "Ücretsiz" : null),
        free,
        released: toSafeString(rawApp.released),
        updated: toIsoDate(rawApp.updated)
    };
}

function dedupeApps(apps) {
    const map = new Map();
    for (const app of apps) {
        if (app && app.appId && !map.has(app.appId)) {
            map.set(app.appId, app);
        }
    }
    return Array.from(map.values());
}

async function fetchDeveloperApps() {
    let lastError = null;

    for (let attempt = 1; attempt <= MAX_ATTEMPTS; attempt += 1) {
        try {
            const results = await gplay.developer({
                devId: CONFIG.devId,
                lang: CONFIG.lang,
                country: CONFIG.country,
                num: CONFIG.num,
                fullDetail: CONFIG.fullDetail
            });

            if (!Array.isArray(results)) {
                throw new Error("Beklenmeyen yanıt biçimi: sonuç bir dizi değil.");
            }

            return results;
        } catch (error) {
            lastError = error;
            console.error(`[update-play-store] Deneme ${attempt}/${MAX_ATTEMPTS} başarısız: ${error.message}`);

            if (attempt < MAX_ATTEMPTS) {
                await sleep(RETRY_DELAY_MS * attempt);
            }
        }
    }

    throw lastError || new Error("Bilinmeyen hata: geliştirici uygulamaları alınamadı.");
}

async function readExistingFile() {
    try {
        const raw = await readFile(OUTPUT_FILE, "utf8");
        const parsed = JSON.parse(raw);
        return Array.isArray(parsed.apps) ? parsed : null;
    } catch (error) {
        return null;
    }
}

async function main() {
    console.log(`[update-play-store] "${CONFIG.devId}" geliştiricisi için Google Play verisi alınıyor...`);

    let rawApps;
    try {
        rawApps = await fetchDeveloperApps();
    } catch (error) {
        console.error("[update-play-store] Google Play verisi alınamadı, mevcut JSON dosyasına dokunulmadı.");
        console.error(`[update-play-store] Hata: ${error.message}`);
        process.exitCode = 1;
        return;
    }

    const normalized = rawApps.map(normalizeApp).filter((app) => app !== null);
    const deduped = dedupeApps(normalized);

    if (deduped.length === 0) {
        const existing = await readExistingFile();
        console.error("[update-play-store] Geçerli hiçbir uygulama bulunamadı. Güvenlik için mevcut dosya korunuyor.");
        if (!existing) {
            console.error("[update-play-store] Uyarı: korunacak mevcut bir dosya da yok.");
        }
        process.exitCode = 1;
        return;
    }

    deduped.sort((a, b) => {
        const dateA = a.updated ? new Date(a.updated).getTime() : 0;
        const dateB = b.updated ? new Date(b.updated).getTime() : 0;
        return dateB - dateA;
    });

    const payload = {
        source: "google-play",
        developerId: CONFIG.devId,
        country: CONFIG.country,
        language: CONFIG.lang,
        updatedAt: new Date().toISOString(),
        apps: deduped
    };

    if (!existsSync(DATA_DIR)) {
        await mkdir(DATA_DIR, { recursive: true });
    }

    await writeFile(OUTPUT_FILE, `${JSON.stringify(payload, null, 2)}\n`, "utf8");

    console.log(`[update-play-store] ${deduped.length} uygulama yazıldı: ${OUTPUT_FILE}`);
}

main().catch((error) => {
    console.error("[update-play-store] Beklenmeyen hata:", error);
    process.exitCode = 1;
});
