<!DOCTYPE html>
<html lang="it">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CONTROLLO PARTENZE</title>
    <style>
        body { font-family: 'Segoe UI', Arial, sans-serif; margin: 0; padding: 20px; background-color: #f4f7f6; }
        .container { max-width: 1300px; margin: auto; background: white; padding: 30px; border-radius: 12px; box-shadow: 0 4px 20px rgba(0,0,0,0.08); }
        h1 { color: #2c3e50; border-bottom: 2px solid #3498db; padding-bottom: 10px; margin-top: 0; }
        .upload-section { margin: 20px 0; padding: 20px; border: 2px dashed #3498db; border-radius: 8px; text-align: center; background: #ebf5fb; }
        #stato { margin-top: 10px; font-weight: bold; color: #2980b9; }
        
        .table-wrapper { margin-top: 30px; overflow-x: auto; border: 1px solid #ddd; border-radius: 8px; }
        table { border-collapse: collapse; width: 100%; background: white; min-width: 1000px; table-layout: fixed; }
        th { background-color: #3498db; color: white; padding: 12px; text-align: left; font-size: 13px; text-transform: uppercase; border: 1px solid #2980b9; }
        td { padding: 8px; border: 1px solid #eee; font-size: 12px; color: #333; overflow: hidden; text-overflow: ellipsis; white-space: pre; }
        
        /* Evidenzia la riga di riferimento */
        .row-header-reference { background-color: #d1ecf1 !important; font-weight: bold; }
        /* Stile per le righe iniziali (metadati) */
        .row-metadata { background-color: #fff3cd; font-style: italic; }
        
        tr:hover { background-color: #f1f9ff; }
    </style>
</head>
<body>

<div class="container">
    <h1>CONTROLLO PARTENZE</h1>
    <p>Elaborazione basata sulla struttura della riga 6. Visualizzazione completa di tutte le righe.</p>
    
    <div class="upload-section">
        <input type="file" id="fileInput" accept=".txt,.csv,.dat,*">
        <div id="stato">In attesa del file...</div>
    </div>

    <div class="table-wrapper" id="tableWrapper" style="display:none;">
        <table id="resultTable">
            <thead id="tableHead"></thead>
            <tbody id="tableBody"></tbody>
        </table>
    </div>
</div>

<script>
    document.getElementById('fileInput').addEventListener('change', function(e) {
        const file = e.target.files[0];
        if (!file) return;

        const reader = new FileReader();
        document.getElementById('stato').innerText = "Elaborazione in corso...";

        reader.onload = function(event) {
            const testo = event.target.result;
            const righe = testo.split(/\r?\n/);
            
            if (righe.length < 6) {
                alert("Errore: Il file deve avere almeno 6 righe.");
                return;
            }

            // 1. ANALISI RIGA 6 (Riferimento per le colonne)
            const riga6 = righe[5];
            const colonneInfo = [];
            const regex = /\S+/g;
            let match;

            while ((match = regex.exec(riga6)) !== null) {
                colonneInfo.push({
                    nome: match[0],
                    inizio: match.index
                });
            }

            // Imposta i limiti di taglio
            for (let i = 0; i < colonneInfo.length; i++) {
                colonneInfo[i].fine = colonneInfo[i+1] ? colonneInfo[i+1].inizio : 1000;
            }

            // 2. COSTRUZIONE INTESTAZIONE (Useremo i nomi trovati nella riga 6)
            const tableHead = document.getElementById('tableHead');
            const tableBody = document.getElementById('tableBody');
            tableHead.innerHTML = "";
            tableBody.innerHTML = "";

            let headerHtml = '<tr>';
            colonneInfo.forEach(col => {
                headerHtml += `<th style="width:${(col.fine - col.inizio)*8}px">${col.nome}</th>`;
            });
            headerHtml += '</tr>';
            tableHead.innerHTML = headerHtml;

            // 3. COSTRUZIONE CORPO (TUTTE le righe dalla 1 alla fine)
            righe.forEach((rigaTesto, index) => {
                if (rigaTesto.trim() === "") return;

                const tr = document.createElement('tr');
                
                // Assegna classi speciali per distinguere le sezioni
                if (index < 5) tr.className = "row-metadata";
                if (index === 5) tr.className = "row-header-reference";

                colonneInfo.forEach((col, colIndex) => {
                    const td = document.createElement('td');
                    
                    // Se siamo nelle prime 5 righe, i dati potrebbero non essere allineati.
                    // Proviamo comunque a tagliarli secondo la riga 6.
                    let valore = rigaTesto.substring(col.inizio, col.fine).trim();
                    
                    // Se è la prima colonna delle righe iniziali e il valore è vuoto ma la riga no, 
                    // scriviamo l'intera riga nella prima cella per non perdere info.
                    if (index < 5 && colIndex === 0 && valore === "") {
                        valore = rigaTesto.trim();
                    }

                    td.innerText = valore;
                    tr.appendChild(td);
                });
                
                tableBody.appendChild(tr);
            });

            document.getElementById('tableWrapper').style.display = "block";
            document.getElementById('stato').innerText = "File elaborato! Visualizzazione di tutte le " + righe.length + " righe.";
        };

        reader.readAsText(file);
    });
</script>

</body>
</html>
