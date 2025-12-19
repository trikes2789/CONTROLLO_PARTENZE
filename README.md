<!DOCTYPE html>
<html lang="it">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CONTROLLO PARTENZE - Definitivo</title>
    <script src="https://cdn.sheetjs.com/xlsx-0.20.1/package/dist/xlsx.full.min.js"></script>
    <style>
        body { font-family: 'Segoe UI', Tahoma, sans-serif; margin: 0; padding: 20px; background-color: #f0f2f5; }
        .container { max-width: 1400px; margin: auto; background: white; padding: 30px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
        h1 { color: #1a73e8; margin-top: 0; border-bottom: 2px solid #1a73e8; padding-bottom: 10px; }
        #drop-area { border: 3px dashed #1a73e8; border-radius: 15px; padding: 30px; text-align: center; background: #f8fbff; cursor: pointer; margin-bottom: 20px; }
        #drop-area.highlight { background: #e1efff; }
        .actions { margin-bottom: 20px; display: flex; align-items: center; gap: 15px; }
        .btn-export { background-color: #27ae60; color: white; padding: 12px 25px; border: none; border-radius: 5px; cursor: pointer; display: none; font-weight: bold; }
        .stat-box { background: #e8f0fe; color: #1a73e8; padding: 10px 20px; border-radius: 8px; font-weight: bold; border: 1px solid #d2e3fc; }
        .table-wrapper { overflow-x: auto; border: 1px solid #ddd; max-height: 750px; }
        table { border-collapse: collapse; width: 100%; background: white; white-space: pre; }
        th { background-color: #1a73e8; color: white; padding: 12px; position: sticky; top: 0; z-index: 10; border: 1px solid #ddd; font-size: 11px; text-align: left; }
        td { padding: 8px; border: 1px solid #ddd; font-size: 12px; }
        tr:nth-child(even) { background-color: #f9f9f9; }
    </style>
</head>
<body>

<div class="container">
    <h1>CONTROLLO PARTENZE</h1>
    <div id="drop-area">
        <p>Trascina il file qui o clicca per caricarlo</p>
        <input type="file" id="fileElem" style="display:none" onchange="handleFiles(this.files)">
    </div>
    <div class="actions">
        <button id="exportBtn" class="btn-export" onclick="exportData()">📥 Esporta Excel</button>
        <div id="statColli" class="stat-box" style="display:none;">Quantità Colli: 0</div>
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
    let processedDataForExport = []; 
    let activeColumns = [];

    // ELENCO COLONNE DA ELIMINARE
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

        const line5_names = lines[4]; 
        const line6_ruler = lines[5]; 
        let rawColumns = [];
        const regex = /-+/g; 
        let match;

        // Identificazione righello
        while ((match = regex.exec(line6_ruler)) !== null) {
            let label = line5_names.substring(match.index, match.index + match[0].length + 2).trim();
            rawColumns.push({ label: label || "Colonna", start: match.index, end: match.index + match[0].length + 1 });
        }
        if(rawColumns.length > 0) rawColumns[rawColumns.length - 1].end = 2000;

        // FILTRO COLONNE (APPLICATO QUI)
        activeColumns = rawColumns.filter(c => {
            return !colonneDaEscludere.some(esclusa => c.label.includes(esclusa) || esclusa.includes(c.label));
        });

        const body = document.getElementById('tableBody');
        const head = document.getElementById('tableHead');
        head.innerHTML = "<tr>" + activeColumns.map(c => `<th>${c.label}</th>`).join('') + "</tr>";
        body.innerHTML = "";
        
        processedDataForExport = [];
        let countColliValidi = 0;

        let autistaColIdx = activeColumns.findIndex(c => c.label.toLowerCase().includes("autista") || c.label.toLowerCase().includes("oper"));
        let nSpedColIdx = activeColumns.findIndex(c => c.label.toLowerCase().includes("n.sped"));

        lines.forEach((line, idx) => {
            // 1. ELIMINA DALLA RIGA 4 IN SU (mostra solo dalla 5 [idx 4] in poi)
            if (idx < 4 || idx === 5 || line.trim() === "") return;

            let rowValues = activeColumns.map(c => line.substring(c.start, c.end).trim());

            // 2. FILTRI RIGHE (dalla riga 7 in poi)
            if (idx > 5) {
                // Filtro Autista Vuoto
                if (autistaColIdx !== -1 && rowValues[autistaColIdx] === "") return;
                // Filtro parola "Sig"
                if (rowValues.some(val => val.toLowerCase().includes("sig"))) return;
                
                // CONTEGGIO QUANTITÀ COLLI (Celle piene in N.Sped)
                if (nSpedColIdx !== -1 && rowValues[nSpedColIdx] !== "") {
                    countColliValidi++;
                }
            }

            const tr = document.createElement('tr');
            let exportRow = {};
            rowValues.forEach((val, i) => {
                const td = document.createElement('td');
                td.innerText = val;
                tr.appendChild(td);
                exportRow[activeColumns[i].label] = val;
            });
            
            body.appendChild(tr);
            processedDataForExport.push(exportRow);
        });

        document.getElementById('tableWrapper').style.display = "block";
        document.getElementById('exportBtn').style.display = "inline-block";
        document.getElementById('statColli').style.display = "block";
        document.getElementById('statColli').innerText = `Quantità Colli: ${countColliValidi}`;
    }

    function exportData() {
        const ws = XLSX.utils.json_to_sheet(processedDataForExport);
        const wb = XLSX.utils.book_new();
        XLSX.utils.book_append_sheet(wb, ws, "Dati");
        XLSX.writeFile(wb, "Controllo_Partenze.xlsx");
    }
</script>
</body>
</html>
