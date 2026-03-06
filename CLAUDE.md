# RaceTelemetry — Claude Code Project Guide

## Overview
App Flutter de telemetrie pour Le Mans Ultimate (LMU). Capture live, visualisation 3D circuit, coaching IA, analyse comparative. Monetise par credits AI.

## Commands

### Build & Run
```bash
flutter pub get                          # Resoudre dependances
flutter run -d windows                   # Run Windows (avec capture live)
flutter run -d chrome                    # Run Web (analyse post-session)
flutter run -t lib/main_dev.dart         # Run avec emulators Firebase
flutter analyze                          # Analyse statique
flutter test                             # Tests
```

## Architecture

### SG Framework Pattern
Chaque feature suit le pattern SG avec separation stricte :
```
feature/
  domain/        # Entites, ports — pure Dart
  services/      # Implementations — retournent Result<T>
  providers/     # Riverpod providers
  presentation/  # Screens (ConsumerWidget), widgets (primitives)
```

### SG Packages (path deps -> ../sg-packages/packages/*)
| Package | Usage |
|---------|-------|
| sg_core | Result<T>, SgFailure, ErrorHandler |
| sg_auth | FirebaseAuthAdapter, authStateProvider |
| sg_data | DatabaseManager, SgBaseRepository |
| sg_ui | Composants UI, racing theme, effets visuels |
| sg_config | Environnements dev/prod |
| sg_cloud | Cloud Functions adapter |
| sg_feature_flags | Feature gates |
| sg_responsive | Layout desktop/web |
| sg_analytics | Tracking usage |

### Key Conventions
- Services retournent Result<T> via ErrorHandler.guardAsync()
- Widgets presentation = primitives (String, callbacks, pas de providers)
- Domain = pure Dart (pas d'import Flutter/Firebase)
- SQLite pour sessions locales via DatabaseManager
- Firestore pour donnees partagees (profil, credits, rapports AI)
- 3D via CustomPainter + vector_math (projection perspective manuelle)
- Capture telemetrie via dart:ffi -> rF2 Shared Memory (Windows only)

### Palette Racing
- darkVoid (#0A0A0F), darkSurface (#12121A), darkElevated (#1A1A2E)
- neonCyan (#00F0FF), neonMagenta (#FF00E5), neonGreen (#00FF88)
- neonYellow (#FFE500), neonOrange (#FF6B00)

### Firebase
- Auth (email + Google)
- Firestore (users, credits, rapports AI)
- Cloud Functions (AI coach, setup advisor, credits)
- Stripe Checkout pour achat credits
