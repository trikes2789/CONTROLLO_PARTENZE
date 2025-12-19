# CONTROLLO_PARTENZE<!DOCTYPE html>
<html lang="it">
<head>
    <meta charset="UTF-8">
    <title>Elaboratore Bolle</title>
    <style>
        body { font-family: sans-serif; margin: 20px; color: #333; }
        table { border-collapse: collapse; width: 100%; margin-top: 20px; font-size: 12px; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f4f4f4; position: sticky; top: 0; }
        tr:nth-child(even) { background-color: #f9f9f9; }
        .controls { background: #eee; padding: 15px; border-radius: 8px; }
    </style>
</head>
<body>

    <h2>Elaboratore Automatico File</h2>
    
    <div class="controls">
        <label>Seleziona il file "bolle": </label>
        <input type="file" id="fileInput" accept="*/*">
        <p><small>Verrà usata la <strong>riga 6</strong> come riferimento per le colonne.</small></p>
    </div>

    <div id="tableContainer"></div>

    <script>
        document.getElementById('fileInput').addEventListener('change', function(e) {
            const file = e.target.files[0];
            if (!file) return;

            const reader = new FileReader();
            reader.onload = function(event) {
                const content = event.target.result;
                elaboraTabella(content);
            };
            reader.readAsText(file);
        });

        function elaboraTabella(testo) {
            const righe = testo.split('\n');
            
            // Verifichiamo che ci siano abbastanza righe
            if (righe.length < 6) {
                alert("Il file non ha abbastanza righe.");
                return;
            }

            // RIGA 6 (indice 5): La usiamo per creare le intestazioni
            // La regex /\s{2,}/ divide il testo dove trova 2 o più spazi consecutivi
            const intestazioni = righe[5].trim().split(/\s{2,}/);

            let html = '<table><thead><tr>';
            intestazioni.forEach(h => {
                html += `<th>${h}</th>`;
            });
            html += '</tr></thead><tbody>';

            // DATI: Partiamo dalla riga 7 (indice 6)
            for (let i = 6; i < righe.length; i++) {
                const rigaPulita = righe[i].trim();
                if (rigaPulita === "") continue; // Salta righe vuote

                const colonne = rigaPulita.split(/\s{2,}/);
                
                html += '<tr>';
                // Riordiniamo le colonne in base alle intestazioni trovate
                for (let j = 0; j < intestazioni.length; j++) {
                    html += `<td>${colonne[j] || ''}</td>`;
                }
                html += '</tr>';
            }

            html += '</tbody></table>';
            document.getElementById('tableContainer').innerHTML = html;
        }
    </script>
</body>
</html>
