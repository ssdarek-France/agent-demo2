# SENTINEL — Console IA générative

Application web de démonstration (formation IAE) : un agent conversationnel dont l'interface reprend les codes d'un tableau de bord de supervision cybersécurité (SOC). Chaque utilisateur choisit son fournisseur de modèle de langage (LLM), saisit **sa propre clé API**, puis discute avec le modèle. L'historique des conversations est archivé dans Firebase Firestore, rangé par utilisateur.

> Démo pédagogique. Ce n'est pas une architecture de production : lisez la section « Limites et risques ».

## Fichiers

| Fichier | Rôle | Où le mettre |
|---|---|---|
| `index.html` | La page : structure et apparence | GitHub |
| `app.js` | La logique : fournisseurs, appel au modèle, conversation, Firestore | GitHub |
| `firebase-config.js` | Configuration Firebase (publique), à remplir | GitHub |
| `README.md` | Ce document | GitHub |
| `firestore.rules` | Règles de sécurité de la base | Collé dans la console Firebase |

La procédure complète de mise en ligne, pas à pas, est dans `GUIDE-mise-en-ligne.md`.

## Fournisseurs pris en charge

| Fournisseur | Modèle par défaut | Appel direct depuis le navigateur |
|---|---|---|
| Groq | `llama-3.3-70b-versatile` | Oui |
| OpenAI | `gpt-4o-mini` | Oui |
| OpenRouter | `openai/gpt-4o-mini` | Oui |
| Anthropic (Claude) | `claude-haiku-5-5` | Oui, avec un en-tête spécial (voir plus bas) |
| Mistral AI | `mistral-small-latest` | Non garanti, à tester |
| Google Gemini (point d'accès compatible OpenAI) | `gemini-2.5-flash` | Non garanti, à tester |
| Autre | URL compatible OpenAI | Dépend du service |

Les noms de modèles changent souvent. Le bouton **Lister** interroge le fournisseur et propose les modèles réellement disponibles pour votre clé.

## Où va la clé API ?

C'est le point central de la démo.

```
Navigateur de l'utilisateur
 ├─ clé API  ──────────────►  Fournisseur LLM (Groq, OpenAI…)   ✔ seul destinataire
 └─ conversations (texte) ─►  Firestore, sous users/{uid}/…      ✘ jamais la clé
```

- La clé est gardée **en mémoire** (variable JavaScript). Si on recharge la page, elle disparaît.
- Option : case « Garder la clé pour cet onglet », qui la place dans `sessionStorage`. Elle est effacée à la fermeture de l'onglet.
- La clé n'est **jamais** écrite dans Firestore ni dans `localStorage`.
- Les règles `firestore.rules` n'acceptent qu'une liste fermée de champs : un champ `apiKey` serait refusé par le serveur.
- Garde-fou : un message qui contient votre clé est bloqué avant l'envoi, car les messages, eux, sont archivés.

### Les deux « clés » à ne pas confondre

| | Clé Firebase (`firebase-config.js`) | Clé du fournisseur LLM |
|---|---|---|
| Nature | Identifiant public du projet | Secret, lié à votre compte et à votre facture |
| Publiable sur GitHub ? | Oui | **Jamais** |
| Ce qui protège | Les règles Firestore et l'authentification | Vous seul |

## Comprendre le CORS (et pourquoi certains fournisseurs refusent)

Quand une page web hébergée sur `https://vous.github.io` appelle `https://api.fournisseur.com`, le navigateur demande d'abord au fournisseur : « acceptes-tu les requêtes venant de ce site ? ». C'est le mécanisme **CORS** (Cross-Origin Resource Sharing). Si le fournisseur ne répond pas « oui », le navigateur bloque la réponse, même si la clé est bonne.

Pourquoi un fournisseur refuserait-il ? Pour **vous protéger** : appeler une API depuis un navigateur, c'est exposer la clé à quiconque ouvre les outils de développement (touche F12, onglet Réseau). Les fournisseurs préfèrent que la clé reste sur un serveur.

- **Groq, OpenAI, OpenRouter** acceptent les appels directs.
- **Anthropic** les accepte seulement si la requête porte l'en-tête `anthropic-dangerous-direct-browser-access: true`. Le mot *dangerous* est volontaire : Anthropic vous fait signer, en quelque sorte, que vous acceptez le risque. C'est acceptable ici parce que chaque utilisateur apporte **sa propre** clé.
- **Mistral, Gemini, autres** : non garanti. Si ça bloque, l'application affiche « Requête bloquée avant d'atteindre le fournisseur ». Le message d'erreur exact est visible dans la console (F12).

Question à poser aux apprenants : *dans quel cas cet en-tête deviendrait-il une vraie faille ?* Réponse : si le développeur mettait **sa** clé dans le code de la page pour tous les visiteurs. N'importe qui pourrait la copier et consommer son crédit.

## Limites et risques (à discuter en formation)

1. **La clé est visible dans le navigateur.** Toute extension malveillante, ou toute faille XSS dans la page, pourrait la lire. L'application n'injecte jamais la réponse du modèle en HTML (`textContent` uniquement) pour limiter ce risque. En production, on placerait un petit serveur intermédiaire (Cloud Function, Cloudflare Worker) qui détient la clé.
2. **Authentification anonyme.** Simple pour une démo, mais l'historique est lié au navigateur : effacer ses données de navigation, ou changer d'appareil, et on perd l'accès à ses archives. N'importe qui peut aussi créer des comptes anonymes et remplir votre base. Pour un usage réel : connexion Google ou e-mail, Firebase App Check, alertes de budget.
3. **Données personnelles (RGPD).** Les conversations sont stockées dans **votre** projet Firebase : vous en êtes responsable. Prévenez les apprenants de ne rien saisir de sensible, et fixez une durée de conservation (par exemple une règle TTL Firestore).
4. **Le contenu envoyé au fournisseur LLM** est soumis à ses propres conditions (conservation, entraînement éventuel). À vérifier pour chaque fournisseur.
5. **Pas de streaming.** La réponse s'affiche d'un bloc, ce qui simplifie le code mais donne une attente plus longue sur les réponses longues.

## Structure des données Firestore

```
users/{uid}/conversations/{convId}
    title, provider, model, createdAt, updatedAt
users/{uid}/conversations/{convId}/messages/{msgId}
    role ("user" | "assistant"), content, createdAt
```

## Tester en local

Les modules JavaScript ne se chargent pas en ouvrant le fichier par double-clic (`file://`). Lancez un petit serveur depuis le dossier :

```bash
python3 -m http.server 8000
# puis ouvrez http://localhost:8000
```

Sans configuration Firebase, l'application fonctionne en **mode local** : la conversation marche, rien n'est archivé.

## Personnaliser

- Consigne système (le « rôle » de l'assistant) : constante `SYSTEM_PROMPT` en haut de `app.js`.
- Ajouter un fournisseur compatible OpenAI : une entrée de plus dans l'objet `PROVIDERS` de `app.js`.
- Couleurs : variables CSS dans `:root`, en haut de `index.html`.
