# Design Document: RaceTelemetry

> Date: 2026-03-06
> Auteur: Sebastien Gousseau + Claude
> Statut: Approuve

## 1. Vision

App Flutter full-stack de telemetrie pour Le Mans Ultimate (LMU).
Dark theme racing futuriste avec effets neon/glassmorphism.
3 modes : overlay live, dashboard second ecran, analyse post-session avec coaching IA.
Monetise par credits AI (modele adriencoaching).

## 2. Stack Technique

| Couche | Technologie |
|--------|-------------|
| Frontend | Flutter (Windows desktop + Web) |
| 3D Circuit | CustomPainter + vector_math (projection perspective manuelle) |
| Capture | dart:ffi -> rF2 Shared Memory (Windows only) |
| Backend | Firebase (Auth, Firestore, Cloud Functions) |
| AI | Claude API via Cloud Functions |
| Paiement | Stripe Checkout + credits SG |

### SG Packages

| Package | Usage |
|---------|-------|
| sg_core | Result<T>, SgFailure, ErrorHandler |
| sg_auth | FirebaseAuthAdapter, auth providers |
| sg_ui | Composants UI, SgResultBuilder, animations, racing theme |
| sg_config | Environnements dev/prod |
| sg_cloud | Cloud Functions adapter |
| sg_data | Repositories, DatabaseManager (SQLite sessions locales) |
| sg_responsive | Layout desktop/web adaptatif |
| sg_feature_flags | Feature gates (beta features) |
| sg_analytics | Tracking usage |

## 3. Donnees Telemetrie LMU

### Sources

| Source | Refresh | Contenu |
|--------|---------|---------|
| Shared Memory Telemetry | 50 FPS | Vehicule joueur : position 3D, inputs, moteur, pneus, suspension, carburant, aero |
| Shared Memory Scoring | 5 FPS | TOUS les vehicules : position, vitesse, temps secteurs, classement |
| DuckDB natif (v1.2+) | Post-session | Export SQL complet |

### Donnees joueur (Telemetry buffer)

- Position monde XYZ (metres)
- Inputs : throttle, brake, steering, clutch (raw + filtre)
- Moteur : RPM, torque, temperatures eau/huile, gear, turbo
- Hybride : battery %, motor torque/RPM/temp, regen state
- 4 roues : temperature 3 points + carcasse, pression, usure, charge, grip, forces lat/long
- Freins : temperature, pression par roue
- Suspensions : deflection, ride height, force pushrod, camber, toe
- Aero : drag, downforce F/R, DRS flaps
- Carburant : niveau, capacite
- Degats : 8 zones, impacts

### Donnees tous vehicules (Scoring buffer)

- Position 3D, vitesse, acceleration, orientation
- Temps secteurs (best/last/current S1, S2, lap)
- Classement, ecarts, tours, pitstops, penalites

### Acces technique

- Windows shared memory via dart:ffi (memory-mapped files rF2)
- Buffer names : $rFactor2SMMP_Telemetry$, $rFactor2SMMP_Scoring$, etc.
- Synchronisation via versionBegin/versionEnd dans le header

## 4. Architecture Features

