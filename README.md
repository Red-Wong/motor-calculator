<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Calcul moteur</title>
  <style>
    body { 
      font-family: Arial, sans-serif; 
      max-width:800px; 
      margin:0 auto; 
      padding:0;
      line-height: 1.6;
    }
    .header-banner {
      background: #ff0000;
      color: white;
      padding: 15px 20px;
      text-align: center;
      font-weight: bold;
      font-size: 1.2em;
      margin-bottom: 20px;
    }
    .container {
      padding: 0 20px 20px;
    }
    .form-group { 
      margin-bottom:15px;
      position: relative;
    }
    label { 
      display:block; 
      font-weight:bold; 
      margin-bottom:5px;
    }
    input, select { 
      width:100%; 
      padding:10px; 
      margin-bottom:5px;
      box-sizing: border-box;
    }
    button { 
      background:#4CAF50; 
      color:white; 
      padding:10px 20px; 
      border:none; 
      cursor:pointer;
      width: 100%;
      font-size: 1.1em;
      margin-top: 10px;
    }
    .resultat { 
      margin-top:20px; 
      font-size:1.1em; 
      padding:15px; 
      background:#f1f1f1; 
      border-radius:5px;
    }
    .suggestions { 
      border:1px solid #ccc; 
      max-height:150px; 
      overflow-y:auto; 
      display:none;
      position: absolute;
      width: 100%;
      z-index: 100;
      background: white;
    }
    .suggestions div { 
      padding:8px; 
      cursor:pointer; 
      background:#fff; 
    }
    .suggestions div:hover { 
      background:#f0f0f0; 
    }
    .error { 
      color:red; 
      font-size:0.9em; 
      margin-top:5px; 
    }
    .info-text {
      font-size: 0.8em;
      color: #666;
      margin-top: -5px;
      margin-bottom: 10px;
    }
    .help-icon {
      color: #06c;
      cursor: help;
      margin-left: 5px;
    }
    @media (max-width: 600px) {
      body {
        padding: 0;
      }
      .container {
        padding: 0 10px 10px;
      }
      input, select {
        padding: 8px;
      }
    }
  </style>
</head>
<body>
<div class="header-banner">Red Wong Super Motor Calculator</div>
<div class="container">
<h1>Calcul des éléments de protection moteur</h1>
<form id="formulaire">
  <div class="form-group">
    <label>Puissance (HP) <small>ex: 0.5 pour 1/2 HP</small></label>
    <input type="number" name="hp" step="0.01" />
  </div>
  <div class="form-group">
    <label>Tension (V)</label>
    <select name="tension">
      <option value="120">120V</option>
      <option value="240">240V</option>
    </select>
  </div>
  <div class="form-group">
    <label>Type</label>
    <select name="acdc">
      <option value="AC">AC</option>
      <option value="DC">DC</option>
    </select>
  </div>
  <div class="form-group">
    <label>Type de service</label>
    <select name="service">
      <option value="Temporaire">Temporaire</option>
      <option value="Intermittent">Intermittent</option>
      <option value="Périodique">Périodique</option>
      <option value="Variable">Variable</option>
    </select>
  </div>
  <div class="form-group">
    <label>Durée</label>
    <select name="duree">
      <option value="5 minutes">5 minutes</option>
      <option value="15 minutes">15 minutes</option>
      <option value="30 minutes">30 minutes</option>
      <option value="60 minutes">60 minutes</option>
      <option value="continu">Continu</option>
    </select>
  </div>
  <div class="form-group">
    <label>Service factor (SF)</label>
    <input type="number" name="sf" step="0.01" />
    <div class="info-text">Quand SF est inconnu, mettre 0</div>
  </div>
  <div class="form-group">
    <label>Nombre de conducteurs EMT</label>
    <input type="number" name="nbConducteursEMT" step="1" min="1" />
  </div>
  <div class="form-group">
    <label>Nombre de conducteurs PVC</label>
    <input type="number" name="nbConducteursPVC" step="1" min="1" />
  </div>
  <div class="form-group">
    <label>Nombre de conducteurs non métalliques flexibles</label>
    <input type="number" name="nbConducteursFlex" step="1" min="1" />
  </div>
  <div class="form-group">
    <label>Type de conducteur</label>
    <input type="text" id="typeConducteur" placeholder="Exemple : RW75XLPE sans enveloppe" />
    <div id="suggestions" class="suggestions"></div>
    <div id="errorMessage" class="error"></div>
  </div>
  <button type="submit">Calculer</button>
