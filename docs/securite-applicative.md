# Durcissement de la sécurité applicative

CSRF, validation des entrées, limitation de débit et sécurisation des jetons JWT.

## 1. Objectif

Une fois l'architecture 3-tiers fonctionnelle et le transport sécurisé (HTTPS, reverse
proxy nginx), cette phase renforce la couche applicative. Quatre protections issues des
recommandations OWASP, réparties entre le frontend Flask (WEB-01) et l'API FastAPI (APP-01) :

- CSRF, contre la falsification de requêtes inter-sites
- validation des entrées, contre les données malformées et les injections
- limitation de débit, contre la force brute
- sécurisation des jetons JWT, contre la forge et le rejeu

## 2. Protection CSRF (WEB-01)

Flask-WTF protège tous les formulaires en POST. Un jeton unique est généré par session et
injecté dans chaque formulaire ; une soumission sans jeton valide est rejetée.

```python
from flask_wtf import CSRFProtect
csrf = CSRFProtect(app)
```

```html
<input type="hidden" name="csrf_token" value="{{ csrf_token() }}">
```

Le bouton de déconnexion, qui était un simple lien en GET, est passé en formulaire POST
protégé par jeton. Une action sensible ne doit jamais s'exécuter sur une requête GET —
il suffirait d'une image pointant vers l'URL de logout pour déconnecter l'utilisateur.

## 3. Validation des entrées

Appliquée aux deux niveaux, pour ne pas dépendre d'une seule couche.

Côté API, `EmailStr` de Pydantic valide le format de l'adresse, et un validateur contrôle
la longueur du mot de passe :

```python
class LoginBody(BaseModel):
    email: EmailStr
    password: str

    @field_validator("password")
    def password_non_vide(cls, v):
        if not v or len(v) > 128:
            raise ValueError("Mot de passe invalide")
        return v
```

Côté frontend, `email-validator` vérifie l'email avant transmission et les longueurs sont
contrôlées.

Détail qui compte : en cas d'identifiants erronés, le message renvoyé est le même que le
compte existe ou non (« Identifiants incorrects »). Sinon, la différence de réponse permet
d'énumérer les comptes valides.

## 4. Limitation de débit

Cinq tentatives par minute et par adresse IP sur l'authentification, côté frontend
(Flask-Limiter) comme côté API (SlowAPI).

```python
@app.post("/token")
@limiter.limit("5/minute")
def login(...): ...
```

Au-delà, réponse `429 Too Many Requests` avec un message explicite pour l'utilisateur
légitime qui se serait trompé plusieurs fois.

## 5. Sécurisation des jetons JWT (APP-01)

Cinq points renforcés :

- **Secret obligatoire** — l'application refuse de démarrer si `JWT_SECRET` est absent ou
  fait moins de 32 caractères. Secure by default : pas de secret faible par oubli.
- **Expiration courte** — 15 minutes, ce qui réduit la fenêtre d'exploitation d'un jeton volé.
- **Identifiant unique (`jti`)** — chaque jeton est identifiable, donc révocable individuellement.
- **Révocation au logout** — le jeton est invalidé côté serveur ; il ne sert plus à rien même
  s'il n'a pas expiré.
- **Validation stricte** — signature, expiration et champs obligatoires vérifiés ; tout jeton
  forgé ou altéré est rejeté.

```python
payload = {
    "sub": email, "role": role, "uid": uid,
    "iat": now, "exp": now + timedelta(minutes=15),
    "jti": str(uuid.uuid4()),
}
```

Vérifié en test : un jeton forgé avec un mauvais secret repart en 401, et un jeton reste
inutilisable après déconnexion.

## 6. Problèmes rencontrés

**L'API refusait de démarrer** — `RuntimeError: JWT_SECRET manquant ou trop court` au
lancement d'uvicorn. C'était le comportement voulu : la variable n'était pas définie dans
le terminal courant. Corrigé avec `export JWT_SECRET=$(openssl rand -hex 32)`. Le refus de
démarrer est en soi une garantie.

**Conflit au `git pull`** — `Your local changes would be overwritten by merge`. Des
correctifs avaient été faits directement sur WEB-01, en conflit avec la version du dépôt.
Annulation des modifications locales devenues obsolètes (`git checkout -- <fichiers>`)
puis `git pull`, la nouvelle version étant plus complète.

**Disque plein** — `No space left on device` pendant l'installation des dépendances, cache
pip et paquets apt accumulés sur WEB-01. Nettoyage (`pip cache purge`, `apt clean`,
`apt autoremove`) puis réinstallation avec `--no-cache-dir`.
