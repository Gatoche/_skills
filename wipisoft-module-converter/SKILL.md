---
name: wipisoft-module-converter
description: Transforme une app C#/.NET 8 (WPF ou console) en Module wipiSoft (UI embedded dans un slot host) ou en ModuleService wipiSoft (app headless invocable par IPC) selon les conventions du SDK partagé dans `_SDKs/Modules/Wps.Module.Sdk` et `_SDKs/Modules/Wps.ModuleService.Sdk`. Utilise systématiquement ce skill quand l'utilisateur veut "transformer cette app en Module wipiSoft", "transformer en ModuleService", "convertir en module wipiSoft", "intégrer cette app au host wipiSoft", "ajouter le SDK Module/ModuleService", "wipiSoft-iser une app", ou toute formulation équivalente impliquant l'intégration d'une app au framework wipiSoft (Module/ModuleService/host ModuloSlot/wipiTools). Ce skill applique aussi les conventions csproj `<WipiModuleKind>`, `<WipiModuleName>`, `<WipiModuleVersion>`, `<LGO>`, le bootstrap dual-mode standalone/embedded, l'enregistrement des handlers Invoke, et la création du Description.md embarqué — donc déclenche aussi sur des demandes plus ciblées comme "ajoute le bootstrap WpsModule" ou "expose une méthode Invoke depuis cette app".
---

# WipiSoft Module Converter

Ce skill transforme une app C#/.NET 8 existante en **Module** ou **ModuleService** wipiSoft.
Le résultat : un csproj conforme aux conventions du SDK, un `App.xaml.cs` qui appelle le
bootstrap du SDK avec branchement standalone/embedded, et un `Description.md` embedded
lu par le host.

## Source de vérité

Les deux HOWTO du SDK définissent la procédure exacte (conventions, patterns, pièges) :

- **Module** (UI WPF embedded) : `_SDKs/Modules/Wps.Module.Sdk/HOWTO.md`
- **ModuleService** (headless invocable) : `_SDKs/Modules/Wps.ModuleService.Sdk/HOWTO.md`

**Toujours lire le HOWTO pertinent au début**, pas la mémoire — si les HOWTO ont évolué
depuis le training, c'est eux qui font foi.

Pattern de référence éprouvé (ModuleService) : `wipiSoft/TracePML-Project/TracePML/App.xaml.cs`
et son `TracePML.csproj`. À consulter pour comprendre le branchement dual-mode complet.

## Workflow

### Étape 1 — Identifier l'app cible et son type

L'utilisateur indique soit explicitement (`Module` ou `ModuleService`), soit implicitement.
Si non spécifié, applique cette heuristique sur le csproj :

| Signal | Type suggéré |
|---|---|
| `OutputType=WinExe` + `UseWPF=true` + présence d'une `MainWindow.xaml` avec UI principale | **Module** |
| `OutputType=Exe` (console) | **ModuleService** |
| `OutputType=WinExe` + `UseWPF=true` mais l'app vit en arrière-plan (systray, FileSystemWatcher, daemon) avec `MainWindow` jouant le rôle de "settings/debug uniquement" | **ModuleService** |

Demande confirmation à l'utilisateur si le choix n'est pas évident. Le mauvais choix
mène à un mauvais bootstrap et l'app ne s'intégrera pas correctement au host.

### Étape 2 — Recueillir les valeurs csproj

Demande (ou lis depuis le csproj si déjà présents) :

- **WipiModuleName** : nom court, sans espace, sans `\ / : * ? " < > |` — utilisé en CLI
  `--show NomCourt` et comme nom du dossier de déploiement
  `C:\_wipiSoft\Apps\Modules\<WipiModuleName>\`. Défaut suggéré : nom du projet csproj.
- **WipiModuleVersion** : version applicative. Défaut : `1.0`.
- **LGO** (optionnel) : Logiciel de Gestion d'Officine compatible(s), séparateur `;` pour
  plusieurs valeurs (ex: `Winpharma;LGPI`). Demande à l'utilisateur s'il veut le préciser.
- **Company** : si absent, propose `wipiSoft` (recommandé pour la détection wpsServices).

### Étape 3 — Modifier le csproj

Lire le csproj actuel, puis :

1. **PropertyGroup** : ajouter ou mettre à jour les tags
   ```xml
   <WipiModuleKind>Module</WipiModuleKind>      <!-- ou ModuleService -->
   <WipiModuleName>NomCourt</WipiModuleName>
   <WipiModuleVersion>1.0</WipiModuleVersion>
   <LGO>Winpharma</LGO>                         <!-- si défini -->
   <Company>wipiSoft</Company>                  <!-- si absent -->
   ```

2. **OutputType** : pour ModuleService, forcer à `WinExe` (silencieux, pas de console au
   démarrage). Si l'app a `Exe` actuellement, change-le en `WinExe` avec un commentaire
   expliquant pourquoi (cf. HOWTO ModuleService section "csproj").

3. **ItemGroup ProjectReference** : ajouter
   ```xml
   <ProjectReference Include="<chemin-relatif>\_SDKs\Modules\Wps.<Kind>.Sdk\Wps.<Kind>.Sdk.csproj" />
   ```
   Le `<chemin-relatif>` dépend de la profondeur du projet par rapport à `dev/`. Calculer
   à partir du chemin absolu du csproj. Exemple typique : `..\..\..\` (depuis
   `wipiSoft\<MonProjet>\<MonApp>\<MonApp>.csproj`, soit 3 niveaux jusqu'à `dev/`, puis
   `_SDKs\Modules\…`).

4. **ItemGroup EmbeddedResource** : ajouter
   ```xml
   <EmbeddedResource Include="Description.md" />
   ```

5. **Import targets** : à la fin du Project (avant `</Project>`)
   ```xml
   <Import Project="<chemin-relatif>\_SDKs\Modules\Wps.<Kind>.Sdk\Wps.<Kind>.Sdk.targets" />
   ```

6. **Nettoyage** : retirer les `<Compile Include="..\WpsDebugSender.cs">` redondants
   (fournis transitivement par `Wps.Module.Sdk.Core` via la chaîne de référence). Garder
   les autres helpers source-linked qui ne sont pas dans Core (`WpsXamlAdjust`,
   `WpsHKCU`, etc.).

### Étape 4 — Créer ou mettre à jour `Description.md`

Si absent, créer à la racine du projet (au même niveau que le csproj) avec ce template :

```markdown
# <NomCourt>

