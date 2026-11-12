# Roadmap — Parcours A · Pré-production (recette)

**Cible :** `https://recto.axo-lab.fr` (branche `develop`, données de test)
**Autorisation :** exercice mandaté par AXO-LAB, périmètre A uniquement.
**Marquage obligatoire sur chaque requête :** en-tête `X-Pentest: <initiales>` + suffixe User-Agent `recto-pentest-<initiales>`.
**Interdits rappelés :** DoS incontrôlé, pivot infra/tiers (Supabase, Dokploy, OVH, Brevo, GitHub, DNS), SSH/système, ingénierie sociale/physique.

---

## Phase 0 — Préparation & accès

- [ ] Récupérer auprès de la coordination les **identifiants HTTP Basic** du site de recette (sinon → 401).
- [ ] Configurer le **proxy d'interception** (Burp/Caido) avec l'auth HTTP + l'en-tête `X-Pentest` et le suffixe UA injectés automatiquement sur tout le trafic.
- [ ] Mettre en place le **journal horodaté** (requête brute / réponse) dès maintenant — exigé pour le compte rendu.
- [ ] Activer les **5 comptes de test** (un par rôle) via `/connexion` : saisir l'adresse → code OTP transmis par la coordination → TOTP (Authy/Google Auth). Invitations valables 7 jours.
  - veilleur 03, veilleur 63, veilleur 03 bis, referent 03, coordination nationale.
- [ ] Préparer des **domaines/canaries sous contrôle** pour SSRF et XSS (charges inertes, jamais destructrices).

---

## Phase 1 — Reconnaissance

- [ ] Cartographier le front Next.js : pages, Server Actions, proxy, routes API.
- [ ] Lister les surfaces API : `/api/v1/*`, `/api/v2/*` (dont `/api/v2/domaines`), `/api/collecte/*`, `/api/altcha`, `/api/interne/*`.
- [ ] Extraire du code client la **clé publiable (anon)** et l'**URL Supabase** (publiques par conception).
- [ ] Énumérer les **RPC PostgREST** ouvertes à `anon` / `authenticated`.
- [ ] Repérer chemins oubliés : `.git`, `.well-known`, source maps, `.env`, artefacts de build, points de debug.
- [ ] Empreinte passive des versions (framework, libs) → CVE connues (A06).

---

## Phase 2 — Boîte noire (sans compte)

Ce qu'un anonyme atteint et devine.

- [ ] **API directe via clé anon** : appeler PostgREST/RPC sans l'interface, observer ce que la base autorise vraiment.
- [ ] **SSRF collecteur (③, A10)** : déclencher la collecte sur un domaine résolvant vers interne (`127.0.0.1`, `169.254.169.254`, plages privées) → refusé ? Tester redirections, DNS rebinding, IPv6→IPv4, schémas non-web. **Quelques domaines ciblés, jamais une rafale.**
- [ ] **XSS public (④, A03)** : contact, signalement, contestation, candidature, recherche (stocké + réfléchi).
- [ ] **Injection** : en-têtes dans les courriels déclenchés ; SQL via entrées des RPC.
- [ ] **API publique (⑥, API3)** : `/api/v2/domaines` → énumération ; cartes divulguent-elles champs internes / données perso ?
- [ ] **Config/headers** : en-têtes de sécurité, drapeaux cookies, `Cache-Control` des pages sensibles, traces verbeuses.
- [ ] **Logique métier publique (⑤)** : via signalement public, ouvrir un cas pour domaine arbitraire, choisir son département, inonder la file.
- [ ] **Routes internes** : `/api/interne/*` refusent-elles proprement sans le secret ?

---

## Phase 3 — Avec comptes (par rôle) — le cœur du test

### 3a. Contrôle d'accès RPC (①) — priorité n°1 (API1/API5 BOLA/BFLA, A01)
- [ ] Un veilleur **03 lit-il** un cas / fiche / dossier / alerte / carte du **63** ? (isolation départements)
- [ ] **03 vs 03 bis** : lit-il la fiche d'un autre correspondant du même cas ? (isolation personnes)
- [ ] **IDOR** : rejouer une requête d'interface avec un identifiant d'objet modifié.
- [ ] Une **RPC d'admin** (invitations, réglages, journal) répond-elle à un non-coordination ou à `anon` ?
- [ ] **Auto-attribution** de rôle / portée via RPC ou champ non filtré ?

### 3b. Auth, session, 2FA (②, A07)
- [ ] **Énumération d'adresses** : invitée vs inconnue se distinguent (réponse / code / délai) ?
- [ ] **Bourrage** OTP et TOTP : limites par personne (pas seulement globales) ?
- [ ] **aal1 → aal2** : atteindre une page/action réservée à `aal2` en restant `aal1` ? Sauter l'étape TOTP ?
- [ ] Fixation / réutilisation de session, jetons survivant à la déconnexion, falsification des claims JWT.

