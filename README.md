<!DOCTYPE html>
<html lang="it">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CONTROLLO PARTENZE - Filtri Selezionati</title>
    <script src="https://cdn.sheetjs.com/xlsx-0.20.1/package/dist/xlsx.full.min.js"></script>
    <style>
        body { font-family: 'Segoe UI', Tahoma, sans-serif; margin: 0; padding: 20px; background-color: #f0f2f5; }
        .container { max-width: 1400px; margin: auto; background: white; padding: 30px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
        h1 { color: #1a73e8; margin-top: 0; border-bottom: 2px solid #1a73e8; padding-bottom: 10px; }

        #drop-area {
            border: 3px dashed #1a73e8; border-radius: 15px; padding: 30px;
            text-align: center; background: #f8fbff; cursor: pointer; margin-bottom: 20px;
        }
        #drop-area.highlight { background: #e1efff; }

        .filter-section {
            display: grid; grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
            gap: 12px; margin-bottom: 20px; padding: 15px;
            background: #fdfdfd; border: 1px solid #ddd; border-radius: 8px; display: none;
        }
        .filter-group label { display: block; font-size: 0.75rem; font-weight: bold; color: #555; margin-bottom: 4px; text-transform: uppercase; }
        .filter-group input { width: 100%; padding: 8px; border: 1px solid #ccc; border-radius: 4px; box-sizing: border-box; font-size: 13px; }

        .actions { margin-bottom: 20px; display: flex; align-items: center; gap: 20px; }
        .btn-export { background-color: #27ae60; color: white; padding: 12px 25px; border: none; border-radius: 5px; cursor: pointer; display: none; font-weight: bold; }
        #counter { font-weight: bold; color: #1a73e8; background: #e8f0fe; padding: 8px 15px; border-radius: 20px; }
        
        .table-wrapper { overflow-x: auto; border: 1px solid #ddd; max-height: 600px; }
        table { border-collapse: collapse; width: 100%; background: white; white-space: pre; }
        th { background-color: #1a73e8; color: white; padding: 12px; position: sticky; top: 0; z-index: 10; border: 1px solid #ddd; font-size: 11px; }
        td { padding: 8px; border: 1px solid #ddd; font-size: 12px; }
        tr:nth-child(even) { background-color: #f9f9f9; }
        .hidden-row { display: none; }
    </style>
</head>
<body>

<div class="container">
    <h1>CONTROLLO PARTENZE</h1>
    
    <div id="drop-area">
        <p>Trascina il file qui o clicca per caricarlo</p>
        <input type="file" id="fileElem" style="display:none" onchange="handleFiles(this.files)">
    </div>

    <div id="filterContainer" class="filter-section"></div>

    <div class="actions">
        <button id="exportBtn" class="btn-export" onclick="exportFilteredData()">📥 Esporta in Excel (.xlsx)</button>
        <div id="counter"></div>
    </div>

    <div id="tableWrapper" class="table-wrapper" style="display:none;">
        <table id="mainTable">
            <thead id="tableHead"></thead>
            <tbody id="tableBody"></tbody>
        </table>
    </div>
</div>

<script>
    let dropArea = document.getElementById('drop-area');
    let allData = []; 
    let activeColumns = [];

    // Colonne da ELIMINARE (come richiesto)
    const colonneDaEscludere = [
        "Collo T P.to P.to Pa", "T P.to P.to Part.", "P.to P.to Part. A", 
        "P.to Part. A Sed", "Sede Dest. I R E/U", "R E/U Autista Mit", 
        "Mittente Ora.", "Ora."
    ];

    ['dragenter', 'dragover', 'dragleave', 'drop'].forEach(e => dropArea.addEventListener(e, (ev) => ev.preventDefault()));
    dropArea.addEventListener('drop', (e) => handleFiles(e.dataTransfer.files));
    dropArea.addEventListener('click', () => document.getElementById('fileElem').click());

    function handleFiles(files) {
        let reader = new FileReader();
        reader.onload = (e) => processContent(e.target.result);
        reader.readAsText(files[0]);
    }

    function processContent(text) {
        const lines = text.split(/\r?\n/);
        if (lines.length < 6) return alert("File non valido");

        const refLine = lines[5];
        const filterNamesLine = lines[4]; 
        let rawColumns = [];
        const regex = /\S+/g;
        let match;

        // Estrazione iniziale di tutte le colonne
        while ((match = regex.exec(refLine)) !== null) {
            let label = filterNamesLine.substring(match.index, match.index + 25).trim() || match[0];
            rawColumns.push({ label: label, start: match.index });
        }
        for (let i = 0; i < rawColumns.length; i++) {
            rawColumns[i].end = rawColumns[i+1] ? rawColumns[i+1].start : 2000;
        }

        // FILTRO COLONNE: Teniamo solo quelle NON incluse nella lista nera
        activeColumns = rawColumns.filter(c => {
            return !colonneDaEscludere.some(esclusa => c.label.includes(esclusa) || esclusa.includes(c.label));
        });

        // 2. Genera Filtri UI
        const filterContainer = document.getElementById('filterContainer');
        filterContainer.innerHTML = "";
        activeColumns.forEach((col, idx) => {
            const group = document.createElement('div');
            group.className = 'filter-group';
            group.innerHTML = `<label>${col.label}</label>
                               <input type="text" data-idx="${idx}" placeholder="Cerca..." oninput="applyFilters()">`;
            filterContainer.appendChild(group);
        });
        filterContainer.style.display = 'grid';

        // 3. Carica Dati
        allData = [];
        const body = document.getElementById('tableBody');
        const head = document.getElementById('tableHead');
        head.innerHTML = "<tr>" + activeColumns.map(c => `<th>${c.label}</th>`).join('') + "</tr>";
        body.innerHTML = "";

        lines.slice(6).forEach(line => {
            if (line.trim() === "" || line.includes("-------")) return;
            
            // Controllo colonna Autista (che solitamente è l'ultima rimasta o specifica)
            // Cerchiamo l'indice della colonna che contiene "Autista" nel nome
            let autistaColIdx = activeColumns.findIndex(c => c.label.toLowerCase().includes("autista") || c.label.toLowerCase().includes("oper"));
            let valAutista = line.substring(activeColumns[autistaColIdx].start, activeColumns[autistaColIdx].end).trim();
            
            if (valAutista === "") return; // Escludi se autista vuoto

            let rowObj = { _cells: [] };
            const tr = document.createElement('tr');
            
            activeColumns.forEach(c => {
                let val = line.substring(c.start, c.end).trim();
                rowObj._cells.push(val.toLowerCase());
                const td = document.createElement('td');
                td.innerText = val;
                tr.appendChild(td);
            });
            
            rowObj._el = tr;
            body.appendChild(tr);
            allData.push(rowObj);
        });

        document.getElementById('tableWrapper').style.display = "block";
        document.getElementById('exportBtn').style.display = "inline-block";
        updateCounter();
    }

    function applyFilters() {
        const inputs = document.querySelectorAll('.filter-group input');
        
        allData.forEach(row => {
            let isVisible = true;
            inputs.forEach(input => {
                const term = input.value.toLowerCase();
                const idx = input.getAttribute('data-idx');
                if (term && !row._cells[idx].includes(term)) {
                    isVisible = false;
                }
            });
            row._el.className = isVisible ? "" : "hidden-row";
        });
        updateCounter();
    }

    function updateCounter() {
        const visibleCount = allData.filter(r => r._el.className !== "hidden-row").length;
        document.getElementById('counter').innerText = `Spedizioni attive: ${visibleCount}`;
    }

    function exportFilteredData() {
        const visibleData = allData.filter(row => row._el.className !== "hidden-row").map(row => {
            let exportRow = {};
            activeColumns.forEach((col, idx) => {
                exportRow[col.label] = row._el.cells[idx].innerText;
            });
            return exportRow;
        });

        const ws = XLSX.utils.json_to_sheet(visibleData);
        const wb = XLSX.utils.book_new();
        XLSX.utils.book_append_sheet(wb, ws, "Dati");
        XLSX.writeFile(wb, "Controllo_Partenze_Pulito.xlsx");
    }
</script>
</body>
</html>