<Phrase d'intro : ce que fait l'app, à qui elle s'adresse>

## Fonctionnement

<Description du flow principal — ce que l'app surveille, traite, expose>

## Méthodes exposées (ModuleService uniquement)

### `MethodName(params) → result`

<Description de la méthode, paramètres, retour>

## Persistance

<Si l'app utilise HKCU ou autre stockage local>
HKCU\Software\wipiSoft\<NomCourt>\... (clés et valeurs)
```

Pour un Module (UI embedded), la section "Méthodes exposées" est inutile — la retirer.

Si `Description.md` existe déjà, ne pas écraser — laisser tel quel ou suggérer
amélioration si trop minimal.

### Étape 5 — Modifier `App.xaml.cs`

#### Pour un Module

Pattern minimal (cf. HOWTO Module section 3) :

```csharp
using Wps.Module;

protected override void OnStartup(StartupEventArgs e)
{
    base.OnStartup(e);
    var window = new MainWindow();
    WpsModule.Bootstrap(this, window, e.Args);
    window.Show();
}
```

Et dans `MainWindow.xaml.cs` (constructeur), ajouter le `NotifyReadyAsync` au moment
adapté :
- WPF basique : `Loaded += async (_, _) => await WpsModule.NotifyReadyAsync();`
- WebView2 : après `EnsureCoreWebView2Async`
- CefSharp : dans `FrameLoadEnd` avec `e.Frame.IsMain`

⚠️ Le moment du Ready dépend du module — **demander à l'utilisateur** quel est le
moment "métier-prêt" si pas évident.

#### Pour un ModuleService (dual-mode)

Pattern de référence : copier le squelette de
`wipiSoft/TracePML-Project/TracePML/App.xaml.cs` et l'adapter à l'app cible :

1. **Préserver tout le code métier existant** (services, ViewModel, FileSystemWatcher,
   systray, etc.). Ne pas en virer.
2. **Encadrer** ce code par le branchement embedded/standalone :
   - `await WpsModuleService.BootstrapAsync(args)` au début de `InitAsync`
   - Mutex singleton conditionnel `if (!WpsModuleService.IsEmbedded)`
   - Si `IsEmbedded` : `RegisterSettingsWindow` + `RegisterInvokeHandler<...>` + `NotifyReadyAsync` + `RunAsync` fire-and-forget
   - Si `!IsEmbedded` : code standalone (systray, fenêtre debug si flag, etc.) — code historique de l'app
3. **MainWindow.Closing** : intercepter pour Hide au lieu de Close (la window est
   réutilisable, le service continue de tourner).

Pour les méthodes Invoke à exposer : si l'utilisateur n'en demande pas explicitement, en
proposer 2-3 pertinentes selon ce que fait l'app (ex: `GetStatus`, `GetLastEvent`, etc.)
ou laisser pour plus tard avec un commentaire `// TODO: RegisterInvokeHandler<...>`.

#### (v1.3) Hooks shutdown négocié — `IWpsModule`

Depuis le contrat **v1.3** (juin 2026), proposer **systématiquement** au moins
`OnCanCloseRequestedAsync` quand l'app a un cleanup applicatif au shutdown (flush, ABM_REMOVE,
sauvegarde, fermeture d'une connexion BDD, etc.). Le pattern recommandé est d'exécuter le
cleanup **proactivement** avant de répondre `Ok` :

