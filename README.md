<!DOCTYPE html>
<html lang="it">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CONTROLLO PARTENZE - Avanzato</title>
    <script src="https://cdn.sheetjs.com/xlsx-0.20.1/package/dist/xlsx.full.min.js"></script>
    <style>
        body { font-family: 'Segoe UI', Tahoma, sans-serif; margin: 0; padding: 20px; background-color: #f0f2f5; }
        .container { max-width: 1400px; margin: auto; background: white; padding: 30px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); }
        
        h1 { color: #1a73e8; margin-top: 0; border-bottom: 2px solid #1a73e8; padding-bottom: 10px; }

        /* Area Drag & Drop */
        #drop-area {
            border: 3px dashed #1a73e8;
            border-radius: 15px;
            padding: 40px;
            text-align: center;
            background: #f8fbff;
            transition: all 0.3s;
            cursor: pointer;
            margin-bottom: 20px;
        }
        #drop-area.highlight { background: #e1efff; border-color: #0d47a1; }
        #drop-area p { font-size: 1.2rem; color: #1a73e8; margin: 0; }

        .btn-export {
            background-color: #27ae60;
            color: white;
            padding: 12px 25px;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            font-size: 1rem;
            display: none; /* Nascosto finché non c'è un file */
            margin-bottom: 20px;
        }
        .btn-export:hover { background-color: #219150; }

        .table-wrapper { overflow-x: auto; margin-top: 20px; border: 1px solid #ddd; }
        table { border-collapse: collapse; width: 100%; background: white; white-space: pre; }
        th { background-color: #1a73e8; color: white; padding: 12px; border: 1px solid #ddd; }
        td { padding: 8px; border: 1px solid #ddd; font-size: 12px; }
        
        .row-metadata { background-color: #fff9c4; } /* Prime 5 righe */
        .row-reference { background-color: #bbdefb; font-weight: bold; } /* Riga 6 */
    </style>
</head>
<body>

<div class="container">
    <h1>CONTROLLO PARTENZE</h1>
    
    <div id="drop-area">
        <p>Trascina qui il file "bolle" o clicca per selezionarlo</p>
        <input type="file" id="fileElem" accept="*/*" style="display:none" onchange="handleFiles(this.files)">
    </div>

    <button id="exportBtn" class="btn-export" onclick="exportToExcel()">📥 Scarica in Excel (.xlsx)</button>

    <div id="stato"></div>

    <div class="table-wrapper" id="tableWrapper" style="display:none;">
        <table id="mainTable">
            <thead id="tableHead"></thead>
            <tbody id="tableBody"></tbody>
        </table>
    </div>
</div>

<script>
    let dropArea = document.getElementById('drop-area');
    let exportBtn = document.getElementById('exportBtn');
    let currentData = []; // Variabile per salvare i dati per l'export

    // Impedisci comportamenti di default del browser per il drag&drop
    ['dragenter', 'dragover', 'dragleave', 'drop'].forEach(eventName => {
        dropArea.addEventListener(eventName, preventDefaults, false);
    });

    function preventDefaults (e) {
        e.preventDefault();
        e.stopPropagation();
    }

    // Effetti visivi al trascinamento
    ['dragenter', 'dragover'].forEach(eventName => {
        dropArea.addEventListener(eventName, () => dropArea.classList.add('highlight'), false);
    });
    ['dragleave', 'drop'].forEach(eventName => {
        dropArea.addEventListener(eventName, () => dropArea.classList.remove('highlight'), false);
    });

    // Gestione del rilascio file
    dropArea.addEventListener('drop', (e) => {
        let dt = e.dataTransfer;
        let files = dt.files;
        handleFiles(files);
    }, false);

    // Clic per aprire selettore file
    dropArea.addEventListener('click', () => document.getElementById('fileElem').click());

    function handleFiles(files) {
        let file = files[0];
        let reader = new FileReader();
        
        reader.onload = function(e) {
            processContent(e.target.result);
        };
        reader.readAsText(file);
    }

    function processContent(text) {
        const lines = text.split(/\r?\n/);
        if (lines.length < 6) return alert("File troppo corto");

        // 1. Analisi Struttura (Riga 6)
        const refLine = lines[5];
        const cols = [];
        const regex = /\S+/g;
        let match;
        while ((match = regex.exec(refLine)) !== null) {
            cols.push({ nome: match[0], start: match.index });
        }
        for (let i = 0; i < cols.length; i++) {
            cols[i].end = cols[i+1] ? cols[i+1].start : 1000;
        }

        // 2. Popolamento Tabella e Array Dati
        const head = document.getElementById('tableHead');
        const body = document.getElementById('tableBody');
        head.innerHTML = "<tr>" + cols.map(c => `<th>${c.nome}</th>`).join('') + "</tr>";
        body.innerHTML = "";
        currentData = [];

        lines.forEach((line, idx) => {
            if (line.trim() === "") return;
            
            let rowData = {};
            const tr = document.createElement('tr');
            if (idx < 5) tr.className = "row-metadata";
            if (idx === 5) tr.className = "row-reference";

            cols.forEach((c, cIdx) => {
                let val = line.substring(c.start, c.end).trim();
                if (idx < 5 && cIdx === 0 && val === "") val = line.trim(); // Recupero info intestazione
                
                const td = document.createElement('td');
                td.innerText = val;
                tr.appendChild(td);
                rowData[c.nome] = val; // Salviamo per Excel
            });
            
            body.appendChild(tr);
            currentData.push(rowData);
        });

        document.getElementById('tableWrapper').style.display = "block";
        exportBtn.style.display = "inline-block";
        document.getElementById('stato').innerText = "File pronto per l'esportazione.";
    }

    // FUNZIONE ESPORTAZIONE EXCEL
    function exportToExcel() {
        if (currentData.length === 0) return;
        
        // Creiamo un foglio di lavoro (worksheet) dai dati
        const ws = XLSX.utils.json_to_sheet(currentData);
        const wb = XLSX.utils.book_new();
        XLSX.utils.book_append_sheet(wb, ws, "Dati_Elaborati");
        
        // Generiamo il file e facciamo partire il download
        XLSX.writeFile(wb, "Controllo_Partenze.xlsx");
    }
</script>

</body>
</html>
