# Configurateur Covering — devis automatiques

Application web qui permet à un client de choisir son véhicule, de le visualiser en 3D dans la couleur et la finition de son choix, et d'obtenir immédiatement un devis pour un covering automobile et des prestations associées (PPF, vitres teintées, detailing). Un espace professionnel permet à l'atelier de gérer les devis reçus en ligne et de créer des devis au comptoir.

> **Statut :** conception terminée, développement à venir.
> Ce dépôt contient uniquement la documentation technique. Le code source est hébergé dans un dépôt privé.

---

## Sommaire

1. [Fonctionnalités](#fonctionnalités)
2. [Parcours et écrans](#parcours-et-écrans)
3. [Stack technique](#stack-technique)
4. [Architecture](#architecture)
5. [Calcul du prix](#calcul-du-prix)
6. [Aperçu 3D](#aperçu-3d)
7. [Modèle de données](#modèle-de-données)
8. [Design](#design)
9. [Sécurité et RGPD](#sécurité-et-rgpd)
10. [Feuille de route](#feuille-de-route)
11. [Méthode de développement](#méthode-de-développement)
12. [Droits](#droits)

---

## Fonctionnalités

### Côté client
- Sélection du véhicule : marque, modèle, année, avec catégorie détectée automatiquement (citadine, berline, SUV, utilitaire) et modifiable.
- Aperçu 3D du véhicule : rotation, vues prédéfinies, couleur et finition appliquées en direct.
- Choix du type de covering (total, partiel, toit + rétroviseurs) et de la finition (brillant, satiné, mat, chrome).
- Prestations complémentaires : PPF, teinte des vitres, detailing, dépose d'un ancien film.
- Prix TTC et délai estimés recalculés à chaque changement.
- Devis récapitulatif (HT, TVA, TTC) téléchargeable en PDF et envoyé par e-mail.
- Déclaration de l'état de la carrosserie et envoi de photos, pour anticiper une éventuelle préparation.
- Demande de rendez-vous.

### Côté atelier (espace pro)
- Liste des devis avec recherche et filtres par statut : envoyé, accepté, à relancer, refusé.
- Origine du devis : site web ou comptoir.
- Création d'un devis au comptoir avec le même moteur de calcul.
- Gestion de la grille de tarifs sans toucher au code.
- Fiches clients et planning (phase ultérieure).

---

## Parcours et écrans

```
Accueil ──▶ 1. Véhicule ──▶ 2. Personnalisation (3D) ──▶ 3. Devis
                                                          │
                                       Espace pro ◀───────┘ (devis enregistré)
```

| Écran | Contenu principal |
|---|---|
| **Accueil** | Présentation de l'atelier, prestations, réalisations, appel à l'action « Simuler mon devis ». |
| **1. Véhicule** | Sélecteurs marque / modèle / année, 4 cartes de catégorie, récapitulatif de la sélection. |
| **2. Personnalisation** | Aperçu 3D + 8 pastilles de couleur à gauche ; type de covering, finition, options et prix à droite. |
| **3. Devis** | Document de devis (n°, véhicule, teinte, lignes, totaux) + formulaire de contact avec consentement RGPD. |
| **Espace pro** | Menu latéral (Devis, Clients, Grille de tarifs, Planning, Paramètres), tableau des devis. |

Les écrans sont responsives : sur mobile, les colonnes s'empilent et les tableaux défilent horizontalement.

---

## Stack technique

| Couche | Choix | Raison |
|---|---|---|
| Front-end | **Vite + React + TypeScript** | Démarrage rapide, typage du moteur de prix. |
| 3D | **three.js** via **@react-three/fiber** et **@react-three/drei** | Intégration native avec React, contrôles de caméra prêts à l'emploi. |
| Style | CSS modules ou Tailwind | Thème sombre simple, variables de couleur centralisées. |
| PDF | Génération côté client (ex. `@react-pdf/renderer`) | Pas de serveur nécessaire pour le PDF. |
| Tests | **Vitest** | Tests unitaires du moteur de prix. |
| Back-end | À définir à l'étape 5 (ex. Supabase ou petite API Node) | Enregistrement des devis, authentification de l'espace pro, envoi d'e-mails. |
| Hébergement | À définir (ex. Vercel, Netlify) | Déploiement statique du front. |

---

## Architecture

```
src/
├── config/
│   ├── tarifs.json          # Grille de tarifs (aucun prix en dur dans le code)
│   └── vehicules.json       # Marque → modèle → catégorie
├── pricing/
│   ├── calculerDevis.ts     # Moteur de calcul, fonction pure
│   └── calculerDevis.test.ts
├── components/
│   ├── Stepper.tsx
│   ├── SelecteurVehicule.tsx
│   ├── CarteCategorie.tsx
│   ├── Apercu3D.tsx
│   ├── PastillesCouleur.tsx
│   ├── OptionsPrestations.tsx
│   ├── EncadrePrix.tsx
│   └── DocumentDevis.tsx
├── pages/
│   ├── Accueil.tsx
│   ├── Vehicule.tsx
│   ├── Personnalisation.tsx
│   ├── Devis.tsx
│   └── pro/
│       └── ListeDevis.tsx
└── state/
    └── configuration.ts     # Sélection en cours (véhicule, couleur, options)

public/
└── models/                  # Modèles 3D .glb, un par catégorie
```

**Principes :**
- Le moteur de prix est une **fonction pure** (`configuration + tarifs → devis`), partagée entre le parcours client et l'espace pro.
- Les tarifs et le catalogue de véhicules sont des **fichiers de configuration**, modifiables sans recompiler la logique.
- L'état de la configuration est centralisé et conservé entre les étapes du parcours.

---

## Calcul du prix

### Format de `tarifs.json` (valeurs d'exemple)

```json
{
  "tva": 0.20,
  "categories": {
    "citadine":   { "coveringTotal": 1800 },
    "berline":    { "coveringTotal": 2200 },
    "suv":        { "coveringTotal": 2600 },
    "utilitaire": { "coveringTotal": 3000 }
  },
  "types": {
    "total":   { "prix": "categorie", "delai": "3 à 4 jours" },
    "partiel": { "prix": 950,         "delai": "1 à 2 jours" },
    "toit":    { "prix": 420,         "delai": "1 jour" }
  },
  "finitions": {
    "brillant": 0.00,
    "satine":   0.10,
    "mat":      0.15,
    "chrome":   0.60
  },
  "options": {
    "ppf":       690,
    "vitres":    290,
    "detailing": 180,
    "depose":    250
  }
}
```

### Règle de calcul

```
base      = prix du type (ou prix de la catégorie pour un covering total)
covering  = base × (1 + majoration de la finition)
totalTTC  = covering + Σ options cochées
totalHT   = totalTTC / (1 + TVA)
TVA       = totalTTC − totalHT
```

Les prix sont exprimés TTC. Les montants sont arrondis à l'euro pour l'affichage et au centime sur le devis.

**Exemple :** berline, covering total satiné, PPF et vitres → 2 200 × 1,10 + 690 + 290 = **3 400 € TTC** (2 833,33 € HT).

Le moteur est couvert par des tests unitaires avant toute interface.

---

## Aperçu 3D

- **Un modèle générique par catégorie** (citadine, berline, SUV, utilitaire) au format `.glb`. Le client voit la silhouette correspondant à son véhicule, pas le modèle exact, ce qui évite l'achat de modèles sous licence pour chaque voiture.
- **Contrainte sur les modèles :** la carrosserie doit être un mesh séparé des vitres, roues et optiques, pour pouvoir changer sa couleur indépendamment.
- **Finitions via le matériau PBR :**

| Finition | `roughness` | `metalness` |
|---|---|---|
| Brillant | basse (~0,1) | 0 – 0,2 |
| Satiné | moyenne (~0,4) | 0 – 0,2 |
| Mat | haute (~0,8) | 0 |
| Chrome | très basse (~0,05) | 1 |

- Contrôles : rotation à la souris ou au doigt, vues prédéfinies (profil, 3/4 avant, arrière, dessus).
- Performance : modèles compressés (Draco), chargement différé, image de remplacement pendant le chargement.
- Tant que les modèles ne sont pas disponibles, une forme simple sert de remplacement.
- Les modèles 3D utilisés doivent avoir une licence autorisant l'usage commercial.

---

## Modèle de données

```ts
type Categorie = "citadine" | "berline" | "suv" | "utilitaire";
type Finition  = "brillant" | "satine" | "mat" | "chrome";
type TypeCovering = "total" | "partiel" | "toit";
type StatutDevis = "envoye" | "accepte" | "a_relancer" | "refuse";

interface Vehicule {
  marque: string;
  modele: string;
  annee: number;
  categorie: Categorie;
}

interface Configuration {
  vehicule: Vehicule;
  couleur: { id: string; nom: string; hex: string };
  finition: Finition;
  type: TypeCovering;
  options: string[];            // ids de tarifs.options
  etatCarrosserie?: string;     // déclaratif client
  photos?: string[];            // URLs des photos envoyées
}

interface Devis {
  numero: string;               // ex. DEV-2026-0001
  dateEmission: string;         // ISO 8601
  validiteJours: number;        // 30
  configuration: Configuration;
  lignes: { libelle: string; montantTTC: number }[];
  totalHT: number;
  tva: number;
  totalTTC: number;
  delai: string;
  statut: StatutDevis;
  origine: "site" | "comptoir";
  client?: { nom: string; email: string; telephone: string; consentement: boolean };
}
```

---

## Design

Thème sombre façon studio automobile.

| Rôle | Couleur |
|---|---|
| Fond | `#0F1012` |
| Cartes | `#17191C` |
| Bordures | `#2C3036` |
| Texte | `#F2F1EE` |
| Texte secondaire | `#A3A7AD` |
| Accent | `#FF7A2F` (texte `#0F1012` sur l'accent) |
| Document de devis | fond `#F7F6F3`, texte `#17191C` |

- Police : **Archivo** (Google Fonts).
- Coins arrondis 12 à 20 px, boutons en pilule.
- Accessibilité : cibles tactiles ≥ 44 px, contrastes ≥ 4,5:1, vrais éléments `<button>`, `<label>` et `<input>`.

---

## Sécurité et RGPD

- **Consentement explicite** (case non cochée par défaut) avant tout enregistrement des coordonnées, avec lien vers la politique de confidentialité.
- Collecte minimale : nom, e-mail, téléphone, configuration du véhicule.
- Durée de conservation définie pour les devis non signés, suppression ou anonymisation au-delà.
- Espace pro protégé par authentification ; aucune donnée client accessible publiquement.
- Aucune clé d'API ou secret dans le code front ni dans le dépôt : variables d'environnement côté serveur uniquement.
- Photos des clients stockées dans un espace privé, accessibles uniquement à l'atelier.
- Mentions légales et politique de confidentialité sur le site.

---

## Feuille de route

Chaque étape est validée (application lancée et testée) avant de passer à la suivante.

- [ ] **1. Moteur de prix** — fonction de calcul + tests unitaires, sans interface.
- [ ] **2. Écran Véhicule** — sélection marque / modèle / année et catégorie.
- [ ] **3. Écran Personnalisation sans 3D** — couleurs, finition, options, prix en direct.
- [ ] **4. Aperçu 3D** — modèles par catégorie, couleur et finition en direct.
- [ ] **5. Écran Devis** — document, PDF, formulaire RGPD ; choix et mise en place du back-end.
- [ ] **6. Espace pro** — authentification, liste et filtres des devis, devis au comptoir.
- [ ] **7. Page d'accueil** — présentation de l'atelier et accès au simulateur.
- [ ] **8. Mise en ligne** — vérifications sécurité, RGPD, performances et affichage mobile.

Évolutions possibles : envoi de photos de carrosserie, prise de rendez-vous en ligne, grille de tarifs éditable depuis l'espace pro, planning de l'atelier.

---

## Méthode de développement

Le projet est développé avec un assistant de code IA, en petites itérations.

- `CLAUDE.md` à la racine du code : règles de travail (modifications ciblées, réponses courtes, solution la plus simple).
- `CONTEXT.md` (50 lignes maximum) : état du projet, fichiers clés, décisions, prochaine étape. Mis à jour à la fin de chaque étape pour pouvoir reprendre dans une nouvelle session sans tout réexpliquer.
- Une fonctionnalité par itération, testée avant de passer à la suivante.

---

## Droits

© 2026 — Tous droits réservés.

Ce dépôt est public à des fins de présentation uniquement. Aucune licence n'est accordée : la documentation, la conception et le code associé ne peuvent être copiés, modifiés ou réutilisés sans autorisation écrite de l'auteur.
