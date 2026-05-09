---
name: wipisoft-app-converter
description: Transforme une app C#/.NET 8 (WPF, WinForms, console) en app standalone wipiSoft selon les conventions du SDK partagé dans `_SDKs/Apps/Wps.StandaloneApp.Sdk` + `_SDKs/Apps/Wps.AppLauncher`. Utilise systématiquement ce skill quand l'utilisateur veut "transformer cette app en app standalone wipiSoft", "ajouter le launcher AOT devant l'AppHost", "ajouter la vérification d'intégrité du déploiement", "intégrer une app standalone au framework wipiSoft", "wipiSoft-iser une app standalone", ou toute formulation équivalente impliquant l'intégration d'une app lancée directement par l'utilisateur (raccourci, double-clic, file association) au framework wipiSoft. Ce skill applique aussi les conventions csproj `<WipiAppKind>`, `<WipiAppName>`, `<WipiAppVersion>`, le source-link des helpers `WpsAppGuard` / `WpsAppUserModelId` / `WpsForegroundBringer`, et la mise en place de la single-instance canonique (mutex `wipiSoft.<App>.SingleInstance` + pipe `wipiSoft.<App>.Pipe`) — donc déclenche aussi sur des demandes plus ciblées comme "ajoute le guard launcher à cette app" ou "aligne le single-instance sur la convention wipiSoft". Distinct de `wipisoft-module-converter` qui cible les apps lancées par un host (Module/ModuleService) — utiliser celui-ci uniquement pour les apps lancées directement par l'utilisateur.
---

# WipiSoft App Converter

Ce skill transforme une app C#/.NET 8 existante en **app standalone wipiSoft**.

Une app standalone wipiSoft est une app classique (WPF, WinForms, console) lancée
**directement par l'utilisateur** (raccourci, double-clic, file association,
drag-drop) — par opposition aux Modules/ModuleServices pilotés par un host comme
ModuloSlot. Exemples : OuiPDF, MailTools.

Le résultat de la conversion :

- Un **launcher natif (NativeAOT)** `<App>.exe` devant l'AppHost réel
  `<App>.app.exe`. Le launcher vérifie l'intégrité du déploiement (manifest +
  master guid SHA-256) avant de démarrer l'app — plus jamais le dialogue runtime
  cryptique *"Install .NET Desktop Runtime"* en cas de fichier manquant.
- Un **AUMID canonique** partagé entre launcher et AppHost (`wipiSoft.<App>`)
  pour qu'ils ne forment qu'une seule icône taskbar.
- Un **garde-fou** anti-lancement direct de `<App>.app.exe` (`WpsAppGuard`).
- Une **single-instance smart** sur les noms canoniques `wipiSoft.<App>.*`
  permettant au launcher de transmettre des arguments à une instance déjà en
  cours sans spawner un second AppHost.
- Un **manifest d'intégrité** (`<App>.deploy.manifest.json` + `.deploy.guid`)
  généré au build, lu par le launcher au démarrage.

## Source de vérité

Le HOWTO du SDK définit la procédure exacte (conventions, patterns, pièges) :

- `_SDKs/Apps/Wps.StandaloneApp.Sdk/HOWTO.md`

**Toujours lire le HOWTO au début**, pas la mémoire — si le HOWTO a évolué
depuis le training, c'est lui qui fait foi.

Pattern de référence éprouvé : `wipiSoft/OuiPDF-Project/OuiPDF/App.xaml.cs`,
son `OuiPDF.csproj`, et son `PdfViewer.cs` (gestion de WebView2 redirigé
vers `%LOCALAPPDATA%`). À consulter pour comprendre l'intégration complète.

## Workflow

### Étape 1 — Vérifier que c'est bien une app standalone (pas un Module)

L'utilisateur indique soit explicitement qu'il veut une app standalone, soit
implicitement par le contexte. Si non spécifié, applique cette heuristique :

| Signal | SDK à utiliser |
|---|---|
| App lancée par l'utilisateur (raccourci, file association, drag-drop, double-clic) | **Wps.StandaloneApp.Sdk** (ce skill) |
| App lancée par un host wipiSoft (ModuloSlot, futur wipiTools) | `Wps.Module.Sdk` ou `Wps.ModuleService.Sdk` (autre skill : `wipisoft-module-converter`) |