```csharp
public partial class App : Application, IWpsModule
{
    public ValueTask<CanCloseDecision> OnCanCloseRequestedAsync(CanCloseContext ctx)
    {
        // Cleanup synchrone rapide ICI (avant de répondre Ok au host)
        MyCleanup.RunSync();
        return new ValueTask<CanCloseDecision>(CanCloseDecision.Ok);
    }

    public void OnShutdownRequested() => Application.Current?.Shutdown();

    public void OnHostDisconnected(HostDisconnectReason reason)
    {
        MyCleanup.RunSync();  // idempotent
        Application.Current?.Shutdown();
    }
}
```

Pour les cleanups longs (> 100ms) : retourner `CanCloseDecision.Busy("Description...", estimatedMs)`,
exécuter le cleanup async fire-and-forget, envoyer `WpsModule.ReportBusyProgress(...)` toutes
les ~3s pendant le travail, puis `WpsModule.ResolveCanClose(CanCloseDecision.Ok)` à la fin.

Pour les dialogs utilisateurs (genre "Voulez-vous sauvegarder ?") : retourner
`CanCloseDecision.NeedUser("...")`. Le host bascule l'onglet et donne le focus, l'app affiche
son dialog, puis appelle `ResolveCanClose(Ok)` ou `ResolveCanClose(Rejected("annulé"))` selon
la décision de l'utilisateur. Si Rejected, le host annule sa fermeture en cascade.

Ne PAS oublier `WpsModule.Register(this)` (Module) ou `WpsModuleService.Register(this)`
(ModuleService) au début du Bootstrap — sans ça les hooks ne sont pas appelés et le SDK
utilise les DIMs (Ok par défaut, comportement v1.2).

### Étape 6 — Build

Lancer `dotnet build` sur le csproj pour valider :

```bash
dotnet build "<chemin-csproj>" -c Debug -p:Platform=x64 -nologo -v:minimal
```

Si erreurs : analyser et fixer (oubli d'un using, conflit CS0436 sur un Compile Include
redondant, mauvais chemin relatif vers `_SDKs/Modules/`, etc.).

### Étape 7 — Reporter à l'utilisateur

Résumer :
- Fichiers créés (Description.md si nouveau)
- Fichiers modifiés (csproj, App.xaml.cs, MainWindow.xaml.cs si Ready ajouté)
- Conventions appliquées (WipiModuleKind, WipiModuleName, etc.)
- Build status (OK/erreurs)
- **Points à vérifier manuellement** :
  - Pour Module : le moment précis du `NotifyReadyAsync` (peut nécessiter ajustement si
    l'app a une init asynchrone non standard)
  - Pour ModuleService : les méthodes Invoke à exposer (à compléter selon le métier)
  - Le `Description.md` (à enrichir si template trop générique)

## Pièges courants

- **Mauvais chemin relatif vers `_SDKs/Modules/`** : la profondeur varie selon l'arborescence
  du projet. Calculer à partir du chemin absolu du csproj. Si l'app est dans
  `dev/wipiSoft/MonApp/MonApp.csproj`, le chemin est `..\..\_SDKs\Modules\`. Si elle est dans
  `dev/wipiSoft/MonProject/MonApp/MonApp.csproj`, c'est `..\..\..\_SDKs\Modules\`.

- **(v1.3) Cleanup applicatif laissé dans `OnShutdownRequested`** quand il dure > 100ms : à
  déplacer proactivement dans `OnCanCloseRequestedAsync` (avant le `return Ok`) pour bénéficier
  de la fenêtre de négociation du host. Sinon le grace period 7s côté SDK Host peut shooter
  le cleanup applicatif si la machine swappe ou que d'autres modules ralentissent la cascade.

- **(v1.3) Oubli du `Register(this)`** au début du Bootstrap : les hooks ne sont pas appelés,
  on retombe sur le comportement v1.2 (CLOSE direct, pas de phase CAN_CLOSE). Symptôme : le
  cleanup applicatif n'est pas proactif.

- **Mutex singleton oublié dans le branchement standalone** : si l'app avait un mutex,
  le préserver mais le rendre conditionnel à `!WpsModuleService.IsEmbedded`. Sinon le
  host ne peut pas relancer le service après un Stop.

- **`Application.Shutdown()` à la fermeture de la MainWindow** : tue le process dès que
  l'utilisateur ferme la window, alors que le service doit continuer en arrière-plan.
  Toujours intercepter `Closing` → `e.Cancel = true; w.Hide()` pour ModuleService.

- **`OutputType=Exe` au lieu de `WinExe` pour ModuleService** : fenêtre console noire
  visible au démarrage en mode embedded. Toujours `WinExe`.

- **`<Compile Include="...WpsDebugSender.cs">` redondant** : conflit CS0436 (type
  dupliqué). `Wps.Module.Sdk.Core` (dépendance transitive) le fournit déjà.

## Limites du skill

Ce skill **ne fait pas** :
- Conception de l'architecture métier (le code existant est préservé)
- Création des méthodes Invoke spécifiques au domaine (à proposer ou laisser TODO)
- Migration des Settings HKCU vers une autre clé (le préfixe HostName de
  `WpsServiceDaemonConfig` est géré côté host, pas côté service)
- Tests fonctionnels embedded (lancement via host à faire manuellement par l'utilisateur)
