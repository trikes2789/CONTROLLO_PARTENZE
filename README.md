<!DOCTYPE html>
<html lang="it">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CONTROLLO PARTENZE - Filtro Sede</title>
    <script src="https://cdn.sheetjs.com/xlsx-0.20.1/package/dist/xlsx.full.min.js"></script>
    <style>
        body { font-family: 'Segoe UI', Tahoma, sans-serif; margin: 0; padding: 20px; background-color: #f0f2f5; }
        .container { max-width: 1400px; margin: auto; background: white; padding: 30px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
        h1 { color: #1a73e8; margin-top: 0; border-bottom: 2px solid #1a73e8; padding-bottom: 10px; }
        #drop-area { border: 3px dashed #1a73e8; border-radius: 15px; padding: 30px; text-align: center; background: #f8fbff; cursor: pointer; margin-bottom: 20px; }
        .filter-box { margin-bottom: 20px; padding: 15px; background: #fffde7; border: 1px solid #fff59d; border-radius: 8px; display: none; }
        .filter-box label { font-weight: bold; margin-right: 10px; color: #827717; }
        .filter-box input { padding: 8px; border: 1px solid #ccc; border-radius: 4px; width: 200px; }
        .actions { margin-bottom: 20px; display: flex; align-items: center; gap: 15px; flex-wrap: wrap; }
        .btn-export { background-color: #27ae60; color: white; padding: 12px 25px; border: none; border-radius: 5px; cursor: pointer; display: none; font-weight: bold; }
        .stat-box { background: #e8f0fe; color: #1a73e8; padding: 10px 20px; border-radius: 8px; font-weight: bold; border: 1px solid #d2e3fc; min-width: 180px; text-align: center; }
        .table-wrapper { overflow-x: auto; border: 1px solid #ddd; max-height: 750px; }
        table { border-collapse: collapse; width: 100%; background: white; white-space: pre; }
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

    <div id="filterContainer" class="filter-box">
        <label for="sedeFilter">Conteggio Sede Destinataria (Col. E):</label>
        <input type="text" id="sedeFilter" placeholder="Inserisci sede..." oninput="applySedeFilter()">
    </div>

    <div class="actions">
        <button id="exportBtn" class="btn-export" onclick="exportData()">📥 Esporta Excel</button>
        <div id="statColli" class="stat-box" style="display:none;">Quantità Colli: 0</div>
        <div id="statSpedizioni" class="stat-box" style="display:none;">Quantità Spedizioni: 0</div>
    </div>

    <div id="tableWrapper" class="table-wrapper" style="display:none;">
        <table id="mainTable">
            <tbody id="tableBody"></tbody>
        </table>
    </div>
</div>

<script>
    let dropArea = document.getElementById('drop-area');
    let allProcessedRows = []; 

    const colonneDaEscludereIniziali = [
        "Collo T P.to P.to Pa", "T P.to P.to Part.", "P.to P.to Part. A", 
        "P.to Part. A Sed", "P.to Part.", "Sede Dest. I R", 
        "R E/U Autista Mit", "Mittente Ora.", "Ora."
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
        const line5_names = lines[4]; 
        const line6_ruler = lines[5]; 
        let rawColumns = [];
        const regex = /-+/g; 
        let match;

        while ((match = regex.exec(line6_ruler)) !== null) {
            let label = line5_names.substring(match.index, match.index + match[0].length + 2).trim();
            rawColumns.push({ label: label || "Colonna", start: match.index, end: match.index + match[0].length + 1 });
        }
        if(rawColumns.length > 0) rawColumns[rawColumns.length - 1].end = 2000;

        let activeColumns = rawColumns.filter(c => !colonneDaEscludereIniziali.some(esc => c.label.includes(esc)));

        const body = document.getElementById('tableBody');
        body.innerHTML = "";
        allProcessedRows = [];
        let tableRowCounter = 0;

        lines.forEach((line, idx) => {
            if (idx < 4 || idx === 5 || line.trim() === "") return;

            let rowValues = activeColumns.map(c => line.substring(c.start, c.end).trim());
            let autistaColIdx = activeColumns.findIndex(c => c.label.toLowerCase().includes("autista") || c.label.toLowerCase().includes("oper"));
            
            if (idx > 5) {
                if (autistaColIdx !== -1 && rowValues[autistaColIdx] === "") return;
                if (rowValues.some(val => val.toLowerCase().includes("sig"))) return;
            }

            if (idx === 4) return;
            tableRowCounter++;

            if (tableRowCounter >= 3) {
                if (!rowValues[0] || !rowValues[0].includes("V8")) return;
            }

            let finalRowValues = rowValues.filter((_, i) => i !== 2 && i !== 5 && i !== 7);

            const tr = document.createElement('tr');
            finalRowValues.forEach(val => {
                const td = document.createElement('td');
                td.innerText = val;
                tr.appendChild(td);
            });
            
            body.appendChild(tr);
            
            // Salviamo l'oggetto riga per il filtro dinamico
            allProcessedRows.push({
                values: finalRowValues,
                element: tr,
                isData: idx > 5
            });
        });

        document.getElementById('tableWrapper').style.display = "block";
        document.getElementById('exportBtn').style.display = "inline-block";
        document.getElementById('filterContainer').style.display = "block";
        document.getElementById('statColli').style.display = "block";
        document.getElementById('statSpedizioni').style.display = "block";
        
        applySedeFilter(); // Inizializza i contatori
    }

    function applySedeFilter() {
        const filterValue = document.getElementById('sedeFilter').value.toLowerCase().trim();
        let totalColli = 0;
        let spedizioniUniche = new Set();

        allProcessedRows.forEach(row => {
            // Colonna E è l'indice 4 (A=0, B=1, C=2, D=3, E=4)
            const sedeValue = row.values[4] ? row.values[4].toLowerCase() : "";
            const matchesFilter = filterValue === "" || sedeValue.includes(filterValue);

            if (matchesFilter) {
                row.element.classList.remove('hidden-row');
                // Aggiorna contatori solo per le righe dati visibili
                if (row.isData && row.values[1] && row.values[1] !== "") {
                    totalColli++;
                    spedizioniUniche.add(row.values[1].trim());
                }
            } else {
                row.element.classList.add('hidden-row');
            }
        });

        document.getElementById('statColli').innerText = `Quantità Colli: ${totalColli}`;
        document.getElementById('statSpedizioni').innerText = `Quantità Spedizioni: ${spedizioniUniche.size}`;
    }

    function exportData() {
        const visibleData = allProcessedRows
            .filter(r => !r.element.classList.contains('hidden-row'))
            .map(r => r.values);

        const ws = XLSX.utils.aoa_to_sheet(visibleData);
        const wb = XLSX.utils.book_new();
        XLSX.utils.book_append_sheet(wb, ws, "Dati_Filtrati");
        XLSX.writeFile(wb, "Controllo_Partenze_Sede.xlsx");
    }
</script>
</body>
</html>