Si l'app a vocation à être **les deux** (rare), demander à l'utilisateur
laquelle prioriser (le SDK Apps et le SDK Modules sont mutuellement exclusifs
sur la même app — ils utilisent des conventions csproj différentes).

### Étape 2 — Pré-requis : le launcher AOT doit être publié

Vérifier la présence de :

```
_SDKs/Apps/Wps.AppLauncher/published/win-x64/Wps.AppLauncher.exe
```

Si absent, le publier :

```
dotnet publish _SDKs/Apps/Wps.AppLauncher/Wps.AppLauncher.csproj `
    -c Release -r win-x64 -o _SDKs/Apps/Wps.AppLauncher/published/win-x64 `
    -p:_hostArchitecture=x64
```

Le `-p:_hostArchitecture=x64` est **nécessaire sur Windows on ARM** (la machine
de dev de Gatoche) — sinon `Microsoft.NETCore.Native.Publish.targets` exige le
package ARM64 ILCompiler. Sur une machine x64 vraie, le flag est no-op.

Note : `vswhere.exe` doit aussi être joignable au moment du publish AOT (depuis
`C:\Program Files (x86)\Microsoft Visual Studio\Installer\`). Si pas dans le
PATH, l'ajouter à la session courante.

### Étape 3 — Auditer la **règle d'or** : aucune écriture dans le DeployDir au runtime

⚠️ **Étape critique**. Le manifest d'intégrité fige les SHA-256 au build. Si
l'app écrit un fichier dans son dossier d'install au runtime, le launcher dira
*"Fichier corrompu"* au prochain démarrage et refusera de la lancer.

Avant tout, **scanner le code** pour identifier les écritures dans le DeployDir :

| À chercher | Action |
|---|---|
| `CoreWebView2Environment.CreateAsync` sans `userDataFolder` ou `EnsureCoreWebView2Async` sans env | Rediriger explicitement vers `%LOCALAPPDATA%\<App>\WebView2\` (cf. HOWTO §1 piège 1) |
| Extraction d'archive (`ZipFile.ExtractToDirectory`, etc.) avec `AppDomain.CurrentDomain.BaseDirectory` | Rediriger l'extraction vers `%LOCALAPPDATA%\<App>\<sub>\` |
| `File.WriteAllText` / `WriteAllBytes` dans `BaseDirectory` ou en relatif | Rediriger vers `%LOCALAPPDATA%\<App>\` |
| Fichiers `*WindowStateSettings.json` ou similaires à côté de l'exe | Déplacer vers `%LOCALAPPDATA%\<App>\` ou `HKCU\Software\wipiSoft\<App>\` |

Patcher chaque cas trouvé **avant** d'aller plus loin. Sinon, l'app crashera
au premier démarrage post-conversion. Le HOWTO §règle d'or fournit du code
prêt à l'emploi pour le cas WebView2.

### Étape 4 — Recueillir les valeurs csproj

Demander (ou lire depuis le csproj si déjà présents) :

- **`WipiAppName`** : nom court, sans espace, sans `\ / : * ? " < > |`. Sert
  comme `<AppName>.exe`, comme dossier dans `%LOCALAPPDATA%`, et dans les noms
  canoniques `wipiSoft.<AppName>.*` (mutex, pipe, AUMID). Défaut suggéré : nom
  du projet csproj.
- **`WipiAppVersion`** : version applicative. Défaut : `1.0`. Distincte de
  `Version`/`AssemblyVersion` (qui peuvent suivre une logique propre).
- **`Company`** : si absent, proposer `wipiSoft` (recommandé pour la détection
  via PE version info — cf. wpsServices, DeploymentContext).

### Étape 5 — Modifier le csproj

Lire le csproj actuel, puis :

1. **PropertyGroup** : ajouter
   ```xml
   <WipiAppKind>Standalone</WipiAppKind>
   <WipiAppName>NomCourt</WipiAppName>
   <WipiAppVersion>1.0</WipiAppVersion>
   <Company>wipiSoft</Company>      <!-- si absent -->
   ```

2. **OutputType** : doit être `WinExe` (WPF/WinForms) ou `Exe` (console). Pas
   de changement à faire si déjà l'un des deux.

3. **`<ApplicationIcon>`** : **doit pointer sur un `.ico` existant** à la racine
   du projet. Sinon le launcher copié restera avec l'icône par défaut
   (`Wps.AppLauncher.ico`). Si l'app n'a pas d'icône, en demander une à
   l'utilisateur (un .ico même basique).

4. **ItemGroup `<Compile Include>`** : source-linker les 3 helpers depuis
   `_libs/`. Calculer le chemin relatif depuis le csproj (typiquement
   `..\..\..\_libs\` si l'app vit dans `dev/<repo>/<sub>/<App>/`) :

   ```xml
   <ItemGroup>
     <Compile Include="..\..\..\_libs\WpsAppGuard.cs"          Link="Helpers\WpsAppGuard.cs" />
     <Compile Include="..\..\..\_libs\WpsAppUserModelId.cs"    Link="Helpers\WpsAppUserModelId.cs" />
     <Compile Include="..\..\..\_libs\WpsForegroundBringer.cs" Link="Helpers\WpsForegroundBringer.cs" />
   </ItemGroup>
   ```

5. **Import targets** : à la fin du Project (avant `</Project>`)
   ```xml
   <Import Project="..\..\..\_SDKs\Apps\Wps.StandaloneApp.Sdk\Wps.StandaloneApp.Sdk.targets" />
   ```

6. **Nettoyage** : retirer les `<Compile Include>` de `WpsDebugSender.cs` s'il
   est déjà source-linké (transitif via Wps.Module.Sdk.Core qui n'est pas
   référencé ici — donc en pratique l'app standalone source-linke
   `WpsDebugSender.cs` directement). Vérifier qu'il n'y a pas de doublon.

### Étape 6 — Modifier `App.xaml.cs` (WPF) ou `Main` (console/WinForms)

Insérer **au tout début** d'`OnStartup` (WPF) ou `Main` (console/WinForms) :

```csharp
protected override void OnStartup(StartupEventArgs e)
{
    // 1) Refus du lancement direct de <App>.app.exe (silent exit si pas de token).
    WpsAppGuard.EnsureLaunchedByLauncherOrExit();

    // 2) AUMID canonique : DOIT etre appele AVANT toute creation de fenetre WPF.
    WpsAppUserModelId.SetCurrentProcess("<AppName>");

    // 3) Patch des shortcuts (.lnk) existants pointant vers <App>.exe pour qu'ils
    //    aient le meme AUMID. Fire-and-forget pour ne pas retarder le startup.
    try
    {
        string ownExe = Environment.ProcessPath ?? string.Empty;
        string? deployDir = Path.GetDirectoryName(ownExe);
        if (!string.IsNullOrEmpty(deployDir))
        {
            string launcherExe = Path.Combine(deployDir, "<AppName>.exe");
            Task.Run(() => WpsAppUserModelId.EnsureAllShortcutsAumid(
                "<AppName>", launcherExe));
        }
    }
    catch { /* best effort */ }

    // 4) Suite normale du demarrage (mutex single-instance, MainWindow, etc.)
    base.OnStartup(e);
    // ...
}
```

⚠️ **L'ordre compte** : guard avant tout, AUMID avant la première fenêtre WPF.

### Étape 7 — Aligner le single-instance sur la convention canonique

Si l'app a déjà une logique single-instance (mutex + named pipe), **réécrire**
les noms aux conventions canoniques attendues par le launcher :

```csharp
private const string MutexName = "wipiSoft.<AppName>.SingleInstance";
private const string PipeName  = "wipiSoft.<AppName>.Pipe";
```

Le launcher attend ces noms exactement (sinon il ne saura pas qu'une instance
est déjà en cours et spawnera un second AppHost qui se shutdown immédiatement).

### Étape 8 — Aligner le protocole pipe sur le multi-args + wake-up

Le launcher envoie **un argument par ligne** sur le pipe (multi-fichiers
drag-drop, file association multi). Côté serveur, lire en boucle jusqu'à EOF :

```csharp
// Server side (StartPipeServer)
using var reader = new StreamReader(pipeServer);
var paths = new List<string>();
string? line;
while ((line = await reader.ReadLineAsync()) is not null)
{
    if (!string.IsNullOrEmpty(line)) paths.Add(line);
}
Application.Current.Dispatcher.Invoke(() =>
{
    BringToForeground();             // TOUJOURS, qu'il y ait des paths ou non
    foreach (var p in paths) OpenFile(p);
});

// Client side (SendArgsToRunningInstance)
// Connexion TOUJOURS faite, meme avec args vide (= signal "wake up").
using var pipeClient = new NamedPipeClientStream(".", PipeName, PipeDirection.Out);
pipeClient.Connect(1000);
using var w = new StreamWriter(pipeClient) { AutoFlush = true };
foreach (var arg in args) if (!string.IsNullOrEmpty(arg)) w.WriteLine(arg);
```

⚠️ **Ne PAS court-circuiter** sur `args.Length == 0` côté client ni sur
`paths.Count == 0` côté serveur. Une connexion vide = signal "wake up" qui doit
déclencher `BringToForeground` pour ramener la fenêtre au premier plan.

### Étape 9 — `BringToForeground` via le helper partagé

Si l'app a un `BringToForeground()` custom (Topmost flip, mouse_event aveugle,
SetForegroundWindow nu...), le **remplacer** par un appel à
`WpsForegroundBringer.BringToForegroundAndClick(window)` :

```csharp
public void BringToForeground()
    => WpsForegroundBringer.BringToForegroundAndClick(window);
```

Le helper fait la combinaison robuste `AttachThreadInput` +
`BringWindowToTop` + `SetForegroundWindow` + `Activate` + `PostMessage
WM_LBUTTONDOWN/UP` (clic synthétique ciblé sur le HWND pour calmer l'alerte
taskbar). Cf. mémoire `_libs/feedback_setforegroundwindow_taskbar_flash.md`.

⚠️ **Effet de bord à anticiper** : le clic synthétique `WM_LBUTTONDOWN` peut
déclencher des handlers WPF (`Window_MouseDown` → `DragMove()` typiquement).
Tout handler `MouseDown` qui appelle `DragMove()` doit **garder** sur
`e.LeftButton == MouseButtonState.Pressed` (sinon `InvalidOperationException`
unhandled → crash silencieux). Auditer les handlers `MouseDown` de l'app.

### Étape 10 — Build & validation

Lancer `dotnet build` Debug + Release pour vérifier :

```bash
dotnet build <App>.csproj -c Debug -p:Platform=x64
dotnet build <App>.csproj -c Release -p:Platform=x64
```

Le post-build :

1. Renomme `<App>.exe` → `<App>.app.exe` (depuis `obj/apphost.exe`)
2. Copie `Wps.AppLauncher.exe` → `<App>.exe`
3. Injecte l'icône `<ApplicationIcon>` dans le launcher (rcedit-x64.exe)
4. Génère `<App>.deploy.manifest.json` + `<App>.deploy.guid`

En Release uniquement : `WpsForceCleanInRelease` supprime `$(TargetDir)` avant
le build pour garantir un manifest 100% reproductible.

Vérifier la sortie :

```
<App>.exe                      ← launcher AOT (~3 Mo) avec icone <App>
<App>.app.exe                  ← AppHost .NET (le vrai)
<App>.dll
<App>.runtimeconfig.json
<App>.deps.json
<App>.deploy.manifest.json     ← integrite (liste fichiers + SHA-256)
<App>.deploy.guid              ← scelle du manifest (32 hex chars)
```

Vérifier l'arch x64 du PE produit (cf. mémoire env_dev_arm64_x64_builds si
machine ARM64) :

```powershell
$bytes = [System.IO.File]::ReadAllBytes("...\<App>.exe")
$peOff = [System.BitConverter]::ToInt32($bytes, 0x3C)
$machine = [System.BitConverter]::ToUInt16($bytes, $peOff + 4)
# 0x8664 = x64 attendu
```

### Étape 11 — Reporter à l'utilisateur

Résumer :

- Fichiers modifiés (csproj, App.xaml.cs / Main, helpers source-linkés)
- Redirections runtime appliquées (WebView2 → %LOCALAPPDATA%, etc.)
- Single-instance migré sur les noms canoniques (mutex/pipe)
- Build status (OK/erreurs)
- **Manifest** : nombre de fichiers protégés (typiquement quelques dizaines
  pour une app simple, jusqu'à ~70-100 si beaucoup de DLLs/resources)
- **Points à vérifier manuellement** :
  - Si l'utilisateur épingle l'app à la taskbar : doit pointer sur
    **`<App>.exe`** (le launcher), PAS `<App>.app.exe` (l'AppHost). Sinon
    contournement du launcher → `WpsAppGuard` exit silencieusement → app
    ne se lance pas. Cf. HOWTO §2.
  - Vérifier qu'aucun handler `MouseDown` n'appelle `DragMove()` sans guarde
    `e.LeftButton == MouseButtonState.Pressed` (cf. étape 9).

## Pièges courants

- **Mauvais chemin relatif vers `_SDKs/Apps/`** : la profondeur varie selon
  l'arborescence du projet. Calculer à partir du chemin absolu du csproj.
  Pour `dev/<repo>/<sub>/<App>/`, le chemin est `..\..\..\_SDKs\Apps\`.

- **Launcher AOT pas publié** : la target échoue avec un message clair
  pointant la commande `dotnet publish` à lancer. Suivre les instructions
  de l'étape 2.

- **`<ApplicationIcon>` absent** : l'icône du launcher copié reste celle par
  défaut de `Wps.AppLauncher`. Demander une `.ico` à l'utilisateur si l'app
  n'en a pas — ou accepter le défaut (à documenter dans le résumé).

- **Single-instance pas migrée** : l'app utilise encore un mutex/pipe legacy
  (`<App>AppMutex` / `<App>Pipe` ou similaire). Le launcher ne sait pas
  qu'une instance tourne déjà → spawn d'un second AppHost qui crée une
  course critique. Aligner sur les noms canoniques `wipiSoft.<App>.*`.

- **Court-circuit `args.Length == 0`** dans le pipe client/serveur : casse le
  signal "wake-up" du double-clic supplémentaire sur `<App>.exe`. La fenêtre
  ne revient pas au foreground. Toujours connecter / toujours
  `BringToForeground`.

- **AUMID posé après création de fenêtre** : la fenêtre garde l'AUMID par
  défaut (path-derived) et ne match pas le shortcut `wipiSoft.<App>` →
  2 icônes taskbar. Toujours `WpsAppUserModelId.SetCurrentProcess(name)` au
  TOUT début d'`OnStartup`, **avant** `base.OnStartup` ou toute création
  de window WPF.

- **Cache WebView2 non redirigé** : ~290 fichiers volatiles
  (`<App>.app.exe.WebView2/EBWebView/...`) entrent dans le manifest au
  premier build post-run, puis le manifest invalide au démarrage suivant
  car les fichiers ont changé. **Toujours rediriger UserDataFolder** vers
  `%LOCALAPPDATA%\<App>\WebView2\`. (cf. mémoire
  `feedback_setforegroundwindow_taskbar_flash.md` n'a rien à voir, mais
  HOWTO §règle d'or piège 1).

- **`DragMove()` non guardé** dans un handler `MouseDown` : le clic
  synthétique de `WpsForegroundBringer` (PostMessage WM_LBUTTONDOWN) déclenche
  le handler avec `LeftButton = Released` → `InvalidOperationException` →
  crash unhandled silencieux. Auditer tous les handlers `MouseDown`.

- **Shortcut taskbar pointant sur `<App>.app.exe`** : épinglage manuel via
  clic-droit sur l'icône d'une fenêtre déjà ouverte (épingle le path du
  process actif = AppHost). Le shortcut contourne le launcher → `WpsAppGuard`
  fait exit silencieusement → impossible de lancer l'app via ce shortcut.
  Toujours épingler **`<App>.exe`** depuis l'explorateur (clic-droit sur
  l'exe lui-même → "Épingler à la barre des tâches").

## Limites du skill

Ce skill **ne fait pas** :

- Conception de l'architecture métier (le code existant est préservé)
- Audit complet de toutes les écritures de fichiers de l'app — il flag les
  patterns courants (WebView2, archives, WriteAllText basé sur BaseDirectory)
  mais peut manquer des cas exotiques. À compléter manuellement après la
  conversion si des "fichier corrompu" apparaissent au démarrage.
- Création de l'icône `.ico` — à fournir par l'utilisateur si absente.
- Setup de l'épinglage taskbar (action utilisateur manuelle, hors scope).
- Mise en place de la "réparation à distance par wipiManager" : c'est une
  vision long terme documentée dans la mémoire
  `_libs/feedback_standalone_apps_repair_via_host.md`. Le launcher actuel
  fait MessageBox + exit en cas de manifest invalide ; la réparation
  automatique reviendra à wipiManager quand cette fonction sera
  implémentée côté host.
