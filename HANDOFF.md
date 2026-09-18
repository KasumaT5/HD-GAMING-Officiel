# HD-Gaming officiel — Document de passation technique
*Rédigé par Claude, à l'attention de Qwen Code, pour continuité du projet*

---

## 1. Vue d'ensemble du projet

Site de téléchargement de jeux mobiles (Android, PSP/PPSSPP, mods, ROMs, ISO).
Propriétaire : Suneraku (KasumaT5 sur GitHub). Public cible : Côte d'Ivoire / francophone, principalement sur mobile.

- **Site en ligne :** https://hd-gaming-officiel.pages.dev
- **Dépôt GitHub :** KasumaT5/HD-Gaming-Officiel
- **Hébergement :** Cloudflare Pages, connecté au dépôt GitHub (déploiement automatique à chaque changement de fichier)
- **Base de données :** Supabase (Postgres)
  - URL : `https://aouuisrouenmqvzssbpp.supabase.co`
  - Clé publique (anon) : `sb_publishable_nQ2k-gbKPLyZ_2Qw2AbqAA_9Y-PM9xC`

---

## 2. Stack technique — choix volontaires, à respecter

**Pas de framework, pas de build, pas de npm.** Chaque page est un fichier `.html` autonome (HTML + CSS + JS dans le même fichier). C'est un choix délibéré, pas un raccourci :

- L'utilisateur publie ses fichiers en les uploadant manuellement sur GitHub via l'interface web (bouton "Add file → Upload files"), depuis son téléphone. Pas de terminal, pas de Git en ligne de commande.
- Introduire un système de build (React, Vite, etc.) casserait ce workflow. **Ne pas le faire sans demande explicite.**
- JavaScript vanilla partout, appels directs à l'API REST de Supabase via `fetch()`.

**Communication avec Supabase :** toujours via l'API REST (`${SUPABASE_URL}/rest/v1/...`), jamais via le SDK JS officiel (pas importé, pour rester en single-file sans dépendances).

---

## 3. Structure des fichiers du site

