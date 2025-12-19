<!DOCTYPE html>
<html lang="it">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CONTROLLO PARTENZE</title>
    <style>
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; margin: 0; padding: 20px; background-color: #f4f7f6; }
        .container { max-width: 1200px; margin: auto; background: white; padding: 30px; border-radius: 12px; box-shadow: 0 4px 20px rgba(0,0,0,0.08); }
        h1 { color: #2c3e50; border-bottom: 2px solid #3498db; padding-bottom: 10px; }
        .upload-section { margin: 20px 0; padding: 20px; border: 2px dashed #3498db; border-radius: 8px; text-align: center; background: #ebf5fb; }
        input[type="file"] { cursor: pointer; }
        #stato { margin-top: 10px; font-weight: bold; color: #2980b9; }
        
        .table-wrapper { margin-top: 30px; overflow-x: auto; border: 1px solid #ddd; border-radius: 8px; }
        table { border-collapse: collapse; width: 100%; background: white; min-width: 800px; }
        th { background-color: #3498db; color: white; padding: 12px; text-align: left; font-size: 13px; text-transform: uppercase; }
        td { padding: 10px; border-bottom: 1px solid #eee; font-size: 13px; color: #333; white-space: pre; }
        tr:hover { background-color: #f1f9ff; }
        tr:nth-child(even) { background-color: #fafafa; }
    </style>
</head>
<body>

<div class="container">
    <h1>CONTROLLO PARTENZE</h1>
    <p>Carica il file "bolle". Il sistema userà la <strong>riga 6</strong> per definire le colonne.</p>
    
    <div class="upload-section">
        <input type="file" id="fileInput" accept=".txt,.csv,.dat,">
        <div id="stato">In attesa del file...</div>
    </div>

    <div class="table-wrapper" id="tableWrapper" style="display:none;">
        <table id="resultTable">
            <thead><tr id="tableHead"></tr></thead>
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
                document.getElementById('stato').innerText = "Errore nel file.";
                return;
            }

            // --- LOGICA RIGA 6 (RIFERIMENTO COLONNE) ---
            const riga6 = righe[5];
            const colonneInfo = [];
            const regex = /\S+/g; // Trova gruppi di caratteri non vuoti
            let match;

            // Identifichiamo dove inizia ogni parola nella riga 6
            while ((match = regex.exec(riga6)) !== null) {
                colonneInfo.push({
                    nome: match[0],
                    inizio: match.index
                });
            }

            // Definiamo la fine di ogni colonna (l'inizio della successiva)
            for (let i = 0; i < colonneInfo.length; i++) {
                colonneInfo[i].fine = colonneInfo[i+1] ? colonneInfo[i+1].inizio : 500; 
            }

            // --- GENERAZIONE TABELLA ---
            const tableHead = document.getElementById('tableHead');
            const tableBody = document.getElementById('tableBody');
            
            // Pulisci tabella precedente
            tableHead.innerHTML = "";
            tableBody.innerHTML = "";

            // Crea Intestazioni
            colonneInfo.forEach(col => {
                const th = document.createElement('th');
                th.innerText = col.nome;
                tableHead.appendChild(th);
            });

            // Crea Righe Dati (dalla riga 7 in poi)
            for (let i = 6; i < righe.length; i++) {
                const rigaCorrente = righe[i];
                if (rigaCorrente.trim() === "" || rigaCorrente.includes("------")) continue;

                const tr = document.createElement('tr');
                
                colonneInfo.forEach(col => {
                    const td = document.createElement('td');
                    // Ritagliamo il testo basandoci sulle coordinate della riga 6
                    const valore = rigaCorrente.substring(col.inizio, col.fine).trim();
                    td.innerText = valore;
                    tr.appendChild(td);
                });
                
                tableBody.appendChild(tr);
            }

            document.getElementById('tableWrapper').style.display = "block";
            document.getElementById('stato').innerText = "File elaborato correttamente!";
        };

        reader.readAsText(file);
    });
</script>

</body>
</html>
