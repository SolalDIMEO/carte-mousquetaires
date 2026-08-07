[README.md](https://github.com/user-attachments/files/30833128/README.md)
# Carte des projets — Réseau Mousquetaires

Carte interactive de France représentant les projets solaires (TI/EPC), avec
mise à jour automatique depuis le fichier Excel SharePoint.

## Contenu du dépôt

```
index.html              → l'outil (carte interactive). C'est le fichier servi au public.
azure-function/          → la fonction qui lit le SharePoint et sert les données en JSON
  GetProjectsData/
    function.json
    index.js
  host.json
  package.json
  local.settings.json.example
```

## Ce que ça donne une fois en place

- N'importe qui ouvre l'URL du site (ex. `https://votre-compte.github.io/carte-mousquetaires/`)
  → voit la carte à jour, sans rien installer, sans se connecter.
- La page interroge automatiquement la fonction Azure, qui lit elle-même le fichier
  Excel sur SharePoint et renvoie les dernières données.
- L'import manuel d'un fichier Excel reste disponible dans l'outil, en secours
  (utile si la synchronisation automatique est momentanément indisponible, ou
  pour tester une version du fichier avant qu'elle ne soit sur SharePoint).

## Étape 1 — Héberger la page (GitHub Pages)

1. Créez un dépôt GitHub (public ou privé selon vos besoins de partage), par
   exemple `carte-mousquetaires`.
2. Poussez-y `index.html` (à la racine du dépôt).
3. Dans les paramètres du dépôt : **Settings → Pages**, choisissez la branche
   `main` et le dossier `/ (root)`.
4. GitHub vous donne une adresse du type
   `https://votre-compte.github.io/carte-mousquetaires/` — c'est le lien à
   partager. Chaque fois que vous poussez une nouvelle version de `index.html`,
   cette adresse affiche automatiquement la dernière version (quelques minutes
   de délai).

À ce stade, la carte fonctionne déjà pour tout le monde, mais avec les données
telles qu'elles étaient au moment du dernier import manuel de chaque visiteur.
L'étape 2 la rend vraiment automatique pour tous.

## Étape 2 — Lire le SharePoint automatiquement (Azure)

Cette partie nécessite un accès administrateur à votre tenant Microsoft 365 —
à faire par vous ou votre service informatique.

### 2.1 Enregistrer une application dans Azure AD

1. [portal.azure.com](https://portal.azure.com) → **Azure Active Directory
   → Inscriptions d'applications → Nouvelle inscription**.
2. Donnez-lui un nom (ex. "Carte Mousquetaires — lecture SharePoint").
3. Une fois créée, notez :
   - **Application (client) ID** → `CLIENT_ID`
   - **Directory (tenant) ID** → `TENANT_ID`
4. **Certificats et secrets → Nouveau secret client** → notez la **valeur**
   générée (visible une seule fois) → `CLIENT_SECRET`.
5. **Autorisations API → Ajouter une autorisation → Microsoft Graph →
   Autorisations d'application** (pas "déléguées") → ajoutez `Files.Read.All`
   (ou `Sites.Read.All` si vous préférez restreindre à un site précis).
6. Cliquez sur **Accorder le consentement administrateur** — cette étape doit
   être faite par un administrateur du tenant. Sans elle, la fonction ne
   pourra rien lire.

### 2.2 Retrouver l'identifiant du site et du fichier

Le plus simple : dans un navigateur, connecté à votre compte Microsoft 365,
ouvrez ces deux adresses (adaptez le nom de domaine) :

```
https://graph.microsoft.com/v1.0/sites/dimeoenergiefr.sharepoint.com:/sites/SUPPORTSCOMMERCIAUX
```
→ donne le **SITE_ID** dans la réponse JSON (champ `id`).

```
https://graph.microsoft.com/v1.0/sites/{SITE_ID}/drive/root:/Avancement_Mousquetaires.xlsm
```
→ donne l'**ITEM_ID** du fichier (champ `id`).

(Ces deux appels nécessitent d'être authentifié — le plus simple est de les
tester dans [Graph Explorer](https://developer.microsoft.com/graph/graph-explorer)
avec votre propre compte, juste pour récupérer les identifiants.)

### 2.3 Déployer la fonction

Avec l'extension Azure Functions dans VS Code, ou en ligne de commande :

```bash
cd azure-function
npm install -g azure-functions-core-tools@4
func azure functionapp publish <nom-de-votre-function-app>
```

Puis dans le portail Azure, sur votre Function App → **Configuration** →
ajoutez les variables d'environnement listées dans
`local.settings.json.example` (`TENANT_ID`, `CLIENT_ID`, `CLIENT_SECRET`,
`SITE_ID`, `ITEM_ID`, et `ALLOWED_ORIGIN` pointant vers votre adresse GitHub
Pages).

Vous obtenez une URL du type :
```
https://carte-mousquetaires-func.azurewebsites.net/api/GetProjectsData
```

### 2.4 Brancher l'URL sur la carte

Deux façons, pas exclusives :

- **Pour tout le monde d'un coup** : ouvrez `index.html`, trouvez la ligne
  `const DEFAULT_SYNC_URL = "";` tout en bas du script, et collez-y l'URL de
  la fonction. Repoussez le fichier sur GitHub — tous les visiteurs du lien
  seront alors synchronisés automatiquement, sans rien configurer eux-mêmes.
- **Pour tester avant de généraliser** : dans l'outil, champ "Synchronisation
  automatique" de la barre latérale, collez la même URL et cliquez sur
  "Activer" — ça ne concerne que votre propre navigateur.

## Alternative plus simple, sans code (Power Automate)

Si écrire/déployer une fonction Azure n'est pas envisageable tout de suite,
un flux Power Automate peut jouer le même rôle. Voici la marche à suivre en
détail.

**Prérequis — transformer les plages en vrais tableaux Excel** (Power Automate
ne lit que des tableaux structurés, pas de simples plages) :
1. Sélectionnez les données de l'onglet "Suivi" (avec en-têtes) → `Ctrl+T` →
   cochez "Mon tableau comporte des en-têtes".
2. Renommez le tableau `TableSuivi` (onglet contextuel "Création de tableau").
3. Répétez pour "Suivi prospect" → nommez-le `TableProspect`.
4. Enregistrez.

**Étape 1 — Créer le réceptacle public des données (Gist GitHub)**
1. [gist.github.com](https://gist.github.com) → nouveau gist, fichier
   `data.json`, contenu initial `{"suivi":[],"prospect":[]}`.
2. **Create public gist** (obligatoire pour un accès public sans authentification).
3. Notez l'URL — la partie après votre nom d'utilisateur est le `GIST_ID`.
4. L'adresse finale utilisée par la carte :
   `https://gist.githubusercontent.com/votre-compte/GIST_ID/raw/data.json`
   (pointe toujours vers la dernière version).

**Étape 2 — Créer un jeton GitHub pour l'écriture**
1. GitHub → Settings → Developer settings → Personal access tokens →
   Tokens (classic) → Generate new token.
2. Cochez uniquement la case `gist`. Définissez une expiration.
3. Copiez le jeton immédiatement (non réaffiché ensuite).

**Étape 3 — Construire le flux**

⚠️ L'action HTTP utilisée plus bas est un connecteur *Premium* — vérifiez
avec votre administrateur Power Platform que votre licence y donne accès.

a. Déclencheur : créez un **Flux cloud programmé** (pas "automatisé") — ce
   modèle démarre directement avec un déclencheur **Récurrence**. Renseignez
   la date de départ et la fréquence (ex. toutes les 30 minutes, fuseau
   `Romance Standard Time`). Alternative : un déclencheur SharePoint
   *"Quand un fichier est modifié dans un dossier"* si vous préférez réagir
   aux changements plutôt qu'à un intervalle fixe. Avec une récurrence de
   30 min, une modification peut prendre jusqu'à 30 min avant d'apparaître
   sur la carte — ajustez la fréquence selon vos besoins.

b. Deux actions *"Lister les lignes présentes dans un tableau"*
   (connecteur Excel Online (Business)) : une sur `TableSuivi`, une sur
   `TableProspect`.

c. Action *Compose* (vue "Texte") :
   ```
   {
     "suivi": [sortie "value" de la 1ère action "Lister les lignes"],
     "prospect": [sortie "value" de la 2ème action "Lister les lignes"]
   }
   ```
   Insérez les sorties via "Ajouter du contenu dynamique" aux bons endroits.

d. Action HTTP (le point délicat — imbriquer du JSON dans du JSON) :
   - Méthode : `PATCH`
   - URI : `https://api.github.com/gists/GIST_ID`
   - En-têtes : `Authorization: token VOTRE_PAT_GITHUB`,
     `User-Agent: PowerAutomate` (obligatoire, sinon GitHub rejette la requête),
     `Content-Type: application/json`
   - Corps :
     ```
     {
       "files": {
         "data.json": {
           "content": @{string(outputs('Compose'))}
         }
       }
     }
     ```
     Le `@{string(outputs('Compose'))}` s'insère via
     "Ajouter du contenu dynamique → Expression" en tapant
     `string(outputs('Compose'))`. C'est l'endroit le plus fréquent d'erreur
     de syntaxe si le flux échoue.

e. Testez manuellement une première fois et vérifiez qu'aucune étape n'échoue.

**Étape 4 — Vérifier** : ouvrez
`https://gist.githubusercontent.com/votre-compte/GIST_ID/raw/data.json` dans
un navigateur — vous devez voir vos données réelles (un léger délai de cache
GitHub de 1-2 min est normal).

**Étape 5 — Brancher sur la carte** : collez cette URL dans le champ
"Synchronisation automatique" de la barre latérale pour tester, puis, une
fois validé, remplacez `DEFAULT_SYNC_URL` dans `index.html` par la même
adresse et republiez sur GitHub Pages.

**À garder en tête** :
- Le Gist est public : n'y faites transiter que ce qui est déjà visible sur
  la carte (statuts, villes, commerciaux) — pas de données sensibles.
- Si le jeton GitHub expire, le flux échoue silencieusement — activez les
  alertes d'échec dans Power Automate (Mes flux → ⋯ → Paramètres → Alertes).
- Si l'action "Lister les lignes" échoue, c'est presque toujours parce que
  les plages ne sont pas de vrais tableaux Excel (prérequis ci-dessus).

C'est moins robuste qu'une fonction Azure (dépend de la disponibilité du
tableau Excel, moins de contrôle sur le format), mais ne nécessite pas
d'écrire de code.

## Import manuel (toujours disponible)

Que la synchronisation automatique soit en place ou non, le bouton "Mettre à
jour depuis Excel" dans la carte permet d'importer un fichier `.xlsm`/`.xlsx`
à la main. Cet import reste local au navigateur de la personne qui l'a fait
(un fichier HTML ne peut pas se réécrire lui-même) — c'est la synchronisation
automatique (étape 2) qui rend les données identiques pour tout le monde.
