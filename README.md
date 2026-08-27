# Oflayn-Mahsulotlar-Katalogi-IndexedDB-
HTML Sahifa (index.html)
HTML
<!DOCTYPE html>
<html lang="uz">
<head>
    <meta charset="UTF-8">
    <title>Oflayn Mahsulotlar Katalogi (IndexedDB)</title>
    <style>
        body {
            font-family: sans-serif;
            max-width: 800px;
            margin: 30px auto;
            padding: 0 15px;
            background: #f8f9fa;
        }
        .card {
            background: #fff;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 8px rgba(0,0,0,0.1);
            margin-bottom: 20px;
        }
        .form-group {
            display: flex;
            gap: 10px;
            margin-bottom: 10px;
            flex-wrap: wrap;
        }
        input {
            padding: 8px;
            border: 1px solid #ccc;
            border-radius: 4px;
            flex: 1;
            min-width: 150px;
        }
        button {
            padding: 8px 15px;
            cursor: pointer;
            border: none;
            background: #007bff;
            color: white;
            border-radius: 4px;
            font-weight: bold;
        }
        button:hover { background: #0056b3; }
        .danger-btn { background: #dc3545; }
        .danger-btn:hover { background: #a71d2a; }
        
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 10px;
            background: #fff;
        }
        th, td {
            border: 1px solid #dee2e6;
            padding: 10px;
            text-align: left;
        }
        th { background: #e9ecef; }
        .status {
            font-size: 13px;
            color: #6c75d7;
            margin-bottom: 10px;
        }
    </style>
</head>
<body>

    <div class="card">
        <h2>Oflayn Mahsulotlar Katalogi (IndexedDB)</h2>
        <div id="status" class="status">Baza ulanmoqda...</div>

        <!-- 3-qism: Forma orqali yangi mahsulot qo'shish -->
        <form id="product-form" class="form-group">
            <input type="text" id="input-nom" placeholder="Mahsulot nomi..." required>
            <input type="number" id="input-narx" placeholder="Narxi (so'm)" step="0.01" required>
            <input type="number" id="input-miqdor" placeholder="Miqdori" required>
            <button type="submit">Qo'shish</button>
        </form>
    </div>

    <div class="card">
        <h3>Mahsulotlar Ro'yxati</h3>
        <table id="product-table">
            <thead>
                <tr>
                    <th>ID</th>
                    <th>Nomi</th>
                    <th>Narxi</th>
                    <th>Miqdori</th>
                    <th>Amallar</th>
                </tr>
            </thead>
            <tbody id="product-list">
                <!-- Dinamik to'ldiriladi -->
            </tbody>
        </table>
    </div>

    <!-- JavaScript skripti -->
    <script src="./app.js"></script>
</body>
</html>
2. JavaScript Mantig'i (app.js)
JavaScript
let db = null;
const DB_NAME = "DokonDB";
const STORE_NAME = "mahsulotlar";
const DB_VERSION = 1;

const statusEl = document.getElementById('status');
const formEl = document.getElementById('product-form');
const inputNom = document.getElementById('input-nom');
const inputNarx = document.getElementById('input-narx');
const inputMiqdor = document.getElementById('input-miqdor');
const productListEl = document.getElementById('product-list');

// --- 1. MA'LUMOTLAR BAZASINI SOZLASH ---

function initDB() {
    const request = indexedDB.open(DB_NAME, DB_VERSION);

    request.onerror = (event) => {
        console.error("IndexedDB ochishda xatolik:", event.target.error);
        statusEl.textContent = "Ma'lumotlar bazasini ochishda xatolik yuz berdi!";
    };

    request.onsuccess = (event) => {
        db = event.target.result;
        statusEl.textContent = "Ma'lumotlar bazasiga muvaffaqiyatli ulandi ✓ (Oflayn rejim)";
        barchaMahsulotlar(); // Sahifa ochilganda mavjud ma'lumotlarni yuklash
    };

    request.onupgradeneeded = (event) => {
        const database = event.target.result;
        
        // Agar object store mavjud bo'lmasa, yaratamiz
        if (!database.objectStoreNames.contains(STORE_NAME)) {
            const store = database.createObjectStore(STORE_NAME, { keyPath: "id", autoIncrement: true });
            
            // Talab qilingan indekslarni yaratish
            store.createIndex("nom", "nom", { unique: false });
            store.createIndex("narx", "narx", { unique: false });
            store.createIndex("miqdor", "miqdor", { unique: false });
            
            console.log("Object store va indekslar muvaffaqiyatli yaratildi.");
        }
    };
}


// --- 2. CRUD AMALLARI ---

// 1. Mahsulot qo'shish (readwrite transaction)
function mahsulotQosh(nom, narx, miqdor, callback) {
    if (!db) return;

    const tx = db.transaction(STORE_NAME, "readwrite");
    const store = tx.objectStore(STORE_NAME);

    const yangiMahsulot = {
        nom: nom,
        narx: parseFloat(narx),
        miqdor: parseInt(miqdor)
    };

    const request = store.add(yangiMahsulot);

    request.onsuccess = () => {
        if (callback) callback();
    };

    request.onerror = (event) => {
        console.error("Mahsulot qo'shish xatosi:", event.target.error);
        alert("Mahsulotni qo'shib bo'lmadi.");
    };
}

// 2. Barcha mahsulotlarni IDBCursor yordamida o'qish (readonly transaction)
function barchaMahsulotlar() {
    if (!db) return;

    const tx = db.transaction(STORE_NAME, "readonly");
    const store = tx.objectStore(STORE_NAME);
    const request = store.openCursor(); // IDBCursor ochish

    const mahsulotlar = [];

    request.onsuccess = (event) => {
        const cursor = event.target.result;
        if (cursor) {
            mahsulotlar.push(cursor.value);
            cursor.continue(); // Keyingi yozuvga o'tish
        } else {
            renderTable(mahsulotlar);
        }
    };

    request.onerror = (event) => {
        console.error("Cursor o'qish xatosi:", event.target.error);
    };
}

// 3. ID bo'yicha o'chirish (readwrite transaction)
function mahsulotOchir(id, callback) {
    if (!db) return;

    const tx = db.transaction(STORE_NAME, "readwrite");
    const store = tx.objectStore(STORE_NAME);
    const request = store.delete(id);

    request.onsuccess = () => {
        if (callback) callback();
    };

    request.onerror = (event) => {
        console.error("O'chirishda xatolik:", event.target.error);
    };
}

// 4. Narxni yangilash / Put amali (readwrite transaction)
function mahsulotNarxiniYangila(id, yangiNarx, callback) {
    if (!db) return;

    const tx = db.transaction(STORE_NAME, "readwrite");
    const store = tx.objectStore(STORE_NAME);

    // Avval obyektni olib, so'ngra put qilamiz
    const getReq = store.get(id);

    getReq.onsuccess = (event) => {
        const mahsulot = event.target.result;
        if (mahsulot) {
            mahsulot.narx = parseFloat(yangiNarx);
            const putReq = store.put(mahsulot);

            putReq.onsuccess = () => {
                if (callback) callback();
            };
        }
    };
}


// --- 3. INTERFEYS VA EVENTLAR ---

// Jadvalni DOM'ga render qilish
function renderTable(mahsulotlar) {
    productListEl.innerHTML = '';

    if (mahsulotlar.length === 0) {
        productListEl.innerHTML = `<tr><td colspan="5" style="text-align: center; color: #777;">Hozircha mahsulotlar yo'q</td></tr>`;
        return;
    }

    mahsulotlar.forEach(item => {
        const tr = document.createElement('tr');
        tr.innerHTML = `
            <td>${item.id}</td>
            <td>${escapeHtml(item.nom)}</td>
            <td>${item.narx.toLocaleString()} so'm</td>
            <td>${item.miqdor} dona</td>
            <td>
                <button onclick="updatePricePrompt(${item.id}, ${item.narx})">Narxni o'zgartirish</button>
                <button class="danger-btn" onclick="deleteProduct(${item.id})">O'chirish</button>
            </td>
        `;
        productListEl.appendChild(tr);
    });
}

// Forma yuborilganda mahsulot qo'shish
formEl.addEventListener('submit', (e) => {
    e.preventDefault();
    const nom = inputNom.value.trim();
    const narx = inputNarx.value;
    const miqdor = inputMiqdor.value;

    if (!nom || !narx || !miqdor) return;

    mahsulotQosh(nom, narx, miqdor, () => {
        inputNom.value = '';
        inputNarx.value = '';
        inputMiqdor.value = '';
        barchaMahsulotlar(); // Ro'yxatni yangilash
    });
});

// O'chirish funksiyasi (global scope uchun)
window.deleteProduct = function(id) {
    if (confirm("Haqiqatan ham bu mahsulotni o'chirmoqchimisiz?")) {
        mahsulotOchir(id, () => {
            barchaMahsulotlar();
        });
    }
};

// Narxni o'zgartirish prompt funksiyasi (global scope uchun)
window.updatePricePrompt = function(id, currentPrice) {
    const newPrice = prompt("Yangi narxni kiriting:", currentPrice);
    if (newPrice !== null && !isNaN(newPrice)) {
        mahsulotNarxiniYangila(id, newPrice, () => {
            barchaMahsulotlar();
        });
    }
};

// XSS himoyasi uchun yordamchi
function escapeHtml(str) {
    return String(str).replace(/[&<>'"]/g, 
        tag => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', "'": '&#39;', '"': '&quot;' }[tag] || tag)
    );
}

// Ilovani ishga tushirish
initDB();
