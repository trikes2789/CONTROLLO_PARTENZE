<!DOCTYPE html>
<html lang="it">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CONTROLLO PARTENZE - Righello R6 e Intestazione R5</title>
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

        .actions { margin-bottom: 20px; display: flex; align-items: center; gap: 20px; }
        .btn-export { background-color: #27ae60; color: white; padding: 12px 25px; border: none; border-radius: 5px; cursor: pointer; display: none; font-weight: bold; }
        #counter { font-weight: bold; color: #1a73e8; background: #e8f0fe; padding: 8px 15px; border-radius: 20px; }
        
        .table-wrapper { overflow-x: auto; border: 1px solid #ddd; max-height: 750px; }
        table { border-collapse: collapse; width: 100%; background: white; white-space: pre; }
        th { background-color: #1a73e8; color: white; padding: 12px; position: sticky; top: 0; z-index: 10; border: 1px solid #ddd; font-size: 11px; text-align: left; }
        td { padding: 8px; border: 1px solid #ddd; font-size: 12px; }
        tr:nth-child(even) { background-color: #f9f9f9; }
        
        .row-header-r5 { background-color: #e3f2fd !important; font-weight: bold; }
        .row-ruler-r6 { background-color: #f1f8e9 !important; color: #777; font-size: 10px; }
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
        <button id="exportBtn" class="btn-export" onclick="exportData()">📥 Esporta in Excel (.xlsx)</button>
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
    let processedDataForExport = []; 
    let activeColumns = [];

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

        const line5_names = lines[4]; // Nomi (Tipo Sped, N.Sped...)
        const line6_ruler = lines[5]; // Righello (---- ----)
        
        let rawColumns = [];
        const regex = /-+/g; // Cerchiamo i blocchi di trattini
        let match;

        // 1. Usa la RIGA 6 (trattini) per trovare le coordinate
        while ((match = regex.exec(line6_ruler)) !== null) {
            // Per ogni blocco di trattini, prendiamo il nome corrispondente dalla RIGA 5
            let label = line5_names.substring(match.index, match.index + match[0].length + 2).trim();
            rawColumns.push({ 
                label: label || "Colonna", 
                start: match.index,
                end: match.index + match[0].length + 1
            });
        }

        // Estendi l'ultimo campo per sicurezza
        if(rawColumns.length > 0) rawColumns[rawColumns.length - 1].end = 2000;

        // 2. Filtra le colonne da escludere
        activeColumns = rawColumns.filter(c => {
            return !colonneDaEscludere.some(esclusa => c.label.includes(esclusa) || esclusa.includes(c.label));
        });

        const body = document.getElementById('tableBody');
        const head = document.getElementById('tableHead');
        
        // Creazione testata HTML basata su R5
        head.innerHTML = "<tr>" + activeColumns.map(c => `<th>${c.label}</th>`).join('') + "</tr>";
        body.innerHTML = "";
        processedDataForExport = [];

        // Trova l'indice della colonna autista tra quelle attive per il filtro
        let autistaColIdx = activeColumns.findIndex(c => c.label.toLowerCase().includes("autista") || c.label.toLowerCase().includes("oper"));

        lines.forEach((line, idx) => {
            if (line.trim() === "") return;
            
            // Taglio della riga secondo le coordinate del righello
            let rowValues = activeColumns.map(c => line.substring(c.start, c.end).trim());
            
            // Filtro autista vuoto (solo per le righe dei dati veri, idx > 5)
            if (idx > 5) {
                if (autistaColIdx !== -1 && rowValues[autistaColIdx] === "") return;
                // Salta righe di separazione extra
                if (line.includes("-------")) return;
            }

            const tr = document.createElement('tr');
            if (idx === 4) tr.className = "row-header-r5";
            if (idx === 5) tr.className = "row-ruler-r6";

            let exportRow = {};
            rowValues.forEach((val, i) => {
                const td = document.createElement('td');
                td.innerText = val;
                tr.appendChild(td);
                // Non includiamo la riga dei trattini nell'export Excel
                if (idx !== 5) exportRow[activeColumns[i].label] = val;
            });
            
            body.appendChild(tr);
            if (idx !== 5) processedDataForExport.push(exportRow);
        });

        document.getElementById('tableWrapper').style.display = "block";
        document.getElementById('exportBtn').style.display = "inline-block";
        document.getElementById('counter').innerText = `Righe elaborate: ${processedDataForExport.length}`;
    }

    function exportData() {
        const ws = XLSX.utils.json_to_sheet(processedDataForExport);
        const wb = XLSX.utils.book_new();
        XLSX.utils.book_append_sheet(wb, ws, "Spedizioni");
        XLSX.writeFile(wb, "Controllo_Partenze.xlsx");
    }
</script>
</body>
</html>