### 3c. Logique métier du circuit humain (⑤, API6)
- [ ] Confirmer / valider / trancher un cas qu'on a soi-même instruit ? Sauter la double confirmation ?
- [ ] Marquer une carte « vérifiée » sans le second avis ?
- [ ] Détourner génération / dépôt d'un dossier (fichiers & liens signés) ?

### 3d. XSS stocké dans l'espace (④)
- [ ] Réponses de fiche, constats, commentaires → réapparaissent sans échappement, dans contexte privilégié ?

---

## Phase 4 — Surfaces avancées (second rideau, selon le temps)

- [ ] **⑦ CSRF & Server Actions (A01/A05)** : déclenchement cross-site d'une Server Action POST ; contrôles origine/hôte & SameSite ; rejeu d'action.
- [ ] **⑧ Redirections ouvertes / confusion d'URL (A01)** : `?retour=` vers domaine externe ; double encodage, casse, Unicode/punycode, `//`, `/./`, `%2e`, point final sur l'hôte → contourner le garde d'accès.
- [ ] **⑨ Race conditions & rejeu (API6)** : double réservation de cas, double consommation invitation/code, dépassement quota ; TOCTOU sur transition d'état.
- [ ] **⑩ Mass assignment & pollution de paramètres (API6/API3)** : champs non attendus (rôle, portée, état, auteur, id cas) ; params dupliqués, tableau vs valeur, type inattendu.
- [ ] **⑪ Crypto & liens signés (A02)** : algo JWT accepté & signature vérifiée ; liens/fichiers signés prédictibles/forgeables, HMAC sensible à troncature/temps ; ALTCHA + honeypot contournables/rejouables.
- [ ] **⑫ Cache web, CORS, Host (A05)** : cache deception (`.css`/`.js`) & poisoning ; `ACAO: *` jamais avec credentials ; injection `Host`/`X-Forwarded-*` sur redirection canonique, liens, compteur de rate-limit.
- [ ] **⑬ Extension de navigateur (A05)** :
  - Décompresser & lire manifeste + bundle (lecture seule) ; pour la pousser contre la recette, rebuild depuis `apps/extension` avec `WXT_API_URL` (voir `apps/extension/README.md`, fournir les creds HTTP).
  - Vie privée : ne transmet que le domaine de l'onglet actif, jamais historique/URL ? Conforme à la politique déclarée ?
  - Permissions minimales (`activeTab`), `externally_connectable`, CSP MV3 sans code distant.
  - Cloisonnement/messaging (`onMessage`/`onMessageExternal`), XSS dans la popup, intégrité du téléchargement de liste de domaines.
  - Usurpation (homographe/punycode/alias `www.`/`m.`) ; injection dans `/api/v2/cartes/<domaine>` ; mêmes rate-limits que le site.
- [ ] **⑭ Composants connus & chemins exposés (A06/A05)** : CVE des versions ; `.git`, backups, sources, artefacts, debug.

---

## Phase 5 — Test de charge (optionnel, EN DERNIER, A.5)

**Seule exception à l'interdit DoS — strictement encadrée.**

- [ ] **Recette uniquement** (prod & recette partagent VPS/Traefik/Supabase).
- [ ] **Coordonné avec l'infra (Clément)** : fenêtre convenue, à l'écart du reste du test, surveillance en direct.
- [ ] **Progressif & borné** : paliers, plafond débit/concurrence convenu, durée limitée — chercher le point de bascule, pas s'y installer.
- [ ] **Surveillé** : espace d'admin, métriques conteneurs (Dokploy), journaux.
- [ ] **Arrêt immédiat** au moindre signe d'impact prod/VPS.
- Observer : tenue des rate-limits par visiteur, dégradation propre (429/503 nets vs 5xx/stack traces), amplification (collecte / génération PDF-zip), épuisement ressources (pooler, mémoire, file), récupération auto.

---

## Phase 6 — Compte rendu (au fil de l'eau)

Pour **chaque constat** :
- Titre + **gravité** (critique / haute / moyenne / basse) avec raisonnement.
- **Où** : URL ou RPC, parcours, rôle, horodatage.
- **Reproduction** : étapes minimales, requête brute, réponse **expurgée des données perso**.
- **Impact** + **correctif proposé**.

→ Journal au fil de l'eau, puis **synthèse triée par gravité**.
→ **Signaler tout point critique immédiatement**, sans attendre la synthèse.

### Conditions d'arrêt (stop + prévenir, ne pas exploiter plus loin)
RCE · accès base en écriture · lecture clé secrète/`service_role` · accès à des données réelles de prod · impact involontaire sur le service réel.
