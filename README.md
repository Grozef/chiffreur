Chiffreur de fichiers

Outil de chiffrement de fichiers universel, hors-ligne, en un seul fichier HTML.
On l'ouvre dans n'importe quel navigateur — zéro installation, zéro serveur, zéro
dépendance — et tout le chiffrement se fait localement dans la page. Aucun fichier
ne quitte la machine. 

> Le seul fichier nécessaire est `chiffreur.html`. On peut le copier sur une clé
> USB et l'utiliser partout (Windows / macOS / Linux / mobile).


Utilisation

1. Ouvrir `chiffreur.html` (double-clic, ou via un serveur local — voir plus bas).
2. Choisir un mode (onglet) :
   - Mot de passe — simple. Un mot de passe pour chiffrer, le même pour déchiffrer.
   - Clés RSA — pour le partage. On chiffre avec la clé publique (diffusable),
     on déchiffre avec la clé privée (secrète, protégée par un mot de passe).
3. Glisser un ou plusieurs fichiers (ou un dossier), saisir le secret, cliquer
   Chiffrer / Déchiffrer. Chaque fichier produit un `.enc` téléchargé.

> Attention: Un navigateur ne peut pas modifier ni supprimer un fichier du disque. L'outil
> crée une copie chiffrée (`.enc`) ; l'original reste à sa place, en clair. Pour réellement
> sécuriser un fichier, supprimer l'original soi-même après vérification (idéalement
> par un effacement sécurisé, ex. `cipher /w` sous Windows).

Glisser-déposer d'un dossier

Le dépôt d'un dossier utilise l'API FileSystem entries, bloquée en `file://`
(ouverture par double-clic). Pour cette fonction, servir la page via http, par ex.
avec Laragon : `http://localhost/chiffreur-fichiers/secure/chiffreur.html`. Le dépôt
de fichiers (et le bouton « choisir ») fonctionne partout, y compris en `file://`.


Technologies

Tout repose sur l'API WebCrypto (`SubtleCrypto`), standard W3C natif des navigateurs
modernes — gratuite, auditée, sans librairie tierce.

<table width="100%">
  <tr>
    <th width="33.33%">Rôle</th>
    <th width="33.33%">Algorithme / Fonction</th>
    <th width="33.33%">Détails & Sécurité</th>
  </tr>
  <tr>
    <td><b>Chiffrement du contenu</b></td>
    <td><code>AES-256-GCM</code></td>
    <td>Chiffrement authentifié (AEAD) : assure la confidentialité et l'intégrité sans HMAC séparé.</td>
  </tr>
  <tr>
    <td><b>Dérivation de clé (Password)</b></td>
    <td><code>PBKDF2-HMAC-SHA256</code></td>
    <td>310 000 itérations (recommandation OWASP) pour résister au <i>brute-force</i>.</td>
  </tr>
  <tr>
    <td><b>Gestion / Partage des clés</b></td>
    <td><code>RSA-4096 (OAEP-SHA256)</code></td>
    <td>Chiffrement hybride (modèle PGP) pour envelopper la clé de session AES.</td>
  </tr>
  <tr>
    <td><b>Génération d'aléa</b></td>
    <td><code>crypto.getRandomValues()</code></td>
    <td>CSPRNG (Générateur de nombres pseudo-aléatoires cryptographiquement sûr).</td>
  </tr>
</table>

Principe de Kerckhoffs : la sécurité ne tient qu'au mot de passe / à la clé privée,
jamais au secret du code (entièrement lisible dans `chiffreur.html`).

Chiffrement par blocs (gros fichiers)
`SubtleCrypto` n'a pas d'API de streaming : le fichier est donc découpé en blocs de
16 Mio, lus un par un (jamais tout en mémoire), chacun chiffré en AES-256-GCM. Une
barre de progression suit l'avancement. Pour empêcher qu'un attaquant retire,
réordonne ou tronque des blocs, chaque bloc embarque :

- un IV unique = préfixe aléatoire (8 o, 1×/fichier) ‖ compteur de bloc (4 o) ;
- des données authentifiées (AAD) = compteur de bloc ‖ marqueur « dernier bloc ».

Toute manipulation de l'ordre ou du nombre de blocs fait échouer l'authentification GCM.

Format du fichier `.enc`
Conteneur auto-décrit, versionné : `MAGIC "WEBENC1" | version | mode | nom d'origine |
matériel de clé | blocs chiffrés`. Le nom (ou chemin relatif) d'origine est restauré au
déchiffrement. Les fichiers produits par les versions précédentes restent déchiffrables
(lecteurs rétro-compatibles).

Option « Masquer le nom du fichier »
Par défaut, le nom d'origine est stocké en clair dans l'en-tête (pratique : on
identifie un `.enc` d'un coup d'œil, et le nom de téléchargement reprend l'original).
Quiconque ouvre le `.enc` peut donc lire ce nom. Si on coche « Masquer le nom du
fichier » (au chiffrement), le nom est chiffré dans l'en-tête (bloc AES-GCM à IV
dédié) et le `.enc` est téléchargé sous un nom générique (ex. `chiffre_a1b2c3.enc`).
Le vrai nom n'est plus lisible qu'après déchiffrement, où il est restauré à l'identique.



Ce projet est né du TP « Chiffrement de fichiers sensibles »,
qui demandait de protéger 4 fichiers selon 3 niveaux de sécurité (12 solutions), le niveau
fort exigeait « chiffrement asymétrique ou combiné, algorithmes modernes (AES-256, RSA) ».
Un immense merci a Mr BRIKCI-SID Boumediene pour l'inspiration et les ressources.

Plutôt qu'un outil différent par fichier (VeraCrypt, GPG, stéganographie…), cet outil
réalise le niveau Sécurité Forte de façon unifiée et reproductible : chiffrement
hybride RSA-4096 + AES-256 appliqué à n'importe quel type de fichier (il travaille sur
les octets, donc agnostique au format), avec en prime un mode mot de passe pour les usages
courants.


Limites (conceptuelles)

- Perdre le mot de passe ou la clé privée = données irrécupérables. Sauvegarder la clé
  privée hors du poste.
- L'outil ne supprime pas l'original (limite du navigateur) — à faire manuellement.
- La sortie est assemblée en mémoire : adapté jusqu'à plusieurs centaines de Mo. Pour du
  multi-Go en flux continu, il faudrait l'API File System Access (Chromium uniquement).
