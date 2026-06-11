# Active Thermoregulation: JOS-3 Simulation

A wearable device that heats or cools a patient's back to help keep body temperature stable. This project simulates a closed-loop control system on a thermophysiological model of the human body, tested against three heat-sensitive patient profiles.

![Python](https://img.shields.io/badge/Python-3.12-3776AB?logo=python&logoColor=white)
![Jupyter](https://img.shields.io/badge/Jupyter-Notebook-F37626?logo=jupyter&logoColor=white)
![JOS-3](https://img.shields.io/badge/model-JOS--3-0E8C8C)
![Status](https://img.shields.io/badge/status-simulation%20prototype-lightgrey)

Project PRICE 39, Safran Tech / MicroPower.

**[Français](#français)** · **[English](#english)**

---

## Français

### Le problème

Certaines personnes ne régulent pas bien leur température corporelle, et pour elles quelques dixièmes de degré comptent.

- **Sclérose en plaques (SEP)** : une légère hausse de la température centrale peut aggraver temporairement des symptômes comme la fatigue, les troubles visuels et la faiblesse. C'est le phénomène d'Uhthoff, qui peut être déclenché par quelque chose d'aussi anodin qu'une séance de kinésithérapie.
- **Hémodialyse** : le sang circule dans un circuit extérieur au corps et s'y refroidit. Au fil d'une séance de plusieurs heures, le patient perd de la chaleur et peut se mettre à frissonner.
- **Sport d'endurance** : un effort intense au soleil fait monter la température centrale. Au-delà d'un certain point, la performance chute et le risque de coup de chaleur augmente.

Aujourd'hui, ces situations sont surtout subies. Le but de ce projet est de les anticiper.

### Notre approche

Un dispositif porté sur le dos, capable de chauffer ou de refroidir jusqu'à 50 W dans chaque sens, piloté par un capteur de température cutanée placé sur le thorax.

Pourquoi un capteur cutané sur le thorax plutôt que la température centrale directement ? La température du sang central (Tcb) est la grandeur qui compte physiologiquement, mais ce n'est pas quelque chose qu'un objet porté au quotidien peut mesurer. La température de la peau, si. La vraie question est donc de savoir si la température cutanée suit le corps d'assez près pour servir de signal de pilotage.

Avant de construire le moindre matériel, nous avons testé cette idée en simulation.

### Comment fonctionne la simulation

#### Le modèle du corps : JOS-3

Nous utilisons [JOS-3](https://github.com/TanabeLab/JOS-3), un modèle thermophysiologique open source qui découpe le corps en 17 segments (tête, cou, thorax, dos, bassin, bras, mains, cuisses, jambes, pieds, etc). À chaque pas de temps, il calcule la température de peau par segment et la température centrale pour l'ensemble du corps, en tenant compte de la transpiration, des frissons, de la vasoconstriction et de la vasodilatation.

Les entrées sont un profil (taille, poids, masse grasse, âge, sexe), des vêtements (isolation thermique par segment) et un environnement (température de l'air, rayonnement, humidité, vent, niveau d'activité).

#### Boucle de régulation

Chaque minute simulée, le programme répète le même cycle :

1. Lecture de la température cutanée du thorax.
2. Le contrôleur décide d'un niveau de puissance.
3. Cette puissance est appliquée sur le dos.
4. JOS-3 fait avancer le corps d'une minute.
5. Toutes les températures sont enregistrées, puis on recommence.

#### Loi de commande

Le contrôleur est une simple fonction en paliers :

- Entre les deux seuils, le dispositif est éteint (zone neutre).
- Au-dessus du seuil haut, il refroidit : 5 W de plus par tranche de 0,1 °C de dépassement.
- En dessous du seuil bas, il chauffe, selon la même logique.
- La puissance est plafonnée à 50 W, la limite de notre matériel.

Les seuils par défaut sont 34,8 °C (refroidissement) et 33,0 °C (chauffe). Pour le scénario SEP, les seuils sont abaissés à 33,4 et 32,0 °C, car ces patients doivent être protégés plus tôt.

<p align="center">
  <img src="figures/loi_de_commande.png" width="80%" alt="Loi de commande par paliers">
</p>

### Trois scénarios

| | Sportif | Patient SEP | Patient dialysé |
|---|---|---|---|
| Profil | Homme, 28 ans, 72 kg | Femme, 45 ans, 65 kg | Homme, 68 ans, 75 kg |
| Situation | Course au soleil, pause à l'ombre, sprint final | Séance de kinésithérapie en intérieur | Séance d'hémodialyse assise |
| Durée | 55 min | 60 min | 190 min |
| Risque visé | Surchauffe | Phénomène d'Uhthoff | Refroidissement |
| Particularité | Fort rayonnement solaire, effort intense | Seuils abaissés | Perte de 50 W au niveau central pendant la séance, pour représenter le refroidissement du sang dans le circuit |

<details>
<summary>Détail des phases de chaque scénario</summary>

**Sportif**

| Phase | Durée | Air | Rayonnement | Activité |
|---|---|---|---|---|
| Course intense au soleil | 30 min | 32 °C | 45 °C | x6 |
| Arrêt à l'ombre | 10 min | 24 °C | 24 °C | x1,2 |
| Sprint final | 15 min | 30 °C | 38 °C | x8 |

**Patient SEP** (pièce à 24 °C)

| Phase | Durée | Activité | Posture |
|---|---|---|---|
| Repos avant séance | 5 min | x1,0 | assis |
| Échauffement léger | 10 min | x2,0 | debout |
| Exercices soutenus | 35 min | x4,0 | debout |
| Récupération | 10 min | x1,2 | assis |

**Patient dialysé** (assis, activité au repos)

| Phase | Durée | Air | Perte centrale |
|---|---|---|---|
| Installation | 10 min | 22 °C | aucune |
| Début de séance | 60 min | 21 °C | 50 W |
| Mi-séance (frissons) | 90 min | 21 °C | 50 W |
| Fin de séance | 30 min | 22 °C | aucune |

L'activité est exprimée en multiple du métabolisme de repos (PAR, Physical Activity Ratio).

</details>

### Résultats

Chaque scénario produit une figure en cinq panneaux : température de peau par segment avec la température centrale (Tcb) en noir épais (en haut), température centrale comparée à la température moyenne de la peau (milieu gauche), puissance du dispositif minute par minute, refroidissement en rouge et chauffe en bleu (milieu droite), segments classés par corrélation avec la température centrale (bas gauche), et température centrale superposée aux 5 segments les mieux classés (bas droite). Seul le côté gauche du corps est affiché, le corps étant symétrique.

**Sportif** : le dispositif refroidit pendant 41 des 55 minutes, à pleine puissance dès le début de la course, en relâchant à l'ombre, puis à fond pendant le sprint. La température de peau du dos descend jusqu'à 29,9 °C. Malgré cela, la température centrale grimpe de 36,7 à 38,1 °C : face à un effort aussi intense au soleil, 50 W sur un seul segment du dos ne suffisent pas.

<p align="center"><img src="figures/scenario_sport.png" width="90%" alt="Simulation du scénario sportif"></p>

**Patient SEP** : grâce aux seuils abaissés, le dispositif tourne à pleine puissance dès le début, puis à nouveau pendant les exercices soutenus (42 minutes de refroidissement au total). La température centrale culmine à 37,36 °C, restant sous le seuil de 37,5 °C retenu pour le phénomène d'Uhthoff. La température de peau du dos descend en revanche jusqu'à 24,7 °C en début de séance, ce qui pose une question de confort.

<p align="center"><img src="figures/scenario_sep.png" width="90%" alt="Simulation du scénario SEP"></p>

**Patient dialysé** : le cas inverse. Le dispositif chauffe pendant 181 des 190 minutes, presque toujours au plafond de 50 W. La température centrale baisse lentement de 37,1 à 36,6 °C. Un point d'attention : la température de peau du dos atteint 45,4 °C sous une chauffe prolongée sur un seul segment, ce qui présente un risque de brûlure si maintenu pendant des heures. Un dispositif réel devrait répartir la puissance sur plusieurs zones et inclure un arrêt de sécurité au-delà d'une certaine température cutanée.

<p align="center"><img src="figures/scenario_dialyse.png" width="90%" alt="Simulation du scénario dialyse"></p>

**Bilan**

| Scénario | Tcb (min à max) | Refroidissement | Chauffe | Inactif | Énergie thermique échangée* |
|---|---|---|---|---|---|
| Sportif | 36,69 à 38,06 °C | 41 min | 0 min | 14 min | ≈ 18 Wh |
| SEP | 36,79 à 37,36 °C | 42 min | 0 min | 18 min | ≈ 12 Wh |
| Dialyse | 36,63 à 37,14 °C | 0 min | 181 min | 9 min | ≈ 148 Wh |

<sub>* Somme des puissances appliquées sur la peau au cours de la séance. Il s'agit de chaleur échangée, pas de consommation électrique, qui sera plus élevée et dépend du rendement du module (Peltier ou autre).</sub>

### Où placer le capteur

Le notebook classe chaque segment selon la corrélation de Pearson entre sa température cutanée et la température centrale sur toute la simulation. Un score proche de 1 signifie que la peau de ce segment suit de plus près la température centrale.

| Scénario | Top 3 segments | Score du thorax (capteur actuel) |
|---|---|---|
| Sportif | main 0,94, pied 0,91, tête 0,83 | 0,39 |
| SEP | main 0,92, pied 0,89, bras 0,89 | 0,01 |
| Dialyse | pied 0,998, bras 0,995, main 0,993 | 0,91 |

Deux constats ressortent. Les extrémités (mains, pieds) suivent le mieux la température centrale dans les trois scénarios, sans doute parce que la circulation sanguine y varie beaucoup selon l'état thermique du corps. Le thorax, où se trouve notre capteur actuel, est mal corrélé dans deux scénarios sur trois, ce qui est un vrai sujet pour la suite du projet.

Ce score est à interpréter avec prudence. Il ignore le décalage temporel entre peau et température centrale, ainsi que le bruit d'un vrai capteur. Il est aussi gonflé quand toutes les températures évoluent dans le même sens : dans le scénario dialyse, presque tous les segments dépassent 0,9 simplement parce que tout le corps se refroidit ensemble. Les mains et les pieds bougent aussi beaucoup au quotidien, ce qui en fait des emplacements peu pratiques pour un capteur.

### Limites et pistes

Ce travail n'est pas une validation clinique. Ce sont des simulations sur un modèle, avec des scénarios et des seuils que nous avons choisis nous-mêmes.

Limites actuelles :

- Un seul segment (le dos) est actionné, ce qui produit des températures locales extrêmes (24,7 et 45,4 °C).
- Les seuils sont réglés à la main, séparément pour chaque scénario.
- Le signal du capteur n'est pas filtré, alors que des changements rapides d'environnement provoquent des variations rapides de la température cutanée.
- Le notebook ne compare pas encore chaque scénario à la même situation sans dispositif, ce qui est pourtant nécessaire pour mesurer son effet réel.
- Le code s'appuie sur une méthode interne de JOS-3 (`_set_ex_q`) pour appliquer la chaleur, d'où la version figée dans `requirements.txt`.

Pistes pour la suite :

- Ajouter une simulation de référence sans dispositif pour chaque scénario
- Répartir la puissance sur plusieurs segments
- Ajouter un arrêt de sécurité basé sur la température cutanée locale
- Filtrer le signal du capteur
- Tester un régulateur plus fin, par exemple PI
- Affiner le choix de l'emplacement du capteur en tenant compte du décalage temporel et du bruit

### Lancer le projet

Nécessite Python 3.10 ou plus récent.

```bash
git clone https://github.com/AdemmBr/Thermoregulation-project.git
cd Thermoregulation-project
pip install -r requirements.txt
jupyter notebook simulation_thermoregulation.ipynb
```

Exécuter toutes les cellules. Les trois simulations prennent quelques secondes.

Organisation du dépôt :

```
.
├── README.md
├── simulation_thermoregulation.ipynb   (modèle, contrôleur, scénarios, graphiques)
├── requirements.txt
└── figures/
```

### Référence

Takahashi, Y., Nomoto, A., Yoda, S., Hisayama, R., Ogata, M., Ozeki, Y., & Tanabe, S. (2021). Thermoregulation model JOS-3 with new open source code. Energy and Buildings, 231, 110575.

### Équipe

Réalisé dans le cadre de PRICE 39, porté par Safran Tech via l'incubateur MicroPower.

---

## English

### The problem

Some people don't regulate body temperature well, and for them a few tenths of a degree matter.

- **Multiple sclerosis (MS)**: a small rise in core temperature can temporarily worsen symptoms such as fatigue, vision problems and weakness. This is Uhthoff's phenomenon, and it can be triggered by something as mild as a physiotherapy session.
- **Hemodialysis**: blood circulates through an external circuit and cools there. Over a multi-hour session, patients lose heat and can start shivering.
- **Endurance sport**: intense effort in the sun raises core temperature. Past a certain point, performance drops and heat stroke risk rises.

Right now these situations are mostly just endured. The point of this project is to anticipate them instead.

### Approach

A device worn on the back, able to heat or cool by up to 50 W in either direction, controlled by a skin temperature sensor on the chest.

Why a chest skin sensor instead of core temperature directly? Core blood temperature (Tcb) is what matters physiologically, but it isn't something an everyday wearable can measure. Skin temperature is. So the real question is whether skin temperature tracks the body closely enough to be a useful control signal.

Before building any hardware, we tested this in simulation.

### How the simulation works

#### Body model: JOS-3

We use [JOS-3](https://github.com/TanabeLab/JOS-3), an open-source thermophysiological model that splits the body into 17 segments (head, neck, chest, back, pelvis, arms, hands, thighs, legs, feet, etc). At each time step it computes skin temperature per segment and core temperature for the whole body, accounting for sweating, shivering, vasoconstriction and vasodilation.

Inputs are a profile (height, weight, body fat, age, sex), clothing (thermal insulation per segment) and an environment (air temperature, radiation, humidity, wind, activity level).

#### Control loop

Every simulated minute, the program repeats the same cycle:

1. Read chest skin temperature.
2. The controller decides on a power level.
3. That power is applied to the back.
4. JOS-3 advances the body by one minute.
5. All temperatures are logged, then it repeats.

#### Control law

The controller is a simple step function:

- Between the two thresholds, the device is off (neutral zone).
- Above the high threshold, it cools: 5 W more per 0.1 degC of overshoot.
- Below the low threshold, it heats, using the same logic.
- Power is capped at 50 W, the limit of our hardware.

Default thresholds are 34.8 degC (cooling) and 33.0 degC (heating). For the MS scenario, thresholds are lowered to 33.4 and 32.0 degC, since these patients need to be protected earlier.

<p align="center">
  <img src="figures/loi_de_commande.png" width="80%" alt="Step control law">
</p>

### Three scenarios

| | Athlete | MS patient | Dialysis patient |
|---|---|---|---|
| Profile | Male, 28, 72 kg | Female, 45, 65 kg | Male, 68, 75 kg |
| Situation | Run in the sun, shade break, final sprint | Indoor physiotherapy session | Seated hemodialysis session |
| Duration | 55 min | 60 min | 190 min |
| Risk targeted | Overheating | Uhthoff's phenomenon | Cooling |
| Notes | Strong solar radiation, intense effort | Lowered thresholds | 50 W core heat loss during the session, simulating blood cooling in the circuit |

<details>
<summary>Phase-by-phase detail for each scenario</summary>

**Athlete**

| Phase | Duration | Air | Radiation | Activity |
|---|---|---|---|---|
| Intense run in the sun | 30 min | 32 degC | 45 degC | x6 |
| Shade break | 10 min | 24 degC | 24 degC | x1.2 |
| Final sprint | 15 min | 30 degC | 38 degC | x8 |

**MS patient** (room at 24 degC)

| Phase | Duration | Activity | Posture |
|---|---|---|---|
| Rest before session | 5 min | x1.0 | sitting |
| Light warm-up | 10 min | x2.0 | standing |
| Sustained exercise | 35 min | x4.0 | standing |
| Recovery | 10 min | x1.2 | sitting |

**Dialysis patient** (seated, resting activity)

| Phase | Duration | Air | Core loss |
|---|---|---|---|
| Setup | 10 min | 22 degC | none |
| Session start | 60 min | 21 degC | 50 W |
| Mid-session (shivering) | 90 min | 21 degC | 50 W |
| Session end | 30 min | 22 degC | none |

Activity is expressed as a multiple of resting metabolism (PAR, Physical Activity Ratio).

</details>

### Results

Each scenario produces a five-panel figure: skin temperature per segment with core temperature (Tcb) in bold black (top), core vs. mean skin temperature (middle left), device power minute by minute, cooling in red and heating in blue (middle right), segments ranked by correlation with core temperature (bottom left), and core temperature against the top 5 ranked segments (bottom right). Only the left side of the body is shown, since it's symmetric.

**Athlete**: the device cools for 41 of 55 minutes, running at full power at the start of the run, easing off in the shade, then maxing out again during the sprint. Back skin temperature drops to 29.9 degC. Even so, core temperature climbs from 36.7 to 38.1 degC: against effort this intense in the sun, 50 W on one back segment isn't enough.

<p align="center"><img src="figures/scenario_sport.png" width="90%" alt="Athlete scenario simulation"></p>

**MS patient**: with lowered thresholds, the device runs at full power from the start, then again during sustained exercise (42 minutes of cooling total). Core temperature peaks at 37.36 degC, staying under the 37.5 degC threshold we set for Uhthoff's phenomenon. Back skin temperature drops to 24.7 degC early in the session, which raises a comfort question.

<p align="center"><img src="figures/scenario_sep.png" width="90%" alt="MS scenario simulation"></p>

**Dialysis patient**: the reverse case. The device heats for 181 of 190 minutes, almost always at the 50 W cap. Core temperature drifts down slowly from 37.1 to 36.6 degC. One concern: back skin temperature reaches 45.4 degC under sustained heating on a single segment, which is a burn risk if held for hours. A real device would need to spread power across multiple zones and include a safety cutoff above some skin temperature limit.

<p align="center"><img src="figures/scenario_dialyse.png" width="90%" alt="Dialysis scenario simulation"></p>

**Summary**

| Scenario | Tcb (min to max) | Cooling | Heating | Off | Thermal energy exchanged* |
|---|---|---|---|---|---|
| Athlete | 36.69 to 38.06 degC | 41 min | 0 min | 14 min | ~18 Wh |
| MS | 36.79 to 37.36 degC | 42 min | 0 min | 18 min | ~12 Wh |
| Dialysis | 36.63 to 37.14 degC | 0 min | 181 min | 9 min | ~148 Wh |

<sub>* Sum of power applied to the skin during the session. This is heat exchanged, not electrical consumption, which will be higher and depends on the module's efficiency (Peltier or otherwise).</sub>

### Where to place the sensor

The notebook ranks each segment by Pearson correlation between its skin temperature and core temperature over the full simulation. A score closer to 1 means that segment's skin tracks core temperature more closely.

| Scenario | Top 3 segments | Chest score (current sensor) |
|---|---|---|
| Athlete | hand 0.94, foot 0.91, head 0.83 | 0.39 |
| MS | hand 0.92, foot 0.89, arm 0.89 | 0.01 |
| Dialysis | foot 0.998, arm 0.995, hand 0.993 | 0.91 |

Two takeaways stand out. Extremities (hands, feet) track core temperature best across all three scenarios, likely because blood flow there varies a lot with the body's thermal state. The chest, where our current sensor sits, correlates poorly in two of the three scenarios, which is a real problem for the next phase of the project.

This score should be read with caution. It ignores the time lag between skin and core temperature and the noise of a real sensor. It's also inflated whenever temperatures move together: in the dialysis scenario, nearly every segment scores above 0.9 simply because the whole body is cooling at once. Hands and feet also move a lot in daily life, which makes them impractical sensor locations.

### Limitations and next steps

This is not clinical validation. These are simulations on a model, with scenarios and thresholds we chose ourselves.

Current limitations:

- Only one segment (the back) is actuated, which produces extreme local temperatures (24.7 and 45.4 degC).
- Thresholds are hand-tuned separately for each scenario.
- The sensor signal isn't filtered, even though fast environment changes cause rapid swings in skin temperature.
- The notebook doesn't yet compare each scenario to the same situation without the device, which is needed to measure its actual effect.
- The code relies on an internal JOS-3 method (`_set_ex_q`) to apply heat, hence the pinned version in `requirements.txt`.

Next steps:

- Add a baseline simulation without the device for each scenario
- Spread power across multiple segments
- Add a safety cutoff based on local skin temperature
- Filter the sensor signal
- Try a finer controller, e.g. PI
- Refine sensor placement accounting for time lag and noise

### Running the project

Requires Python 3.10 or later.

```bash
git clone https://github.com/AdemmBr/Thermoregulation-project.git
cd Thermoregulation-project
pip install -r requirements.txt
jupyter notebook simulation_thermoregulation.ipynb
```

Run all cells. The three simulations take a few seconds.

Repo layout:

```
.
├── README.md
├── simulation_thermoregulation.ipynb   (model, controller, scenarios, plots)
├── requirements.txt
└── figures/
```

### Reference

Takahashi, Y., Nomoto, A., Yoda, S., Hisayama, R., Ogata, M., Ozeki, Y., & Tanabe, S. (2021). Thermoregulation model JOS-3 with new open source code. Energy and Buildings, 231, 110575.

### Team

Built as part of PRICE 39, run by Safran Tech through the MicroPower incubator.
