<!DOCTYPE html>
<html lang="it">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CONTROLLO PARTENZE</title>
    <style>
        body { font-family: Arial, sans-serif; padding: 20px; background-color: #f0f2f5; }
        .box { background: white; padding: 20px; border-radius: 10px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        h1 { color: #1a73e8; }
        input[type="file"] { margin: 20px 0; padding: 10px; border: 2px dashed #1a73e8; width: 100%; box-sizing: border-box; }
        table { border-collapse: collapse; width: 100%; background: white; margin-top: 20px; }
        th, td { border: 1px solid #ccc; padding: 10px; text-align: left; }
        th { background-color: #1a73e8; color: white; }
        tr:nth-child(even) { background-color: #f8f9fa; }
    </style>
</head>
<body>

    <div class="box">
        <h1>CONTROLLO PARTENZE</h1>
        <p>Seleziona il file per estrarre i dati (Riferimento riga 6):</p>
        
        <input type="file" id="fileInput">
        
        <div id="stato">In attesa del file...</div>
    </div>

    <div id="tableContainer"></div>

    <script>
        document.getElementById('fileInput').addEventListener('change', function(e) {
            const file = e.target.files[0];
            if (!file) return;

            document.getElementById('stato').innerText = "Elaborazione in corso...";

            const reader = new FileReader();
            reader.onload = function(event) {
                const testo = event.target.result;
                const righe = testo.split('\n');
                
                if (righe.length < 6) {
                    alert("Errore: Il file deve avere almeno 6 righe.");
                    return;
                }

                // Usiamo la riga 6 (indice 5) per le colonne
                const intestazioni = righe[5].trim().split(/\s{2,}/);
                
                let html = '<table><thead><tr>';
                intestazioni.forEach(h => html += `<th>${h}</th>`);
                html += '</tr></thead><tbody>';

                // Dati dalla riga 7
                for (let i = 6; i < righe.length; i++) {
                    const rigaPulita = righe[i].trim();
                    if (rigaPulita === "") continue;
                    const colonne = rigaPulita.split(/\s{2,}/);
                    
                    html += '<tr>';
                    for (let j = 0; j < intestazioni.length; j++) {
                        html += `<td>${colonne[j] || ''}</td>`;
                    }
                    html += '</tr>';
                }
                html += '</tbody></table>';
                
                document.getElementById('tableContainer').innerHTML = html;
                document.getElementById('stato').innerText = "File elaborato con successo!";
            };
            reader.readAsText(file);
        });
    </script>
</body>
</html>