</form>
<div id="resultat" class="resultat" style="display:none;"></div>
</div>

<script>
// Fonction pour parser correctement les lignes CSV
function parseCSVLine(line) {
  const result = [];
  let inQuotes = false;
  let currentField = '';
  for (let char of line) {
    if (char === '"') {
      inQuotes = !inQuotes;
    } else if (char === ',' && !inQuotes) {
      result.push(currentField.trim());
      currentField = '';
    } else {
      currentField += char;
    }
  }
  result.push(currentField.trim());
  return result;
}
document.addEventListener("DOMContentLoaded", async () => {
  // Charger les types de conducteurs depuis T10A.csv
  const t10Atxt = await (await fetch("Calcul%20moteur_csv/T10A.csv")).text();
  const L10A = t10Atxt.split("\n").filter(l => l.trim());
  const H10A = parseCSVLine(L10A[0]);
  const typesConducteur = H10A.slice(1); // Ignorer la première colonne "Clibre"
  // Gestion de l'autocomplétion
  const input = document.getElementById("typeConducteur");
  const suggestionsDiv = document.getElementById("suggestions");
  const errorMessage = document.getElementById("errorMessage");
  input.addEventListener("input", () => {
    const query = input.value.trim().toLowerCase();
    errorMessage.textContent = ""; // Effacer les messages d'erreur précédents
    suggestionsDiv.innerHTML = ""; // Effacer les suggestions précédentes
    if (!query) {
      suggestionsDiv.style.display = "none";
      return;
    }
    const matches = typesConducteur.filter(type =>
      type.toLowerCase().includes(query)
    );
    if (matches.length > 0) {
      matches.forEach(match => {
        const div = document.createElement("div");
        div.textContent = match;
        div.addEventListener("click", () => {
          input.value = match;
          suggestionsDiv.style.display = "none";
        });
        suggestionsDiv.appendChild(div);
      });
      suggestionsDiv.style.display = "block";
    } else {
      errorMessage.textContent = "Aucun type de conducteur ne correspond à votre saisie.";
      suggestionsDiv.style.display = "none";
    }
  });
  // Masquer les suggestions lorsqu'on clique ailleurs
  document.addEventListener("click", (e) => {
    if (!suggestionsDiv.contains(e.target) && e.target !== input) {
      suggestionsDiv.style.display = "none";
    }
  });
});
document.getElementById("formulaire").addEventListener("submit", async function (e) {
  e.preventDefault();
  const d = {
    hp: parseFloat(this.hp.value),
    tension: this.tension.value,
    acdc: this.acdc.value,
    service: this.service.value,
    duree: this.duree.value,
    sf: parseFloat(this.sf.value),
    nbConducteursEMT: parseInt(this.nbConducteursEMT.value),
    nbConducteursPVC: parseInt(this.nbConducteursPVC.value),
    nbConducteursFlex: parseInt(this.nbConducteursFlex.value),
    typeConducteur: document.getElementById("typeConducteur").value.trim()
  };
  try {
    // 1. Trouver le FLA de base
    const tab = d.acdc === "AC" ? "T45" : "TD2";
    const txt = await (await fetch(`Calcul%20moteur_csv/${tab}.csv`)).text();
    const L = txt.split("\n").filter(l => l.trim());
    const H = L[0].split(",").map(h => h.trim());
    const iV = H.indexOf(d.tension + "V"), iHP = H.indexOf("HP");
    if (iV < 0 || iHP < 0) throw "Colonnes manquantes dans FLA";
    let FLA = null;
    for (let i = 1; i < L.length; i++) {
      const c = L[i].split(",");
      if (parseFloat(c[iHP]) === d.hp) {
        FLA = parseFloat(c[iV]);
        break;
      }
    }
    if (FLA == null) throw "FLA non trouvé";
    // 2. Correction du FLA avec T27
    const t27txt = await (await fetch("Calcul%20moteur_csv/T27.csv")).text();
    const L27 = t27txt.split("\n").filter(l => l.trim());
    const H27 = L27[0].split(",").map(h => h.trim().toLowerCase());
    const iDuree = H27.indexOf(d.duree.toLowerCase());
    if (iDuree < 0) throw "Durée non trouvée dans T27";
    let facteurCorrection = null;
    for (let i = 1; i < L27.length; i++) {
      const row = L27[i].split(",").map(x => x.trim().toLowerCase());
      if (row[0] === d.service.toLowerCase()) {
        facteurCorrection = parseFloat(row[iDuree]);
        break;
      }
    }
    if (facteurCorrection == null) throw "Facteur de correction non trouvé dans T27";
    const FLAcorrige = FLA * facteurCorrection;
    // 3. Calcul du relais de surcharge (basé sur FLA brut)
    const relais = (d.sf >= 1.15) ? FLA * 1.25 : FLA * 1.15;
    // 4. Calcul des fusibles (maintenant basés sur FLA brut)
    const fusibleTValue = d.acdc === "AC" ? FLA * 1.75 : FLA * 1.5;
    const fusibleNTValue = d.acdc === "AC" ? FLA * 3 : FLA * 1.5;
    // 5. Trouver les protections dans T13
    const t13txt = await (await fetch("Calcul%20moteur_csv/T13.csv")).text();
    const L13 = t13txt.split("\n").filter(l => l.trim());
    const H13 = L13[0].split(",").map(h => h.trim());
    const iFusible = H13.indexOf("Fusible"), iSectionneur = H13.indexOf("Sectionneur");
    if (iFusible < 0 || iSectionneur < 0) throw "Colonnes manquantes dans T13";
    const valeursFusibles = [];
    for (let i = 1; i < L13.length; i++) {
      const c = L13[i].split(",").map(v => v.trim());
      if (c[iFusible] && !isNaN(parseFloat(c[iFusible]))) {
        valeursFusibles.push(parseFloat(c[iFusible]));
      }
    }
    valeursFusibles.sort((a, b) => a - b);
    const trouverValeurSuperieure = (valeur) => {
      for (const val of valeursFusibles) {
        if (val >= valeur) return val;
      }
      return "non trouvé";
    };
    const fusibleT = trouverValeurSuperieure(fusibleTValue);
    const fusibleNT = trouverValeurSuperieure(fusibleNTValue);
    let secT = "non trouvé", secNT = "non trouvé";
    for (let i = 1; i < L13.length; i++) {
      const c = L13[i].split(",").map(v => v.trim());
      if (parseFloat(c[iFusible]) === fusibleT) {
        secT = c[iSectionneur] || "non trouvé";
      }
      if (parseFloat(c[iFusible]) === fusibleNT) {
        secNT = c[iSectionneur] || "non trouvé";
      }
    }
    // 6. Trouver le calibre des conducteurs dans T2
    const t2txt = await (await fetch("Calcul%20moteur_csv/T2.csv")).text();
    const L2 = t2txt.split("\n").filter(l => l.trim());
    const H2 = parseCSVLine(L2[0]);
    const iCalibre = H2.indexOf("Calibre");
    const iValeur = H2.indexOf("Valeur");
    if (iCalibre < 0 || iValeur < 0) throw "Colonnes manquantes dans T2";
    let calibreConducteur = "non trouvé";
    for (let i = 1; i < L2.length; i++) {
      const c = parseCSVLine(L2[i]);
      const valeur = parseFloat(c[iValeur]);
      if (!isNaN(valeur) && valeur >= FLAcorrige) {
        calibreConducteur = c[iCalibre].replace(/"/g, '') || "non trouvé";
        break;
      }
    }
    // 7. Trouver le calibre de la CDM dans T16A
    const t16Atxt = await (await fetch("Calcul%20moteur_csv/T16A.csv")).text();
    const L16A = t16Atxt.split("\n").filter(l => l.trim());
    const H16A = parseCSVLine(L16A[0]);
    const iConducteur = H16A.indexOf("conducteur"), iCDM = H16A.indexOf("cdm");
    if (iConducteur < 0 || iCDM < 0) throw "Colonnes manquantes dans T16A";
    let calibreCDM = "non trouvé";
    for (let i = 1; i < L16A.length; i++) {
      const c = parseCSVLine(L16A[i]);
      if (c[iConducteur] === calibreConducteur) {
        calibreCDM = c[iCDM] || "non trouvé";
        break;
      }
    }
    // 8. Calcul de la section des câbles avec T10A
    const t10Atxt = await (await fetch("Calcul%20moteur_csv/T10A.csv")).text();
    const L10A = t10Atxt.split("\n").filter(l => l.trim());
    const H10A = parseCSVLine(L10A[0]);
    const iCalibreT10A = H10A.indexOf("Clibre");
    const iTypeConducteur = H10A.indexOf(d.typeConducteur);
    if (iCalibreT10A < 0 || iTypeConducteur < 0) throw `Colonne "${d.typeConducteur}" introuvable dans T10A`;
    let sectionConducteur = 0, sectionCDM = 0;
    for (let i = 1; i < L10A.length; i++) {
      const c = parseCSVLine(L10A[i]);
      if (c[iCalibreT10A] === calibreConducteur) {
        sectionConducteur = parseFloat(c[iTypeConducteur]) || 0;
      }
      if (c[iCalibreT10A] === calibreCDM) {
        sectionCDM = parseFloat(c[iTypeConducteur]) || 0;
      }
    }
    // Calcul des sections séparées pour EMT, PVC et Flexible
    const sectionCablesEMT = (sectionConducteur * d.nbConducteursEMT) + sectionCDM;
    const sectionCablesPVC = (sectionConducteur * d.nbConducteursPVC) + sectionCDM;
    const sectionCablesFlex = (sectionConducteur * d.nbConducteursFlex) + sectionCDM;
    // 9. Trouver la grosseur de conduit EMT avec T9I.csv
    const t9Itxt = await (await fetch("Calcul%20moteur_csv/T9I.csv")).text();
    const L9I = t9Itxt.split("\n").filter(l => l.trim());
    const H9I = parseCSVLine(L9I[0]);
    const iGrosseurEMT = H9I.indexOf("Grosseur du conduit");
    const iColonneAEMT = H9I.indexOf("Colonne A");
    const iColonneBEMT = H9I.indexOf("Colonne B");
    if (iGrosseurEMT < 0 || iColonneAEMT < 0 || iColonneBEMT < 0) throw "Colonnes manquantes dans T9I";
    let grosseurConduitEMT = "non trouvé";
    const colonneRechercheEMT = d.nbConducteursEMT > 2 ? iColonneAEMT : iColonneBEMT;
    for (let i = 1; i < L9I.length; i++) {
      const c = parseCSVLine(L9I[i]);
      const valeurColonne = parseFloat(c[colonneRechercheEMT]);
      if (!isNaN(valeurColonne)) {
        if (valeurColonne >= sectionCablesEMT) {
          grosseurConduitEMT = c[iGrosseurEMT] || "non trouvé";
          break;
        }
      }
    }
    // 10. Trouver la grosseur de conduit PVC avec T9C.csv
    const t9Ctxt = await (await fetch("Calcul%20moteur_csv/T9C.csv")).text();
    const L9C = t9Ctxt.split("\n").filter(l => l.trim());
    const H9C = parseCSVLine(L9C[0]);
    const iGrosseurPVC = H9C.indexOf("Grosseur du conduit");
    const iColonneAPVC = H9C.indexOf("Colonne A");
    const iColonneBPVC = H9C.indexOf("Colonne B");
    if (iGrosseurPVC < 0 || iColonneAPVC < 0 || iColonneBPVC < 0) throw "Colonnes manquantes dans T9C";
    let grosseurConduitPVC = "non trouvé";
    const colonneRecherchePVC = d.nbConducteursPVC > 2 ? iColonneAPVC : iColonneBPVC;
    for (let i = 1; i < L9C.length; i++) {
      const c = parseCSVLine(L9C[i]);
      const valeurColonne = parseFloat(c[colonneRecherchePVC]);
      if (!isNaN(valeurColonne)) {
        if (valeurColonne >= sectionCablesPVC) {
          grosseurConduitPVC = c[iGrosseurPVC] || "non trouvé";
          break;
        }
      }
    }
    // 11. Trouver la grosseur de conduit non métallique flexible avec T9H.csv
    const t9Htxt = await (await fetch("Calcul%20moteur_csv/T9H.csv")).text();
    const L9H = t9Htxt.split("\n").filter(l => l.trim());
    const H9H = parseCSVLine(L9H[0]);
    const iGrosseurFlex = H9H.indexOf("Grosseur du conduit");
    const iColonneAFlex = H9H.indexOf("Colonne A");
    const iColonneBFlex = H9H.indexOf("Colonne B");
    if (iGrosseurFlex < 0 || iColonneAFlex < 0 || iColonneBFlex < 0) throw "Colonnes manquantes dans T9H";
    let grosseurConduitFlex = "non trouvé";
    const colonneRechercheFlex = d.nbConducteursFlex > 2 ? iColonneAFlex : iColonneBFlex;
    for (let i = 1; i < L9H.length; i++) {
      const c = parseCSVLine(L9H[i]);
      const valeurColonne = parseFloat(c[colonneRechercheFlex]);
      if (!isNaN(valeurColonne)) {
        if (valeurColonne >= sectionCablesFlex) {
          grosseurConduitFlex = c[iGrosseurFlex] || "non trouvé";
          break;
        }
      }
    }
    // Affichage final simplifié
    document.getElementById("resultat").innerHTML = `
      <strong>FLA :</strong> ${FLA.toFixed(2)} A<br>
      <strong>Calibre des conducteurs :</strong> ${calibreConducteur}<br>
      <strong>Calibre de la CDM :</strong> ${calibreCDM}<br>
      <strong>Fusible temporisé :</strong> ${fusibleT} A<br>
      <strong>Fusible non temporisé :</strong> ${fusibleNT} A<br>
      <strong>Sectionneur temporisé :</strong> ${secT}<br>
      <strong>Sectionneur non temporisé :</strong> ${secNT}<br>
      <strong>Relais de surcharge :</strong> ${relais.toFixed(2)} A<br>
      <strong>Section des câbles EMT :</strong> ${sectionCablesEMT.toFixed(2)} mm²<br>
      <strong>Section des câbles PVC :</strong> ${sectionCablesPVC.toFixed(2)} mm²<br>
      <strong>Section des câbles Flex :</strong> ${sectionCablesFlex.toFixed(2)} mm²<br>
      <strong>Grosseur de conduit EMT :</strong> ${grosseurConduitEMT} mm<br>
      <strong>Grosseur de conduit PVC :</strong> ${grosseurConduitPVC} mm<br>
      <strong>Grosseur de conduit non métallique flexible :</strong> ${grosseurConduitFlex} mm
    `;
    document.getElementById("resultat").style.display = "block";
  } catch (err) {
    document.getElementById("errorMessage").textContent = err;
    document.getElementById("resultat").style.display = "none";
  }
});

// Ajout du calcul dynamique
const formInputs = document.querySelectorAll('#formulaire input, #formulaire select');
formInputs.forEach(input => {
  input.addEventListener('change', () => {
    document.getElementById('formulaire').dispatchEvent(new Event('submit'));
  });
});
</script>
</body>
</html># motor-calculator
Calculator Motor AC/DC FLA