```
lib/features/
  capture/              # Capture shared memory (desktop only)
    domain/
      telemetry_frame.dart
      scoring_frame.dart
      wheel_data.dart
      vehicle_state.dart
      session_info.dart
    services/
      shared_memory_service.dart    # FFI -> rF2 shared memory
      capture_service.dart          # Orchestration capture
      session_recorder.dart         # Enregistrement -> SQLite
    providers/
      capture_providers.dart

  session/              # Gestion sessions enregistrees
    domain/
      session.dart
      lap.dart
      sector.dart
    services/
      session_repository.dart       # SQLite local
      session_sync_service.dart     # Sync -> Firestore
    providers/

  circuit_3d/           # Visualisation 3D du circuit
    domain/
      circuit_point.dart            # x, y, z, trackWidth
      trajectory_point.dart         # x, y, z + speed, throttle, brake
      camera_state.dart             # azimuth, elevation, distance, target
      circuit_model.dart
    presentation/
      circuit_3d_view.dart          # GestureDetector + CustomPaint
      painters/
        circuit_painter.dart        # Rendu piste
        trajectory_painter.dart     # Trajectoires colorees
        heatmap_painter.dart        # Overlay heatmap
    controllers/
      orbit_camera.dart             # Drag->rotation, scroll->zoom
    utils/
      projection.dart               # MVP matrix, project3D->2D
      color_map.dart                # Speed -> gradient color

  dashboard/            # Dashboard live + post-session
    presentation/
      live_dashboard_screen.dart
      post_session_screen.dart
      widgets/
        gauge_cluster.dart          # RPM + vitesse + fuel
        tire_monitor.dart           # 4 pneus live
        input_trace.dart            # Throttle/brake trace
        lap_table.dart
        sector_comparison.dart
        position_tracker.dart

  analysis/             # Analyse comparative
    domain/
      comparison.dart
      delta_analysis.dart
    services/
      comparison_service.dart
    presentation/
      comparison_screen.dart
      widgets/
        speed_trace_overlay.dart
        braking_points_view.dart
        sector_delta_chart.dart

  ai_coach/             # Coaching IA (credits)
    domain/
      coaching_report.dart
      coaching_request.dart
      improvement_tip.dart
    services/
      ai_coach_service.dart         # Cloud Function -> Claude
    presentation/
      coaching_screen.dart
      widgets/
        tip_card.dart
        session_summary.dart

  setup_advisor/        # Conseiller setup (credits)
    domain/
      setup_recommendation.dart
      setup_context.dart
    services/
      setup_service.dart            # Cloud Function -> Claude

  credits/              # Systeme credits (modele adriencoaching)
    domain/
      credit_account.dart
      credit_pack.dart
      credit_transaction.dart
    services/
      credit_service.dart
    providers/
      credit_providers.dart

  auth/                 # sg_auth standard

  profile/              # Profil pilote
    domain/
      driver_profile.dart
    presentation/
      profile_screen.dart
```

## 5. Upgrades sg_ui - Racing Theme

### Nouveaux composants

```
sg_ui/lib/src/
  effects/
    sg_glass_container.dart         # BackdropFilter + semi-transparent bg
    sg_neon_border.dart             # Bordure luminescente animee
    sg_glow_box.dart                # Multi-layer glow (outer + inner + pulse)
    sg_gradient_container.dart      # LinearGradient/RadialGradient tokens
  racing/
    sg_gauge.dart                   # Jauge circulaire animee (RPM, vitesse)
    sg_mini_chart.dart              # Sparkline chart anime
    sg_telemetry_bar.dart           # Barre horizontale temps reel (throttle/brake)
    sg_sector_indicator.dart        # Indicateur secteurs (vert/jaune/violet)
    sg_lap_delta.dart               # Delta temps avec couleur animee
    sg_tire_widget.dart             # Vue 4 pneus (temp heatmap 3 zones)
  theme/
    sg_racing_color_scheme.dart     # Palette neon racing
    sg_racing_typography.dart       # Typo avec text-shadow glow
```

### Palette Racing

```dart
// Fond
darkVoid: Color(0xFF0A0A0F)
darkSurface: Color(0xFF12121A)
darkElevated: Color(0xFF1A1A2E)

// Neon accents
neonCyan: Color(0xFF00F0FF)         // Vitesse
neonMagenta: Color(0xFFFF00E5)      // Freinage
neonGreen: Color(0xFF00FF88)        // Amelioration
neonYellow: Color(0xFFFFE500)       // Warning
neonOrange: Color(0xFFFF6B00)       // Alerte

// Heatmap
heatCold: Color(0xFF0044FF)         // Lent
heatWarm: Color(0xFFFFFF00)         // Moyen
heatHot: Color(0xFFFF0000)          // Rapide
```

### Effets visuels a ajouter

- Glassmorphism : BackdropFilter(ImageFilter.blur) + container semi-transparent
- Neon glow : Multi-layer BoxShadow avec animation pulse
- Gradients : LinearGradient/RadialGradient sur containers et textes
- Text glow : TextShadow avec blur neon
- Hover glow : Animation sur hover pour elements interactifs