| Fichier | Rôle |
|---|---|
| `index.html` | Accueil — recherche + filtres catégorie, grille de jeux, favoris (cœur), squelette de chargement animé, section communauté, pubs Adsterra |
| `jeu.html` | Fiche jeu dynamique (lue via `?slug=...` dans l'URL) — galerie, description, likes/dislikes, commentaires, bouton(s) de téléchargement |
| `telecharger.html` | Page intermédiaire avec compte à rebours (8s) avant déblocage du lien, bloc partenaire MMOEXP, pub, bouton "copier le lien" |
| `favoris.html` | Liste des jeux mis en favori (stockés en `localStorage`, pas de compte utilisateur) |
| `admin.html` | Centre de contrôle privé — protégé par mot de passe vérifié côté serveur. 3 onglets : Publier / Mes jeux / Commentaires |
| `robots.txt`, `sitemap.xml` | SEO de base |
| `prompt-fiche-jeu.md` | Prompt réutilisable (hors site, à usage de l'utilisateur) pour générer une fiche jeu formatée à partir d'un lien |

**Convention de nommage important :** ne jamais renommer ces fichiers. Le site les référence par leur nom exact (`index.html`, `jeu.html`, etc.) dans les liens internes.

---

## 4. Schéma de la base de données Supabase

### Table `"Games"` (avec majuscule et guillemets obligatoires en SQL — piège connu, voir section 7)
```
id, created_at, title, slug, description, category, platform,
file_size, download_url, thumbnail_url, screenshots_urls (texte, urls séparées par virgules),
min_requirements, extra_links (jsonb, array de {label, url}),
likes (int), dislikes (int), downloads (int)
```

### Table `comments`
```
id, created_at, game_slug, author_name, message, admin_reply (texte, réponse de l'admin)
```

### Table `stats`
```
key (text, primary key), count (bigint)
```
Actuellement une seule ligne : `mmoexp_clicks`.

### Table `admin_secret`
```
id (=1), secret (text)
```
**Jamais accessible en lecture publique** (`revoke all ... from anon, authenticated`). Uniquement lue à l'intérieur des fonctions `security definer` ci-dessous.

### Fonctions RPC (Postgres) — le cœur du modèle de sécurité
Toutes en `security definer`, exécutées avec les droits du propriétaire de la base, pas ceux du visiteur anonyme :

- `verify_admin_secret(p_secret text) returns boolean` — vérifie le mot de passe sans rien modifier
- `admin_add_game(p_secret text, p_data jsonb)` — insère un jeu
- `admin_update_game(p_secret text, p_slug text, p_data jsonb)` — modifie un jeu existant
- `admin_delete_game(p_secret text, p_slug text)` — supprime un jeu
- `admin_reply_comment(p_secret text, p_id bigint, p_reply text)` — ajoute/modifie une réponse admin
- `admin_delete_comment(p_secret text, p_id bigint)` — supprime un commentaire
- `bump_game_stat(p_slug text, p_field text)` — incrémente likes/dislikes/downloads/views de +1 (pas de valeur arbitraire acceptée, uniquement +1)
- `bump_partner_click()` — incrémente le compteur de clics MMOEXP
- `admin_add_games_bulk(p_secret text, p_games jsonb)` — **ajoutée le 18/09**, insère/met à jour plusieurs jeux en un seul appel (upsert sur `slug`, qui est maintenant `unique`). Traite chaque jeu dans son propre `begin/exception` : un jeu en échec ne bloque pas les autres. Renvoie `{results: [{slug, title, ok, error}, ...]}`. Alimentée par le nouvel onglet "Masse & SEO" de `admin.html`. SQL à coller une fois dans Supabase : voir `sql-bulk-publish.sql`.

**Toutes ces fonctions commencent par vérifier `p_secret` contre la table `admin_secret` et lèvent une exception `'Accès refusé'` si le mot de passe ne correspond pas.**

---

## 5. Modèle de sécurité — À NE JAMAIS CONTOURNER

Règle stricte suivie depuis le début : **aucune écriture directe sur les tables depuis le client (JS)**. Tout passe par les fonctions RPC ci-dessus.

Pourquoi : la clé Supabase publique (`sb_publishable_...`) est visible dans le code source de chaque page — n'importe qui peut l'extraire. Si on autorisait un `PATCH`/`INSERT`/`UPDATE`/`DELETE` direct sur une table avec une policy RLS ouverte (`using (true)`), n'importe qui pourrait modifier ou supprimer le contenu du site à distance, ou spammer les compteurs.

**Si une nouvelle fonctionnalité nécessite une écriture en base, créer une nouvelle fonction RPC `security definer` avec vérification de `p_secret` (si action admin) — ne jamais ouvrir de policy RLS publique en écriture.**

La lecture (`SELECT`) reste publique sur `Games` et `comments` (policy `using (true)` pour select uniquement) — c'est voulu, le site doit être lisible par tout le monde.

---

## 6. Conventions de code à respecter

- **Langue :** tout le texte visible sur le site est en français. Les noms de variables/fonctions JS restent en anglais (convention de code standard).
- **Palette de couleurs (variables CSS présentes en haut de chaque fichier) :**
  ```css
  --bg:#060a14; --bg-panel:#0d1526; --bg-panel-2:#111b30; --line:#1c2740;
  --text:#eaf0fb; --text-dim:#8b96b3; --blue:#2e8bff; --blue-bright:#5ec9ff;
  --cyan:#26e0ff; --gold:#f4c95d; --green:#2ecc71; --red:#ff5c7a;
  ```
  Réutiliser ces variables, ne pas introduire de nouvelles couleurs sans raison.
- **Polices :** `Chakra Petch` (titres, gaming) + `Inter` (texte courant), chargées via Google Fonts.
- **Mobile-first strict :** l'utilisateur est exclusivement sur téléphone (petit écran, presse-papier limité). `max-width:480px` sur le conteneur principal de chaque page.
- **Animations :** toujours légères et discrètes (transitions douces, pas de rebonds ni d'effets criards). L'utilisateur a explicitement demandé de la sobriété ici.
- **Gestion d'erreurs JS :** tous les appels `fetch()` sont enveloppés en `try/catch` silencieux côté visiteur (`catch(e){}`) pour ne jamais faire planter l'affichage — mais avec message utilisateur clair dans les zones de statut (`#statusMsg`, etc.) côté admin.
- **Échappement HTML :** toute donnée utilisateur affichée (commentaires, pseudos) passe par une fonction `escapeHtml()` locale à chaque fichier avant d'être injectée dans le DOM — protection XSS de base. Répliquer ce pattern pour tout nouveau contenu généré par les visiteurs.

---

## 7. Pièges connus / erreurs déjà rencontrées (à éviter)

1. **Nom de table `Games` avec majuscule.** Supabase l'a créée ainsi automatiquement. En SQL, toujours écrire `"Games"` avec guillemets doubles et majuscule — `games` (minuscule) échoue silencieusement avec une erreur "relation does not exist".
2. **Renommage de fichiers par Chrome.** Quand l'utilisateur télécharge plusieurs fois un fichier du même nom depuis son téléphone, Chrome ajoute automatiquement `-1`, `-2`, etc. Il doit renommer manuellement en `index.html` (ou autre nom exact attendu) avant de l'uploader sur GitHub, sinon le site charge l'ancienne version.
3. **RLS (Row Level Security) doit être activé** sur chaque nouvelle table (`alter table X enable row level security`), sinon Supabase bloque tout par défaut même la lecture — mais il faut aussi une policy explicite pour autoriser le `select` public.
4. **Chrome traduit automatiquement le code SQL collé** si la traduction auto est activée sur supabase.com — ça corrompt les mots-clés SQL. L'utilisateur doit désactiver la traduction sur cette page si l'éditeur SQL affiche du texte traduit au lieu du code.
5. **`sessionStorage` pour le mot de passe admin** n'est PAS une vérification de sécurité en soi — un ancien bug (déjà corrigé) laissait entrer n'importe qui tant que le champ n'était pas vide. Le vrai contrôle doit toujours passer par `verify_admin_secret()` côté serveur, jamais une simple vérification `if(pass)` côté client.

---

## 8. Format de livraison des fichiers (convention de session)

Quand un fichier est modifié ou créé, il est :
1. Écrit/modifié dans un espace de travail
2. Copié vers un dossier de sortie où l'utilisateur peut le télécharger
3. Présenté à l'utilisateur avec un lien de téléchargement direct

**L'utilisateur télécharge ensuite le fichier sur son téléphone, puis l'upload manuellement sur GitHub** (Add file → Upload files → Commit changes). Cloudflare Pages redéploie automatiquement en quelques secondes après chaque commit. Il n'y a jamais d'accès direct en écriture au dépôt GitHub depuis l'assistant — c'est toujours l'utilisateur qui fait le transfert final.

**Format des fichiers livrés :** HTML brut (`.html`), SQL brut à copier-coller dans l'éditeur SQL Supabase (jamais de fichier `.sql` à uploader), Markdown (`.md`) pour la documentation/prompts.

---

## 9. État actuel du projet (dernier point d'arrêt)

✅ **Terminé et fonctionnel :**
- Les 5 pages du site (accueil, fiche jeu, téléchargement, favoris, admin)
- Recherche + filtres catégorie fonctionnels
- Likes/dislikes/téléchargements/vues comptabilisés de façon sécurisée
- Commentaires avec réponse admin
- Liens de téléchargement multiples (jeu + texture + save pour les mods PPSSPP)
- Page admin complète : publier, modifier, supprimer un jeu ; statuts publié/brouillon/masqué avec badges et filtres ; "à la une" (`featured`) ; voir/répondre/supprimer les commentaires ; gestion des partenaires
- Auto-remplissage niveau 1 (format structuré) + niveau 2 (détection regex sur texte désordonné) + niveau 3 (validation ✓/⚠️ par champ)
- Upload d'images direct vers Supabase Storage (bucket `Game-image`) depuis l'admin — plus besoin de coller une URL à la main
- SEO par fiche, généré dynamiquement à chaque chargement de `jeu.html` : `<title>`, meta description, Open Graph, JSON-LD `SoftwareApplication` avec note agrégée, et **balise canonical** (ajoutée le 18/09)
- `telecharger.html` et `favoris.html` passées en `noindex` (18/09) pour ne pas diluer le référencement avec des pages intermédiaires/pauvres
- **Publication en masse** (18/09) : nouvel onglet "Masse & SEO" dans `admin.html`, colle plusieurs fiches à la suite (une nouvelle fiche = une ligne `title:`), aperçu avec détection des champs manquants et des slugs en double, publication en un seul appel via la RPC `admin_add_games_bulk` (voir `sql-bulk-publish.sql`, à coller une fois dans Supabase)
- **Générateur de sitemap.xml** (18/09) : bouton dans le même onglet, reconstruit le fichier complet à partir de tous les jeux `publie` en base — à relancer après chaque session de publication, puis copier/coller dans `sitemap.xml` et re-uploader sur GitHub
- Sécurité : toutes les écritures passent par RPC avec vérification de mot de passe serveur
- MMOEXP intégré (code HDGAMING, lien réel, config centralisée dans `PARTNER_CONFIG` sur `telecharger.html`)
- Adsterra intégré (bannière classique, native, Social Bar, Popunder)
- Clause légale en pied de page

🔲 **Connu mais non résolu (limite technique, pas un bug à corriger en code) :**
- Le script Popunder d'Adsterra redirige parfois le navigateur entier vers une offre publicitaire cassée/expirée au lieu d'ouvrir un nouvel onglet discret. Ce n'est pas un bug du site — c'est un comportement du script tiers Adsterra. Solution recommandée à l'utilisateur : régler le "Frequency Cap" dans son dashboard Adsterra et signaler l'offre cassée à leur support. **Ne pas essayer de "patcher" ça en JS, ce n'est pas notre code qui déclenche la redirection.**

🔲 **En attente / pas encore commencé :**
- Publication du contenu réel (~20 jeux prévus, 0 publiés actuellement après suppression du jeu de test)
- Upload d'images via Supabase Storage (bucket `game-images` à créer, pas encore fait à ma connaissance)
- Nouvelles propositions apportées par l'utilisateur (travaillées avec ChatGPT) — pas encore communiquées à ce document, à récupérer directement auprès de l'utilisateur
- Soumission du sitemap à Google Search Console (action manuelle utilisateur, pas de code)

---

## 10. Comment continuer dans le même esprit

- Toujours proposer des changements **incrémentaux et testables**, jamais une refonte complète sans validation.
- Toujours garder le site dans un **état fonctionnel** après chaque livraison — ne jamais laisser une fonctionnalité à moitié câblée (ex: un bouton visible qui ne fait rien).
- Poser une question de clarification unique et ciblée si une demande est ambiguë, plutôt que de deviner sur un point qui touche la sécurité ou l'argent (MMOEXP, Adsterra).
- Expliquer chaque terme technique en français simple — l'utilisateur n'est pas développeur, il gère un site par délégation complète du code, mais veut comprendre ce qui se passe.
- Ne jamais casser le workflow "upload manuel de fichiers HTML autonomes" sans qu'il le demande explicitement.
