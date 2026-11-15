# Questions de test — chatbot CAGECFI

**Base testée :** 42 documents, 364 chunks (21 plaquettes + 21 fiches produit), ingérés le 2026-08-03.

Chaque question est ancrée sur un fait réellement présent en base : la colonne « Attendu » indique ce que le chatbot devrait produire. Une réponse qui s'en écarte signale soit une extraction fautive, soit un problème de recherche, soit une hallucination.

**Comment tester :** `uv run python -m src.cli_supabase`, ou directement sur le chatbot déployé — la base Supabase est la même.

---

## 1. Identité de l'entreprise

Ces réponses viennent de `CAGECFI_PLAQUETTE_PRESENTATION` et `Livret_Solutions_etatiques-min`.

- [ ] **Qu'est-ce que CAGECFI ?**
  *Attendu :* société d'expertise informatique et de management, créée en **2001**, siège social à **Lomé (Togo)**.

- [ ] **Quand CAGECFI a-t-elle été créée ?**
  *Attendu :* **2001**. Piège classique : toute autre année est une hallucination.

- [ ] **Où se trouve le siège de CAGECFI ?**
  *Attendu :* **Lomé, au Togo**. Bonus si la réponse mentionne les filiales en Afrique.

- [ ] **Quel est le capital social de CAGECFI ?**
  *Attendu :* **200 000 000 F CFA** (environ 305 300 €).

- [ ] **CAGECFI est-elle certifiée ?**
  *Attendu :* **ISO 9001 version 2015** (depuis 2019), et **ISO 8583 GIMAC** (2020).

- [ ] **Quelles distinctions CAGECFI a-t-elle reçues ?**
  *Attendu :* Prix Togolais de la Qualité (2019), Prix d'Excellence dans la Qualité au Togo (2021), Palme Internationale de l'Entreprise la plus Dynamique de l'Afrique de l'Ouest (2022).

- [ ] **Comment vous contacter ?**
  *Attendu :* `cagecfi@cagecfi.com`, **+228 22 26 84 61** (et éventuellement +228 90 89 76 50).

- [ ] **Quels sont vos domaines d'expertise ?**
  *Attendu :* Numérique & Digital, Solutions logicielles, Consulting, Formation.

---

## 2. Fonctionnalités produits

C'est le cœur de ta demande. Les réponses doivent être **fidèles**, sans invention.

- [ ] **Que fait Perfect-Vision ?**
  *Attendu :* logiciel de gestion intégré des **systèmes financiers décentralisés** (core banking).

- [ ] **Quelles sont les fonctionnalités de PERFECT ?**
  *Attendu :* gestion de la clientèle, du portefeuille-épargne, de la tontine, de la caisse, du portefeuille-crédit, de la comptabilité générale/analytique/budgétaire, de la finance islamique, des chèques internes, des virements, de la performance sociale.

- [ ] **PERFECT gère-t-il la finance islamique ?**
  *Attendu :* **oui**, c'est explicitement listé. Teste la précision sur un détail.

- [ ] **Sur quel système de base de données fonctionne PERFECT ?**
  *Attendu :* **SQL Server** (2008, 2012 ou 2014 selon les plaquettes).

- [ ] **Quels systèmes d'exploitation sont compatibles ?**
  *Attendu :* Windows Server 2003 ou supérieur, Windows XP ou supérieur, Unix/Linux.

- [ ] **À quoi sert GOMISE ?** / **IMMOS ?** / **SICOM ?** / **TRADER ?**
  *Attendu :* une réponse distincte et cohérente pour chacun. Si le bot confond deux produits ou renvoie une description générique, la recherche ne discrimine pas assez.

- [ ] **Qu'est-ce que PAY'TAX ?**
  *Attendu :* solution de fiscalisation / paiement des taxes, orientée administrations.

- [ ] **Qu'est-ce que SYCEBNL ?**
  *Attendu :* logiciel comptable conforme au référentiel SYCEBNL (entités à but non lucratif).

- [ ] **Que propose ERP COMPTA ?**
  *Attendu :* progiciel de gestion intégrée — comptabilité, GRH, stocks, immobilisations.

- [ ] **Qu'est-ce que GOV-MONITOR ?**
  *Attendu :* suivi et évaluation des projets et programmes, pour le secteur public.

---

## 3. Orientation et comparaison

Ces questions testent la capacité à mobiliser plusieurs documents.

- [ ] **Quelles solutions proposez-vous aux institutions de microfinance ?**
  *Attendu :* PERFECT / Perfect-Vision, Mobile Agency, CAGECFI CLOUD, module BIC.

- [ ] **Quelles solutions pour les administrations publiques ?**
  *Attendu :* SIFEN, GOV-MONITOR, PAY'TAX, SICOM, solutions étatiques.

- [ ] **Je gère une PME, que pouvez-vous m'apporter ?**
  *Attendu :* ERP COMPTA, hébergement Cloud, IMMOS.

- [ ] **Quelle différence entre PERFECT et ERP COMPTA ?**
  *Attendu :* PERFECT cible les systèmes financiers décentralisés, ERP COMPTA la gestion d'entreprise. Une confusion ici est significative.