## 6. Circuit 3D - Rendu natif Flutter

### Approche : CustomPainter + vector_math

- Projection perspective manuelle (pas de Transform widget avec perspective)
- Camera orbit : coordonnees spheriques (azimuth, elevation, distance)
- vector_math : makePerspectiveMatrix + makeViewMatrix -> MVP matrix
- 5000 points XYZ = trivial pour Canvas
- canvas.drawRawPoints pour nuages de points
- canvas.drawLine avec couleur par segment pour heatmap
- Fonctionne identiquement Windows desktop et Web

### Construction du circuit

Le circuit est construit automatiquement a partir des donnees de position :
1. Premier tour complet -> capture tous les mPos a 50 FPS
2. Lissage par spline cubique
3. Largeur de piste estimee depuis les ecarts lateraux des trajectoires
4. Cache en SQLite pour les sessions suivantes sur le meme circuit

### Trajectoires

- Chaque tour = liste de TrajectoryPoint (x, y, z, speed, throttle, brake, gear)
- Overlay sur le circuit 3D avec code couleur (speed heatmap)
- Comparaison visuelle : trajectoire joueur vs AI la plus rapide vs meilleur tour perso

## 7. Features AI et Credits

| Feature | Credits | Donnees envoyees a Claude |
|---------|---------|---------------------------|
| Coach post-session | 3/session | Temps secteurs, inputs moyennes, ecarts vs best, points de freinage |
| Setup advisor | 5/demande | Piste, meteo, style pilotage, donnees pneus/suspension historiques |
| Analyse comparative | 2/analyse | Delta trajectoire player vs AI, zones de perte |

### Flux Cloud Function

1. Flutter appelle Cloud Function
2. Cloud Function verifie auth + credits suffisants
3. Appel Claude API avec contexte telemetrie
4. Si succes -> consumeCredits()
5. Retour rapport structure

## 8. Stockage

| Donnee | Stockage | Raison |
|--------|----------|--------|
| Sessions brutes | SQLite local (sg_data) | Volume important, pas besoin cloud |
| Meilleurs tours | Firestore | Sync cross-device, partage |
| Circuits (points 3D) | SQLite local + cache | Construits depuis premiere session |
| Profil pilote | Firestore | Auth-linked |
| Credits | Firestore | Temps reel, securise |
| Rapports AI | Firestore | Historique consultable |

## 9. Ecrans principaux

| Ecran | Mode | Description |
|-------|------|-------------|
| Live Dashboard | Desktop overlay / second screen | Gauges, pneus, inputs, delta, position |
| Post-Session | Desktop + Web | Timeline tour par tour, analyse complete |
| Circuit 3D | Desktop + Web | Vue 3D, trajectoires, heatmaps, comparaison |
| AI Coach | Desktop + Web | Rapport coaching, tips, progression |
| Setup Lab | Desktop + Web | Recommandations setup, historique |
| Credits | Desktop + Web | Achat packs, historique transactions |
| Profile | Desktop + Web | Stats globales, circuits, progression |

## 10. Integration CrewChief (Extension future)

Possibilite d'integration vocale AI avec CrewChief :
- CrewChief est open source (C#, MIT license)
- Expose une architecture de plugins pour ajouter des messages vocaux custom
- RaceTelemetry pourrait fournir des tips AI en temps reel via un plugin CrewChief
- Communication inter-process : named pipes ou TCP local entre Flutter app et plugin C#
- Exemple : "Tu perds du temps au virage 5, essaie de freiner 10 metres plus tard"

## 11. Conventions

- Clean Architecture 4 couches (domain/services/providers/presentation)
- Modeles immutables (@immutable, const, copyWith)
- Services retournent Result<T> via ErrorHandler.guardAsync()
- Widgets = primitives uniquement (String, int, callbacks)
- Riverpod pour state management
- Conventional commits : feat(scope): message
- Worktrees pour features isolees
- Feature Manifest + Sprint State pour tracking
