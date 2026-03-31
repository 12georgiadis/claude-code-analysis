# Plan de Communication : Claude Code Leaked Source + Buddy System

## 1. ASSETS VISUELS IMMÉDIATS (à produire)

### Les 18 Buddy Sprites (ASCII -> Vector -> Animation)

Chaque espèce a 3 frames d'animation, des yeux interchangeables (·, ✦, ×, ◉, @, °), et des chapeaux optionnels.

**Pipeline de production** :
1. ASCII Art -> SVG (script de conversion automatique)
2. SVG -> Animation Lottie/GIF (3 frames idle loop)
3. Animation -> Video clips pour réseaux sociaux
4. Version haute-résolution via IA (Flux/SDXL stylisation)

### Espèces par rareté (pour le storytelling)

| Rareté | % | Couleur | Espèces potentielles | Usage com |
|--------|---|---------|---------------------|-----------|
| Common | 60% | Gris | duck, goose, blob, snail, turtle | "Everyone starts here" |
| Uncommon | 25% | Vert | cat, owl, rabbit, mushroom | "Getting interesting" |
| Rare | 10% | Bleu/violet | penguin, ghost, axolotl | "Now we're talking" |
| Epic | 4% | Orange/or | dragon, octopus, robot | "The chosen ones" |
| Legendary | 1% | Rouge/warning | capybara, cactus, chonk | "The myth" |

*Note: le capybara est nommé d'après le codename interne du modèle Claude 4.6*

### Les 8 chapeaux (accessoires)
```
crown    : \^^^/     (royauté)
tophat   : [___]     (classe)
propeller: -+-       (fun/geek)
halo     : (   )     (divin)
wizard   : /^\       (magie)
beanie   : (___)     (chill)
tinyduck : ,>        (meta-easter egg, 1% chance de shiny!)
none     : (rien)
```

### Les 5 stats (personnalité)
- DEBUGGING (compétence technique)
- PATIENCE (tolérance)
- CHAOS (imprévisibilité)
- WISDOM (sagesse)
- SNARK (sarcasme)

---

## 2. STRATÉGIE CONTENU X/TWITTER

### Thread Principal (8 tweets) — PRÊT À POSTER
Voir le thread préparé dans la session Claude Code.

### Threads Additionnels (à programmer)

**Thread 2 : "Meet the Buddies"** (viral potential max)
- Présenter chaque espèce avec son sprite animé
- "Which Claude Code Buddy are you?"
- Sondage : quelle espèce préférez-vous?
- Les stats comme des cartes Pokémon

**Thread 3 : "The System Prompt Revealed"**
- Comparaison internal vs external prompt
- Les règles cachées que Claude suit
- "Here's exactly what Claude thinks about before answering you"

**Thread 4 : "Claude Code Dreams While You Sleep"**
- AutoDream explained
- L'IA qui consolide ses mémoires pendant la nuit
- Angle philosophique : qu'est-ce que ça signifie qu'une IA "rêve"?

**Thread 5 : "AI Can Now Pay For Things"**
- x402 crypto payment protocol
- Claude Code + USDC on Base
- Machine-to-machine commerce is here

**Thread 6 : "Anthropic's Secret Open Source Strategy"**
- Undercover mode decoded
- Comment les employés Anthropic contribuent sans laisser de trace

### Format vidéo (Reels/TikTok/Shorts)
- **"All 18 Claude Code Pets"** : animation loop de chaque espèce (15-30s)
- **"Legendary Buddy unboxing"** : simuler l'ouverture d'un buddy légendaire
- **"What's really in Claude's system prompt"** : screen recording avec annotations

---

## 3. SITE WEB / LANDING PAGE

### Option A : Page statique sur Vercel (rapide)
- Gallery des 18 buddies avec animation
- Simulateur "What's your Buddy?" (hash du nom utilisateur)
- Countdown vers la feature officielle
- Lien vers le rapport GitHub

### Option B : App interactive (plus ambitieux)
- Buddy viewer 3D (Three.js, convertir ASCII en voxels)
- "Hatch your Buddy" experience interactive
- Leaderboard des rarétés
- NFT-style cards des buddies (sans blockchain, juste le visuel)

### Option C : Plugin Claude Code (meta)
- Un plugin qui active et personnalise le Buddy system
- Distribué via le marketplace
- Marketing : "Get your Buddy before everyone else"

---

## 4. CALENDRIER DE PUBLICATION

### Jour 1 (Aujourd'hui, 31 mars)
- [x] Rapport GitHub publié
- [ ] Thread principal X (8 tweets)
- [ ] Extraction sprites terminée

### Jour 2 (1er avril — BUDDY DAY!)
*Note: Anthropic avait prévu le teaser Buddy pour le 1er avril!*
- [ ] Thread "Meet the Buddies" avec animations
- [ ] Vidéo courte des 18 espèces
- [ ] Post "April Fools? No, it's real"

### Semaine 1
- [ ] Thread system prompt
- [ ] Thread AutoDream
- [ ] Landing page Vercel
- [ ] Vidéo "What's in Claude's System Prompt"

### Semaine 2
- [ ] Thread x402 crypto
- [ ] Thread Undercover mode
- [ ] App interactive Buddy viewer
- [ ] Article long format (blog)

---

## 5. ANGLES NARRATIFS

### Pour la tech community
- "512K lines of TypeScript analyzed"
- "Every feature flag, every hidden command"
- Focus sur l'architecture et les patterns

### Pour le grand public IA
- "Your AI assistant has a secret pet"
- "The AI that dreams while you sleep"
- "Your AI can pay for things with crypto"

### Pour les journalistes/media
- "Anthropic's internal vs external quality bar"
- "How Anthropic employees secretly contribute to open source"
- "The next Claude model is codenamed Numbat"
- Credentials exposées (GrowthBook keys, Datadog token, staging URL)

### Pour les créatifs/artistes
- Les sprites comme oeuvre ASCII art
- Le système de rareté comme commentaire sur la gamification
- L'IA qui rêve comme concept artistique

---

## 6. PRODUCTION D'ASSETS

### Scripts à créer
1. `ascii-to-svg.py` : Convertir les sprites ASCII en SVG vectoriel
2. `buddy-animator.py` : Générer des GIF/MP4 des animations idle
3. `buddy-card-generator.py` : Créer des cartes style Pokémon/trading card
4. `buddy-hash-simulator.html` : Page web "What's your Buddy?"

### Outils
- **SVG** : script Python avec les coordonnées de chaque caractère
- **Animation** : Lottie via script After Effects ou programmatique
- **Vidéo** : Remotion (React) ou ffmpeg pour les clips courts
- **IA upscale** : Flux/SDXL pour des versions haute résolution des sprites
- **3D** : Three.js voxel conversion pour le site web

---

## 7. MÉTRIQUES DE SUCCÈS

- Impressions thread principal : cible 100K+
- GitHub stars : cible 500+ première semaine
- Followers gagnés : cible 200+
- Reposts/likes thread Buddy : cible viralité (1K+ reposts)
- Trafic landing page : cible 5K+ première semaine
