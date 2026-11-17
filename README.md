# ClairBrowser 2.0 — navigateur Android minimaliste (GeckoView + uBlock Origin)

Page blanche, une barre de recherche, aucune publicité, aucune collecte de données, aucune IA.
Moteur : **GeckoView** (le moteur de Firefox). Blocage : **uBlock Origin**, le vrai, embarqué dans l'application.

> Testé sur Android 16 (arm64) avec GeckoView 159.

## Mise en route (3 étapes)

1. **GeckoView — choisir la version.**
   Ouvrez <https://maven.mozilla.org/maven2/org/mozilla/geckoview/> et repérez l'artefact et la dernière
   version disponibles. Modifiez `geckoviewChannel` et `geckoviewVersion` en tête de `app/build.gradle.kts`.
   Le projet est préréglé sur `geckoview-arm64-v8a` version `159.0.20261022150525` (vue sur le dépôt Mozilla).
   Appareils 64 bits uniquement ; pour un émulateur x86_64, utilisez `x86_64` dans `app/build.gradle.kts`.
2. **uBlock Origin — l'embarquer.** Sur votre ordinateur, à la racine du projet :
   ```
   # Windows (PowerShell)
   powershell -ExecutionPolicy Bypass -File .\fetch-ublock.ps1 1.76.2
   # Linux / macOS
   ./fetch-ublock.sh 1.76.2
   ```
   Remplacez `1.76.2` par la dernière version stable de github.com/gorhill/uBlock/releases
   (ignorez les versions « préliminaires » comme `1.76.3b0`).
   Le script télécharge le `.xpi` signé par Mozilla et le décompresse dans `app/src/main/assets/ublock/`.
   Sans cette étape, l'application fonctionne mais **ne bloque aucune publicité** (un message vous le dit).
3. **Compiler.** Ouvrir le dossier dans Android Studio, laisser Gradle se synchroniser, ▶ Run (Android 10 minimum).

## Ce que fait chaque couche

| Protection | Mécanisme |
|---|---|
| Publicités, traceurs, encarts, anti-contournement | uBlock Origin (listes uBO, EasyList, EasyPrivacy, etc.) |
| Traceurs et cookies tiers | Protection renforcée de Firefox (ETP « stricte ») + cookies tiers bloqués |
| Pop-ups / nouvelles fenêtres | Jamais de fenêtre créée. Un vrai clic sur un lien « nouvel onglet » s'ouvre dans l'onglet courant ; sans clic = refusé |
| Redirections automatiques | Navigation vers un autre site sans geste de l'utilisateur = refusée |
| Ouverture d'applications | `intent:`, `market:`… toujours refusés |
| Permissions de sites | Localisation, caméra, micro, notifications : toujours refusées |
| HTTPS | Connexions non chiffrées refusées (réglable) |

uBlock Origin se règle comme sur ordinateur : menu ⋮ › **uBlock Origin** (popup avec le compteur et
l'interrupteur par site), puis l'icône des paramètres pour le tableau de bord (listes de filtres, règles).

## Vie privée

- Permissions : `INTERNET` et `ACCESS_NETWORK_STATE`. Rien d'autre.
- Aucune télémétrie ni rapport de plantage (Gecko sans service Mozilla), pas de Safe Browsing Google.
- En-tête `Sec-GPC` activé, pas d'historique, pas de favoris, pas de sauvegarde cloud, pas d'autoremplissage.
- `about:config` et le débogage distant sont désactivés.
- Les mises à jour des listes de filtres sont faites par uBlock Origin lui-même (requêtes vers les
  serveurs de listes, sans compte ni identifiant). Pour les couper : tableau de bord uBO › Listes de filtres.

## Limites à connaître

- **Taille** : l'APK fait plusieurs dizaines de Mo et l'application consomme plus de mémoire qu'un WebView.
- **Mises à jour de sécurité** : Gecko est embarqué ; il faut **recompiler avec la dernière version de GeckoView**
  régulièrement. Un navigateur resté sur une vieille version est dangereux.
- **Licence** : uBlock Origin est sous GPLv3. Distribuer l'application l'embarquant impose de publier le code source
  de Clair sous une licence compatible.
- **Pas d'onglets** (c'est le principe) ; certaines connexions « Se connecter avec… » par pop-up peuvent échouer.
  Réglage : Paramètres › désactiver « Bloquer pop-ups ».
- **Téléchargements** : enregistrés dans Téléchargements via MediaStore.
- Le popup de uBlock Origin est affiché dans une fenêtre superposée ; c'est la partie la plus susceptible de
  demander un ajustement selon la version de GeckoView (voir `ExtensionManager.kt` et `MainActivity.showPopup`).

## Dépannage (pièges déjà rencontrés)

- **L'application se ferme seule après quelques secondes** : le moteur ne doit être créé que dans le processus
  principal. `ClairApp.onCreate()` commence par `if (getProcessName() != packageName) return` ; ne pas l'enlever
  (Android exécute `onCreate()` aussi dans les processus enfants de Gecko).
- **« uBlock Origin n'est pas installé » / erreur `Addon must include an id`** : le dossier `assets/ublock/` doit
  contenir la version **Firefox** (`uBlock0_<version>.firefox.signed.xpi`), pas la version Chrome/Chromium.
  Test rapide : `manifest.json` contient `browser_specific_settings` et la permission `menus` (pas `contextMenus`).
  Le plus sûr est de passer par `fetch-ublock.ps1` / `fetch-ublock.sh`, qui téléchargent la bonne archive.
- **Popup de uBlock blanc** : en général l'extension n'a pas démarré (mauvaise version, voir ci-dessus). Les erreurs
  JavaScript des extensions sont visibles dans Logcat avec le filtre `tag:GeckoConsole`
  (`consoleOutput(true)` dans `ClairApp.kt` ; à retirer pour une version finale).
- **Popup qui clignote** : `ExtensionManager.openPopup()` réutilise le popup déjà ouvert au lieu d'en recréer un.
- Les erreurs `Native manifests are not supported on android` et `Unknown localization message` dans Logcat sont
  sans conséquence.

## Structure

```
app/src/main/java/app/clair/browser/
  ClairApp.kt          GeckoRuntime (un seul par processus) + protections de base
  MainActivity.kt      navigateur, barre d'adresse, menu, dialogues, popup uBO
  ExtensionManager.kt  installation de uBlock Origin (extension intégrée)
  SettingsActivity.kt  écran des paramètres
  Prefs.kt, DataCleaner.kt
app/src/main/assets/ublock/   (rempli par fetch-ublock.ps1 / fetch-ublock.sh)
fetch-ublock.sh
```