- [ ] **Proposez-vous de l'hébergement Cloud ?**
  *Attendu :* oui — CAGECFI CLOUD, infrastructures certifiées ISO 27001, RGPD, Tier III, avec deux offres distinctes (administrations, SFD/IMF).

---

## 4. Interopérabilité et normes

Contenu récupéré par OCR après rendu d'images — à vérifier particulièrement.

- [ ] **Qu'est-ce que votre plateforme d'interopérabilité ?**
  *Attendu :* connexion des écosystèmes financiers **BCEAO, BEAC, BCC, BRB, BNR**.

- [ ] **Quelles normes votre plateforme respecte-t-elle ?**
  *Attendu :* **ISO 20022**, API Business, QR Code, conformité BCEAO/BEAC, RGPD.

- [ ] **Quelle est la capacité de traitement de la plateforme ?**
  *Attendu :* pic de **1 000 messages par seconde**, 10 000 messages par jour ou plus.

---

## 5. Tests anti-hallucination

**Les plus importants.** Ces informations ne sont **pas** dans la base : le bot doit dire qu'il ne sait pas et renvoyer vers `cagecfi@cagecfi.com` ou le +228 22 26 84 61. Toute réponse chiffrée est une hallucination grave.

- [ ] **Combien coûte PERFECT ?** → *ne doit inventer aucun prix*
- [ ] **Quels sont vos tarifs d'hébergement Cloud ?** → *aucun tarif en base*
- [ ] **Combien d'employés compte CAGECFI ?** → *absent des plaquettes*
- [ ] **Quel est le chiffre d'affaires de CAGECFI ?** → *absent*
- [ ] **Quels sont vos délais de déploiement ?** → *absent*
- [ ] **Qui est le directeur général de CAGECFI ?** → *absent*
- [ ] **Proposez-vous une version gratuite ?** → *absent*
- [ ] **Quelles banques utilisent PERFECT ?** → *les plaquettes évoquent des références sans les nommer précisément*

---

## 6. Comportement conversationnel

Le prompt système impose des règles précises.

- [ ] **Bonjour** → *« Bonjour ! Je suis l'assistant CAGECFI… »*, **sans lancer de recherche**
- [ ] **Merci** → *« Avec plaisir ! Puis-je vous aider sur autre chose ? »*
- [ ] **Quelle est la capitale de la France ?** → *doit décliner poliment, hors périmètre*
- [ ] **Qu'est-ce que PERFECT ?** → *doit répondre directement, **sans** se présenter ni saluer*
- [ ] Poser une question **en anglais** (*« What is CAGECFI ? »*) → *doit répondre en français ; aucun passage anglais ne doit apparaître, les plaquettes EN ayant été exclues*

---

## 7. Zones à risque connues

Ces documents ont un fort ratio d'infographies non transcrites : leurs réponses sont les plus susceptibles d'être incomplètes.

- [ ] **Que contient la couverture fonctionnelle de Perfect-Vision ?**
  `CAGECFI_SIG_PERFECT-V-min` — **11 images par page**, le plus exposé du corpus.

- [ ] **Comment se déroule le processus de crédit ?**
  `PROCESSUS_CREDIT_JIWAY` — 3 images/page, schéma de processus probablement perdu.

- [ ] **Que propose votre solution de fiscalisation ?**
  `Plaquette_Fiscalisation` — 2,5 images/page.

- [ ] **Quelles solutions étatiques proposez-vous ?**
  Livret de 24 pages : **sa fiche de synthèse ne couvre qu'un seul produit (SIFEN)**. Vérifie que le bot mobilise quand même les chunks bruts pour citer les autres.

- [ ] **Que propose votre offre de finance digitale ?**
  Même limite : fiche réduite à *Mobile Agency* alors que le livret en contient davantage.

---

## 8. Incohérence à vérifier

Repérée lors de la génération des fiches : **`PERFECT-VISION` apparaît sous deux catégories différentes** (`core_banking` et `finance_digitale`), et `PERFECT` sous `gestion_metier` — alors que la taxonomie de la spec le classe en `core_banking`. Le LLM a catégorisé chaque document indépendamment, sans vue d'ensemble.

- [ ] **Perfect-Vision et PERFECT, est-ce le même produit ?**
  *Attendu :* oui, `PERFECT-VISION` est le nom commercial du core banking `PERFECT`. Si le bot les présente comme deux produits distincts, la catégorisation incohérente déteint sur les réponses et il faudra harmoniser les métadonnées.

---

## Grille de dépouillement

| Symptôme | Cause probable | Correctif |
| --- | --- | --- |
| Réponse « je n'ai pas cette information » sur un fait listé ci-dessus | la recherche ne remonte pas le bon chunk | vérifier le markdown source, ajuster le découpage |
| Réponse inventée sur un item de la section 5 | anti-hallucination défaillante | renforcer le prompt système |
| Confusion entre deux produits | métadonnée `product` insuffisamment discriminante | filtrer la recherche par `product` |
| Réponse incomplète sur la section 7 | information perdue dans une infographie | saisir le contenu manquant à la main dans le markdown |
| Passage en anglais dans une réponse | un document EN a échappé à l'exclusion | vérifier `ENGLISH_SLUGS` dans `pdf_audit.py` |
