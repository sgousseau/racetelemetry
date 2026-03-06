# RaceTelemetry Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a Flutter telemetry app for Le Mans Ultimate with live capture, 3D circuit visualization, and AI coaching monetized via credits.

**Architecture:** Full Flutter (Windows desktop + Web), SG packages framework, CustomPainter 3D rendering, Firebase backend, dart:ffi shared memory capture, Claude API for AI coaching.

**Tech Stack:** Flutter, Riverpod, SG packages (sg_core/sg_ui/sg_auth/sg_data/sg_config/sg_cloud), Firebase, Stripe, dart:ffi, vector_math, Claude API.

---

## Phase 0: Project Bootstrap & SG Integration

### Task 0.1: Configure pubspec.yaml with SG packages

**Files:**
- Modify: `pubspec.yaml`

**Step 1: Update pubspec.yaml**

Replace the entire pubspec.yaml with proper SG package dependencies:

```yaml
name: racetelemetry
description: "Telemetry app for Le Mans Ultimate"
publish_to: 'none'
version: 0.1.0+1

environment:
  sdk: ^3.12.0-68.0.dev

dependencies:
  flutter:
    sdk: flutter

  # SG Framework
  sg_core:
    path: ../sg-packages/packages/sg_core
  sg_ui:
    path: ../sg-packages/packages/sg_ui
  sg_auth:
    path: ../sg-packages/packages/sg_auth
  sg_data:
    path: ../sg-packages/packages/sg_data
  sg_config:
    path: ../sg-packages/packages/sg_config
  sg_cloud:
    path: ../sg-packages/packages/sg_cloud
  sg_responsive:
    path: ../sg-packages/packages/sg_responsive
  sg_feature_flags:
    path: ../sg-packages/packages/sg_feature_flags
  sg_analytics:
    path: ../sg-packages/packages/sg_analytics

  # State management
  flutter_riverpod: ^2.6.1

  # Navigation
  go_router: ^14.8.1

  # Firebase
  firebase_core: ^3.12.1
  firebase_auth: ^5.5.1
  cloud_firestore: ^5.6.7
  cloud_functions: ^5.3.4

  # 3D rendering
  vector_math: ^2.1.4

  # Local DB
  sqflite: ^2.4.1
  path_provider: ^2.1.5
  path: ^1.9.1

  # FFI (shared memory)
  ffi: ^2.1.3
  win32: ^5.12.0

  # Utils
  intl: ^0.19.0
  collection: ^1.19.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^6.0.0
  mocktail: ^1.0.4

flutter:
  uses-material-design: true
  fonts:
    - family: JetBrainsMono
      fonts:
        - asset: assets/fonts/JetBrainsMono-Regular.ttf
        - asset: assets/fonts/JetBrainsMono-Bold.ttf
          weight: 700
```

**Step 2: Run pub get**

Run: `cd F:/Code/racetelemetry && flutter pub get`
Expected: Dependencies resolved successfully.

**Step 3: Commit**

```bash
git add pubspec.yaml pubspec.lock
git commit -m "chore: configure SG packages and dependencies"
```

---

### Task 0.2: App shell with Racing Theme & GoRouter

**Files:**
- Create: `lib/app.dart`
- Create: `lib/config/racing_theme.dart`
- Create: `lib/config/racing_colors.dart`
- Create: `lib/routing/app_router.dart`
- Modify: `lib/main.dart`

**Step 1: Create racing color scheme**

```dart
// lib/config/racing_colors.dart
import 'package:flutter/material.dart';

abstract final class RacingColors {
  // Backgrounds
  static const darkVoid = Color(0xFF0A0A0F);
  static const darkSurface = Color(0xFF12121A);
  static const darkElevated = Color(0xFF1A1A2E);

  // Neon accents
  static const neonCyan = Color(0xFF00F0FF);
  static const neonMagenta = Color(0xFFFF00E5);
  static const neonGreen = Color(0xFF00FF88);
  static const neonYellow = Color(0xFFFFE500);
  static const neonOrange = Color(0xFFFF6B00);
  static const neonRed = Color(0xFFFF003C);

  // Heatmap
  static const heatCold = Color(0xFF0044FF);
  static const heatWarm = Color(0xFFFFFF00);
  static const heatHot = Color(0xFFFF0000);

  // Sector colors
  static const sectorGreen = Color(0xFF00FF88);   // Personal best
  static const sectorPurple = Color(0xFFAA00FF);   // Overall best
  static const sectorYellow = Color(0xFFFFE500);   // Slower

  // Text
  static const textPrimary = Color(0xFFEEEEF0);
  static const textSecondary = Color(0xFF8888A0);
  static const textMuted = Color(0xFF555570);
}
```

**Step 2: Create racing theme builder**

```dart
// lib/config/racing_theme.dart
import 'package:flutter/material.dart';
import 'racing_colors.dart';

ThemeData buildRacingTheme() {
  final colorScheme = ColorScheme.dark(
    primary: RacingColors.neonCyan,
    secondary: RacingColors.neonMagenta,
    surface: RacingColors.darkSurface,
    error: RacingColors.neonRed,
    onPrimary: RacingColors.darkVoid,
    onSecondary: RacingColors.darkVoid,
    onSurface: RacingColors.textPrimary,
    onError: RacingColors.darkVoid,
    outline: RacingColors.textMuted,
  );

  return ThemeData(
    useMaterial3: true,
    colorScheme: colorScheme,
    scaffoldBackgroundColor: RacingColors.darkVoid,
    fontFamily: 'JetBrainsMono',
    cardTheme: CardThemeData(
      color: RacingColors.darkSurface,
      elevation: 0,
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(12),
        side: BorderSide(color: RacingColors.textMuted.withAlpha(40)),
      ),
    ),
    appBarTheme: const AppBarTheme(
      backgroundColor: RacingColors.darkVoid,
      foregroundColor: RacingColors.textPrimary,
      elevation: 0,
    ),
    navigationRailTheme: NavigationRailThemeData(
      backgroundColor: RacingColors.darkSurface,
      selectedIconTheme: const IconThemeData(color: RacingColors.neonCyan),
      unselectedIconTheme: const IconThemeData(color: RacingColors.textSecondary),
      indicatorColor: RacingColors.neonCyan.withAlpha(30),
    ),
  );
}
```

**Step 3: Create app router**

```dart
// lib/routing/app_router.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import '../features/dashboard/presentation/dashboard_shell.dart';

final appRouter = GoRouter(
  initialLocation: '/dashboard',
  routes: [
    ShellRoute(
      builder: (context, state, child) => DashboardShell(child: child),
      routes: [
        GoRoute(
          path: '/dashboard',
          builder: (context, state) => const Placeholder(), // TODO: LiveDashboardScreen
        ),
        GoRoute(
          path: '/sessions',
          builder: (context, state) => const Placeholder(), // TODO: SessionListScreen
        ),
        GoRoute(
          path: '/circuit',
          builder: (context, state) => const Placeholder(), // TODO: Circuit3DScreen
        ),
        GoRoute(
          path: '/coach',
          builder: (context, state) => const Placeholder(), // TODO: CoachingScreen
        ),
        GoRoute(
          path: '/profile',
          builder: (context, state) => const Placeholder(), // TODO: ProfileScreen
        ),
      ],
    ),
  ],
);
```

**Step 4: Create dashboard shell with navigation rail**

```dart
// lib/features/dashboard/presentation/dashboard_shell.dart
import 'package:flutter/material.dart';
import 'package:go_router/go_router.dart';
import '../../../config/racing_colors.dart';

class DashboardShell extends StatelessWidget {
  final Widget child;
  const DashboardShell({super.key, required this.child});

  static const _destinations = [
    NavigationRailDestination(
      icon: Icon(Icons.speed),
      selectedIcon: Icon(Icons.speed),
      label: Text('Live'),
    ),
    NavigationRailDestination(
      icon: Icon(Icons.history),
      selectedIcon: Icon(Icons.history),
      label: Text('Sessions'),
    ),
    NavigationRailDestination(
      icon: Icon(Icons.map),
      selectedIcon: Icon(Icons.map),
      label: Text('Circuit'),
    ),
    NavigationRailDestination(
      icon: Icon(Icons.psychology),
      selectedIcon: Icon(Icons.psychology),
      label: Text('Coach'),
    ),
    NavigationRailDestination(
      icon: Icon(Icons.person),
      selectedIcon: Icon(Icons.person),
      label: Text('Profil'),
    ),
  ];

  static const _routes = [
    '/dashboard',
    '/sessions',
    '/circuit',
    '/coach',
    '/profile',
  ];

  int _currentIndex(BuildContext context) {
    final location = GoRouterState.of(context).uri.path;
    final index = _routes.indexWhere((r) => location.startsWith(r));
    return index >= 0 ? index : 0;
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Row(
        children: [
          NavigationRail(
            selectedIndex: _currentIndex(context),
            onDestinationSelected: (i) => context.go(_routes[i]),
            labelType: NavigationRailLabelType.selected,
            leading: Padding(
              padding: const EdgeInsets.symmetric(vertical: 16),
              child: Text(
                'RT',
                style: TextStyle(
                  fontSize: 24,
                  fontWeight: FontWeight.bold,
                  color: RacingColors.neonCyan,
                  shadows: [
                    Shadow(color: RacingColors.neonCyan.withAlpha(120), blurRadius: 12),
                  ],
                ),
              ),
            ),
            destinations: _destinations,
          ),
          const VerticalDivider(thickness: 1, width: 1),
          Expanded(child: child),
        ],
      ),
    );
  }
}
```

**Step 5: Create app.dart**

```dart
// lib/app.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'config/racing_theme.dart';
import 'routing/app_router.dart';

class RaceTelemetryApp extends StatelessWidget {
  const RaceTelemetryApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp.router(
      title: 'RaceTelemetry',
      debugShowCheckedModeBanner: false,
      theme: buildRacingTheme(),
      routerConfig: appRouter,
    );
  }
}
```

**Step 6: Update main.dart**

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'app.dart';

void main() {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(
    const ProviderScope(
      child: RaceTelemetryApp(),
    ),
  );
}
```

**Step 7: Run the app**

Run: `cd F:/Code/racetelemetry && flutter run -d windows`
Expected: App launches with dark racing theme, navigation rail on left, placeholder content.

**Step 8: Commit**

```bash
git add lib/ assets/
git commit -m "feat(shell): app shell with racing theme and navigation"
```

---

## Phase 1: sg_ui Racing Theme Upgrades

### Task 1.1: Racing effects — SgGlassContainer

**Files:**
- Create: `F:/Code/sg-packages/packages/sg_ui/lib/src/effects/sg_glass_container.dart`
- Modify: `F:/Code/sg-packages/packages/sg_ui/lib/sg_ui.dart` (add export)
- Test: `F:/Code/sg-packages/packages/sg_ui/test/effects/sg_glass_container_test.dart`

**Step 1: Write the test**

```dart
// test/effects/sg_glass_container_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:sg_ui/sg_ui.dart';

void main() {
  testWidgets('SgGlassContainer renders with BackdropFilter', (tester) async {
    await tester.pumpWidget(
      MaterialApp(
        home: Stack(
          children: [
            Container(color: Colors.blue),
            const SgGlassContainer(
              child: Text('Glass content'),
            ),
          ],
        ),
      ),
    );

    expect(find.text('Glass content'), findsOneWidget);
    expect(find.byType(BackdropFilter), findsOneWidget);
  });

  testWidgets('SgGlassContainer applies border color', (tester) async {
    await tester.pumpWidget(
      MaterialApp(
        home: Stack(
          children: [
            SgGlassContainer(
              borderColor: Colors.cyan,
              child: const Text('Bordered'),
            ),
          ],
        ),
      ),
    );

    expect(find.text('Bordered'), findsOneWidget);
  });
}
```

**Step 2: Run test to verify it fails**

Run: `cd F:/Code/sg-packages/packages/sg_ui && flutter test test/effects/sg_glass_container_test.dart`
Expected: FAIL — SgGlassContainer not found.

**Step 3: Implement SgGlassContainer**

```dart
// lib/src/effects/sg_glass_container.dart
import 'dart:ui';
import 'package:flutter/material.dart';

class SgGlassContainer extends StatelessWidget {
  final Widget child;
  final double blurAmount;
  final Color backgroundColor;
  final Color? borderColor;
  final double borderRadius;
  final EdgeInsetsGeometry padding;

  const SgGlassContainer({
    super.key,
    required this.child,
    this.blurAmount = 12.0,
    this.backgroundColor = const Color(0x1AFFFFFF),
    this.borderColor,
    this.borderRadius = 12.0,
    this.padding = const EdgeInsets.all(16),
  });

  @override
  Widget build(BuildContext context) {
    return ClipRRect(
      borderRadius: BorderRadius.circular(borderRadius),
      child: BackdropFilter(
        filter: ImageFilter.blur(sigmaX: blurAmount, sigmaY: blurAmount),
        child: Container(
          padding: padding,
          decoration: BoxDecoration(
            color: backgroundColor,
            borderRadius: BorderRadius.circular(borderRadius),
            border: borderColor != null
                ? Border.all(color: borderColor!.withAlpha(80), width: 1)
                : null,
          ),
          child: child,
        ),
      ),
    );
  }
}
```

**Step 4: Add export to sg_ui.dart**

Add line: `export 'src/effects/sg_glass_container.dart';`

**Step 5: Run test to verify it passes**

Run: `cd F:/Code/sg-packages/packages/sg_ui && flutter test test/effects/sg_glass_container_test.dart`
Expected: PASS

**Step 6: Commit**

```bash
cd F:/Code/sg-packages
git add packages/sg_ui/lib/src/effects/sg_glass_container.dart packages/sg_ui/lib/sg_ui.dart packages/sg_ui/test/effects/sg_glass_container_test.dart
git commit -m "feat(sg_ui): add SgGlassContainer with glassmorphism effect"
```

---

### Task 1.2: Racing effects — SgNeonBorder

**Files:**
- Create: `F:/Code/sg-packages/packages/sg_ui/lib/src/effects/sg_neon_border.dart`
- Modify: `F:/Code/sg-packages/packages/sg_ui/lib/sg_ui.dart`
- Test: `F:/Code/sg-packages/packages/sg_ui/test/effects/sg_neon_border_test.dart`

**Step 1: Write the test**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:sg_ui/sg_ui.dart';

void main() {
  testWidgets('SgNeonBorder renders with glow effect', (tester) async {
    await tester.pumpWidget(
      const MaterialApp(
        home: SgNeonBorder(
          color: Colors.cyan,
          child: Text('Neon'),
        ),
      ),
    );

    expect(find.text('Neon'), findsOneWidget);
    // Should have a Container with BoxShadow
    final container = tester.widget<Container>(find.byType(Container).first);
    final decoration = container.decoration as BoxDecoration?;
    expect(decoration?.boxShadow, isNotNull);
  });

  testWidgets('SgNeonBorder animates when animated=true', (tester) async {
    await tester.pumpWidget(
      const MaterialApp(
        home: SgNeonBorder(
          color: Colors.cyan,
          animated: true,
          child: Text('Pulsing'),
        ),
      ),
    );

    expect(find.text('Pulsing'), findsOneWidget);
    // Advance animation
    await tester.pump(const Duration(milliseconds: 500));
    await tester.pump(const Duration(milliseconds: 500));
  });
}
```

**Step 2: Run test — expected FAIL**

**Step 3: Implement SgNeonBorder**

```dart
// lib/src/effects/sg_neon_border.dart
import 'package:flutter/material.dart';

class SgNeonBorder extends StatefulWidget {
  final Widget child;
  final Color color;
  final double borderRadius;
  final double glowSpread;
  final double glowBlur;
  final bool animated;
  final Duration animationDuration;

  const SgNeonBorder({
    super.key,
    required this.child,
    required this.color,
    this.borderRadius = 12.0,
    this.glowSpread = 2.0,
    this.glowBlur = 16.0,
    this.animated = false,
    this.animationDuration = const Duration(milliseconds: 2000),
  });

  @override
  State<SgNeonBorder> createState() => _SgNeonBorderState();
}

class _SgNeonBorderState extends State<SgNeonBorder>
    with SingleTickerProviderStateMixin {
  AnimationController? _controller;

  @override
  void initState() {
    super.initState();
    if (widget.animated) {
      _controller = AnimationController(
        vsync: this,
        duration: widget.animationDuration,
      )..repeat(reverse: true);
    }
  }

  @override
  void dispose() {
    _controller?.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (_controller != null) {
      return AnimatedBuilder(
        animation: _controller!,
        builder: (context, child) => _buildContainer(_controller!.value),
        child: widget.child,
      );
    }
    return _buildContainer(1.0);
  }

  Widget _buildContainer(double intensity) {
    final alpha = (80 * intensity).round();
    final blur = widget.glowBlur * (0.5 + 0.5 * intensity);

    return Container(
      decoration: BoxDecoration(
        borderRadius: BorderRadius.circular(widget.borderRadius),
        border: Border.all(
          color: widget.color.withAlpha(alpha + 40),
          width: 1.5,
        ),
        boxShadow: [
          BoxShadow(
            color: widget.color.withAlpha(alpha),
            blurRadius: blur,
            spreadRadius: widget.glowSpread * intensity,
          ),
          BoxShadow(
            color: widget.color.withAlpha((alpha * 0.3).round()),
            blurRadius: blur * 2,
            spreadRadius: widget.glowSpread * 2 * intensity,
          ),
        ],
      ),
      child: ClipRRect(
        borderRadius: BorderRadius.circular(widget.borderRadius),
        child: widget.child,
      ),
    );
  }
}
```

**Step 4: Add export, run test — PASS**

**Step 5: Commit**

```bash
git commit -m "feat(sg_ui): add SgNeonBorder with animated glow effect"
```

---

### Task 1.3: Racing widgets — SgGauge

**Files:**
- Create: `F:/Code/sg-packages/packages/sg_ui/lib/src/racing/sg_gauge.dart`
- Modify: `F:/Code/sg-packages/packages/sg_ui/lib/sg_ui.dart`
- Test: `F:/Code/sg-packages/packages/sg_ui/test/racing/sg_gauge_test.dart`

**Step 1: Write the test**

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:sg_ui/sg_ui.dart';

void main() {
  testWidgets('SgGauge renders with value and label', (tester) async {
    await tester.pumpWidget(
      const MaterialApp(
        home: SgGauge(
          value: 0.75,
          maxValue: 1.0,
          label: 'RPM',
          displayValue: '7500',
          size: 150,
          color: Colors.cyan,
        ),
      ),
    );

    expect(find.text('RPM'), findsOneWidget);
    expect(find.text('7500'), findsOneWidget);
    expect(find.byType(CustomPaint), findsWidgets);
  });

  testWidgets('SgGauge animates value change', (tester) async {
    await tester.pumpWidget(
      const MaterialApp(
        home: SgGauge(
          value: 0.0,
          maxValue: 1.0,
          label: 'Speed',
          displayValue: '0',
          size: 150,
          color: Colors.cyan,
        ),
      ),
    );

    await tester.pumpWidget(
      const MaterialApp(
        home: SgGauge(
          value: 0.9,
          maxValue: 1.0,
          label: 'Speed',
          displayValue: '280',
          size: 150,
          color: Colors.cyan,
        ),
      ),
    );

    // Animation in progress
    await tester.pump(const Duration(milliseconds: 150));
  });
}
```

**Step 2: Run test — FAIL**

**Step 3: Implement SgGauge**

```dart
// lib/src/racing/sg_gauge.dart
import 'dart:math' as math;
import 'package:flutter/material.dart';

class SgGauge extends StatefulWidget {
  final double value;
  final double maxValue;
  final String label;
  final String displayValue;
  final double size;
  final Color color;
  final Color? warningColor;
  final double warningThreshold;
  final double startAngle;
  final double sweepAngle;

  const SgGauge({
    super.key,
    required this.value,
    required this.maxValue,
    required this.label,
    required this.displayValue,
    required this.size,
    required this.color,
    this.warningColor,
    this.warningThreshold = 0.85,
    this.startAngle = 135,
    this.sweepAngle = 270,
  });

  @override
  State<SgGauge> createState() => _SgGaugeState();
}

class _SgGaugeState extends State<SgGauge> with SingleTickerProviderStateMixin {
  late AnimationController _controller;
  late Animation<double> _animation;
  double _previousValue = 0;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 300),
    );
    _animation = Tween(begin: 0.0, end: widget.value / widget.maxValue)
        .animate(CurvedAnimation(parent: _controller, curve: Curves.easeOutCubic));
    _controller.forward();
  }

  @override
  void didUpdateWidget(SgGauge oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (oldWidget.value != widget.value) {
      _previousValue = _animation.value;
      _animation = Tween(
        begin: _previousValue,
        end: (widget.value / widget.maxValue).clamp(0.0, 1.0),
      ).animate(CurvedAnimation(parent: _controller, curve: Curves.easeOutCubic));
      _controller
        ..reset()
        ..forward();
    }
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
      animation: _animation,
      builder: (context, _) {
        final fraction = _animation.value;
        final isWarning = fraction >= widget.warningThreshold;
        final activeColor = isWarning ? (widget.warningColor ?? Colors.red) : widget.color;

        return SizedBox(
          width: widget.size,
          height: widget.size,
          child: Stack(
            alignment: Alignment.center,
            children: [
              CustomPaint(
                size: Size(widget.size, widget.size),
                painter: _GaugePainter(
                  fraction: fraction,
                  color: activeColor,
                  startAngle: widget.startAngle,
                  sweepAngle: widget.sweepAngle,
                ),
              ),
              Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  Text(
                    widget.displayValue,
                    style: TextStyle(
                      fontSize: widget.size * 0.22,
                      fontWeight: FontWeight.bold,
                      color: activeColor,
                      shadows: [
                        Shadow(color: activeColor.withAlpha(100), blurRadius: 8),
                      ],
                    ),
                  ),
                  Text(
                    widget.label,
                    style: TextStyle(
                      fontSize: widget.size * 0.09,
                      color: Colors.white70,
                    ),
                  ),
                ],
              ),
            ],
          ),
        );
      },
    );
  }
}

class _GaugePainter extends CustomPainter {
  final double fraction;
  final Color color;
  final double startAngle;
  final double sweepAngle;

  _GaugePainter({
    required this.fraction,
    required this.color,
    required this.startAngle,
    required this.sweepAngle,
  });

  @override
  void paint(Canvas canvas, Size size) {
    final center = Offset(size.width / 2, size.height / 2);
    final radius = size.width / 2 - 12;
    final startRad = startAngle * math.pi / 180;
    final sweepRad = sweepAngle * math.pi / 180;

    // Background arc
    final bgPaint = Paint()
      ..color = color.withAlpha(25)
      ..strokeWidth = 6
      ..style = PaintingStyle.stroke
      ..strokeCap = StrokeCap.round;

    canvas.drawArc(
      Rect.fromCircle(center: center, radius: radius),
      startRad, sweepRad, false, bgPaint,
    );

    // Active arc
    final activePaint = Paint()
      ..color = color
      ..strokeWidth = 6
      ..style = PaintingStyle.stroke
      ..strokeCap = StrokeCap.round;

    canvas.drawArc(
      Rect.fromCircle(center: center, radius: radius),
      startRad, sweepRad * fraction, false, activePaint,
    );

    // Glow arc
    final glowPaint = Paint()
      ..color = color.withAlpha(40)
      ..strokeWidth = 14
      ..style = PaintingStyle.stroke
      ..strokeCap = StrokeCap.round
      ..maskFilter = const MaskFilter.blur(BlurStyle.normal, 8);

    canvas.drawArc(
      Rect.fromCircle(center: center, radius: radius),
      startRad, sweepRad * fraction, false, glowPaint,
    );

    // Tick marks
    final tickPaint = Paint()
      ..color = color.withAlpha(60)
      ..strokeWidth = 1;

    for (int i = 0; i <= 10; i++) {
      final angle = startRad + sweepRad * i / 10;
      final innerR = radius - 8;
      final outerR = radius + 4;
      canvas.drawLine(
        Offset(center.dx + innerR * math.cos(angle), center.dy + innerR * math.sin(angle)),
        Offset(center.dx + outerR * math.cos(angle), center.dy + outerR * math.sin(angle)),
        tickPaint,
      );
    }
  }

  @override
  bool shouldRepaint(_GaugePainter old) =>
      old.fraction != fraction || old.color != color;
}
```

**Step 4: Add export, run test — PASS**

**Step 5: Commit**

```bash
git commit -m "feat(sg_ui): add SgGauge racing widget with animated arc and neon glow"
```

---

### Task 1.4: Racing widgets — SgTelemetryBar, SgLapDelta, SgTireWidget

**Files:**
- Create: `F:/Code/sg-packages/packages/sg_ui/lib/src/racing/sg_telemetry_bar.dart`
- Create: `F:/Code/sg-packages/packages/sg_ui/lib/src/racing/sg_lap_delta.dart`
- Create: `F:/Code/sg-packages/packages/sg_ui/lib/src/racing/sg_tire_widget.dart`
- Create: `F:/Code/sg-packages/packages/sg_ui/lib/src/racing/sg_sector_indicator.dart`
- Modify: `F:/Code/sg-packages/packages/sg_ui/lib/sg_ui.dart`
- Test: `F:/Code/sg-packages/packages/sg_ui/test/racing/sg_racing_widgets_test.dart`

Follow same TDD pattern as Task 1.3. Each widget:

**SgTelemetryBar:** Horizontal bar showing throttle (green) or brake (red) with real-time value 0.0-1.0. Neon glow on the filled portion.

**SgLapDelta:** Shows +0.342 or -0.215 with animated color transition (green=faster, red=slower). Text with neon glow shadow.

**SgTireWidget:** 4-tire layout showing temperature heatmap (3 zones per tire: inner/center/outer). Uses RacingColors heatmap gradient. Pressure and wear % labels.

**SgSectorIndicator:** 3 colored boxes (S1/S2/S3) — green=PB, purple=overall best, yellow=slower. Animated flash on new best.

**Step 5: Commit**

```bash
git commit -m "feat(sg_ui): add SgTelemetryBar, SgLapDelta, SgTireWidget, SgSectorIndicator"
```

---

### Task 1.5: Racing widgets — SgMiniChart

**Files:**
- Create: `F:/Code/sg-packages/packages/sg_ui/lib/src/racing/sg_mini_chart.dart`
- Modify: `F:/Code/sg-packages/packages/sg_ui/lib/sg_ui.dart`
- Test: `F:/Code/sg-packages/packages/sg_ui/test/racing/sg_mini_chart_test.dart`

Sparkline chart widget. Takes `List<double>` data points, draws a smooth line with gradient fill underneath. Neon glow on the line. Used for lap time trends, fuel consumption, tire degradation.

**Commit:** `feat(sg_ui): add SgMiniChart sparkline with neon glow`

---

## Phase 2: Domain Models (Pure Dart)

### Task 2.1: Telemetry domain models

**Files:**
- Create: `lib/features/capture/domain/wheel_data.dart`
- Create: `lib/features/capture/domain/vehicle_state.dart`
- Create: `lib/features/capture/domain/telemetry_frame.dart`
- Create: `lib/features/capture/domain/scoring_frame.dart`
- Create: `lib/features/capture/domain/session_info.dart`
- Test: `test/features/capture/domain/vehicle_state_test.dart`

**Step 1: Write failing test for WheelData**

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:racetelemetry/features/capture/domain/wheel_data.dart';

void main() {
  test('WheelData creates with default values', () {
    const wheel = WheelData();
    expect(wheel.temperature, [0, 0, 0]);
    expect(wheel.pressure, 0);
    expect(wheel.wear, 0);
  });

  test('WheelData.avgTemperature computes correctly', () {
    const wheel = WheelData(temperature: [350, 360, 340]);
    expect(wheel.avgTemperature, closeTo(350, 0.1));
  });

  test('WheelData.copyWith preserves other fields', () {
    const wheel = WheelData(pressure: 180, wear: 0.95);
    final updated = wheel.copyWith(pressure: 185);
    expect(updated.pressure, 185);
    expect(updated.wear, 0.95);
  });
}
```

**Step 2: Implement all domain models**

```dart
// lib/features/capture/domain/wheel_data.dart
import 'package:flutter/foundation.dart';

@immutable
class WheelData {
  final List<double> temperature; // [inner, center, outer] in Kelvin
  final double carcassTemperature;
  final double pressure;          // kPa
  final double wear;              // 0.0-1.0
  final double load;              // N
  final double gripFraction;      // 0.0-1.0
  final double brakeTemp;         // C
  final double brakePressure;     // 0.0-1.0
  final double suspensionDeflection; // m
  final double rideHeight;        // m

  const WheelData({
    this.temperature = const [0, 0, 0],
    this.carcassTemperature = 0,
    this.pressure = 0,
    this.wear = 0,
    this.load = 0,
    this.gripFraction = 0,
    this.brakeTemp = 0,
    this.brakePressure = 0,
    this.suspensionDeflection = 0,
    this.rideHeight = 0,
  });

  double get avgTemperature =>
      temperature.isEmpty ? 0 : temperature.reduce((a, b) => a + b) / temperature.length;

  WheelData copyWith({
    List<double>? temperature,
    double? carcassTemperature,
    double? pressure,
    double? wear,
    double? load,
    double? gripFraction,
    double? brakeTemp,
    double? brakePressure,
    double? suspensionDeflection,
    double? rideHeight,
  }) {
    return WheelData(
      temperature: temperature ?? this.temperature,
      carcassTemperature: carcassTemperature ?? this.carcassTemperature,
      pressure: pressure ?? this.pressure,
      wear: wear ?? this.wear,
      load: load ?? this.load,
      gripFraction: gripFraction ?? this.gripFraction,
      brakeTemp: brakeTemp ?? this.brakeTemp,
      brakePressure: brakePressure ?? this.brakePressure,
      suspensionDeflection: suspensionDeflection ?? this.suspensionDeflection,
      rideHeight: rideHeight ?? this.rideHeight,
    );
  }
}
```

```dart
// lib/features/capture/domain/vehicle_state.dart
import 'package:flutter/foundation.dart';
import 'wheel_data.dart';

@immutable
class VehicleState {
  // Position
  final double posX, posY, posZ;
  final double speed;             // m/s

  // Inputs
  final double throttle;          // 0.0-1.0
  final double brake;             // 0.0-1.0
  final double steering;          // -1.0 to 1.0
  final double clutch;            // 0.0-1.0

  // Engine
  final double rpm;
  final double maxRpm;
  final int gear;                 // -1=R, 0=N, 1+=forward
  final double waterTemp;
  final double oilTemp;
  final double fuelLevel;         // litres
  final double fuelCapacity;

  // Wheels [FL, FR, RL, RR]
  final List<WheelData> wheels;

  // Aero
  final double frontDownforce;
  final double rearDownforce;

  // Timing
  final double lapDistance;       // m from start
  final double currentLapTime;   // seconds

  const VehicleState({
    this.posX = 0, this.posY = 0, this.posZ = 0,
    this.speed = 0,
    this.throttle = 0, this.brake = 0,
    this.steering = 0, this.clutch = 0,
    this.rpm = 0, this.maxRpm = 8000,
    this.gear = 0,
    this.waterTemp = 0, this.oilTemp = 0,
    this.fuelLevel = 0, this.fuelCapacity = 0,
    this.wheels = const [WheelData(), WheelData(), WheelData(), WheelData()],
    this.frontDownforce = 0, this.rearDownforce = 0,
    this.lapDistance = 0, this.currentLapTime = 0,
  });

  double get speedKmh => speed * 3.6;
  double get rpmFraction => maxRpm > 0 ? (rpm / maxRpm).clamp(0, 1) : 0;
  double get fuelFraction => fuelCapacity > 0 ? (fuelLevel / fuelCapacity).clamp(0, 1) : 0;

  VehicleState copyWith({
    double? posX, double? posY, double? posZ,
    double? speed,
    double? throttle, double? brake,
    double? steering, double? clutch,
    double? rpm, double? maxRpm,
    int? gear,
    double? waterTemp, double? oilTemp,
    double? fuelLevel, double? fuelCapacity,
    List<WheelData>? wheels,
    double? frontDownforce, double? rearDownforce,
    double? lapDistance, double? currentLapTime,
  }) {
    return VehicleState(
      posX: posX ?? this.posX, posY: posY ?? this.posY, posZ: posZ ?? this.posZ,
      speed: speed ?? this.speed,
      throttle: throttle ?? this.throttle, brake: brake ?? this.brake,
      steering: steering ?? this.steering, clutch: clutch ?? this.clutch,
      rpm: rpm ?? this.rpm, maxRpm: maxRpm ?? this.maxRpm,
      gear: gear ?? this.gear,
      waterTemp: waterTemp ?? this.waterTemp, oilTemp: oilTemp ?? this.oilTemp,
      fuelLevel: fuelLevel ?? this.fuelLevel, fuelCapacity: fuelCapacity ?? this.fuelCapacity,
      wheels: wheels ?? this.wheels,
      frontDownforce: frontDownforce ?? this.frontDownforce,
      rearDownforce: rearDownforce ?? this.rearDownforce,
      lapDistance: lapDistance ?? this.lapDistance,
      currentLapTime: currentLapTime ?? this.currentLapTime,
    );
  }
}
```

```dart
// lib/features/capture/domain/telemetry_frame.dart
import 'package:flutter/foundation.dart';
import 'vehicle_state.dart';

@immutable
class TelemetryFrame {
  final VehicleState player;
  final double timestamp;          // elapsed time in seconds

  const TelemetryFrame({
    required this.player,
    required this.timestamp,
  });
}
```

```dart
// lib/features/capture/domain/scoring_frame.dart
import 'package:flutter/foundation.dart';

@immutable
class ScoredVehicle {
  final String driverName;
  final String vehicleName;
  final String vehicleClass;
  final bool isPlayer;
  final int position;
  final int totalLaps;
  final double lapDistance;
  final double bestLapTime;
  final double lastLapTime;
  final double bestS1, bestS2;
  final double lastS1, lastS2;
  final double posX, posY, posZ;
  final double speed;
  final bool inPits;

  const ScoredVehicle({
    this.driverName = '',
    this.vehicleName = '',
    this.vehicleClass = '',
    this.isPlayer = false,
    this.position = 0,
    this.totalLaps = 0,
    this.lapDistance = 0,
    this.bestLapTime = 0, this.lastLapTime = 0,
    this.bestS1 = 0, this.bestS2 = 0,
    this.lastS1 = 0, this.lastS2 = 0,
    this.posX = 0, this.posY = 0, this.posZ = 0,
    this.speed = 0,
    this.inPits = false,
  });
}

@immutable
class ScoringFrame {
  final String trackName;
  final int sessionType;           // 0=test..13=race
  final double ambientTemp;
  final double trackTemp;
  final double raining;            // 0.0-1.0
  final List<ScoredVehicle> vehicles;
  final double timestamp;

  const ScoringFrame({
    this.trackName = '',
    this.sessionType = 0,
    this.ambientTemp = 0,
    this.trackTemp = 0,
    this.raining = 0,
    this.vehicles = const [],
    this.timestamp = 0,
  });

  ScoredVehicle? get player =>
      vehicles.where((v) => v.isPlayer).firstOrNull;

  ScoredVehicle? get fastestAI {
    final ais = vehicles.where((v) => !v.isPlayer && v.bestLapTime > 0).toList();
    if (ais.isEmpty) return null;
    ais.sort((a, b) => a.bestLapTime.compareTo(b.bestLapTime));
    return ais.first;
  }
}
```

```dart
// lib/features/capture/domain/session_info.dart
import 'package:flutter/foundation.dart';

@immutable
class SessionInfo {
  final String trackName;
  final String vehicleName;
  final String vehicleClass;
  final int sessionType;
  final DateTime startTime;
  final double ambientTemp;
  final double trackTemp;

  const SessionInfo({
    this.trackName = '',
    this.vehicleName = '',
    this.vehicleClass = '',
    this.sessionType = 0,
    required this.startTime,
    this.ambientTemp = 0,
    this.trackTemp = 0,
  });

  String get sessionTypeLabel => switch (sessionType) {
    0 => 'Test',
    >= 1 && <= 4 => 'Practice',
    >= 5 && <= 8 => 'Qualifying',
    9 => 'Warmup',
    >= 10 && <= 13 => 'Race',
    _ => 'Unknown',
  };
}
```

**Step 3: Run tests — PASS**

**Step 4: Commit**

```bash
git commit -m "feat(capture): add telemetry domain models — VehicleState, WheelData, frames"
```

---

### Task 2.2: Session domain models

**Files:**
- Create: `lib/features/session/domain/session.dart`
- Create: `lib/features/session/domain/lap.dart`
- Create: `lib/features/session/domain/sector.dart`
- Test: `test/features/session/domain/lap_test.dart`

Same TDD pattern. Models: Session (id, trackName, vehicleName, date, laps), Lap (number, time, s1, s2, s3, fuel, tireWear, isValid), Sector (number, time, deltaVsBest).

**Commit:** `feat(session): add Session, Lap, Sector domain models`

---

### Task 2.3: Circuit 3D domain models

**Files:**
- Create: `lib/features/circuit_3d/domain/circuit_point.dart`
- Create: `lib/features/circuit_3d/domain/trajectory_point.dart`
- Create: `lib/features/circuit_3d/domain/camera_state.dart`
- Create: `lib/features/circuit_3d/domain/circuit_model.dart`
- Test: `test/features/circuit_3d/domain/camera_state_test.dart`

Same TDD pattern. CameraState with orbit calculations (azimuth, elevation -> eye position via spherical coords).

**Commit:** `feat(circuit_3d): add CircuitPoint, TrajectoryPoint, CameraState domain models`

---

### Task 2.4: Credits domain models (clone adriencoaching)

**Files:**
- Create: `lib/features/credits/domain/credit_account.dart`
- Create: `lib/features/credits/domain/credit_pack.dart`
- Create: `lib/features/credits/domain/credit_transaction.dart`
- Test: `test/features/credits/domain/credit_account_test.dart`

Direct port from adriencoaching. Adjust pack names/prices for racing context.

**Commit:** `feat(credits): add CreditAccount, CreditPack, CreditTransaction models`

---

### Task 2.5: AI Coach domain models

**Files:**
- Create: `lib/features/ai_coach/domain/coaching_report.dart`
- Create: `lib/features/ai_coach/domain/coaching_request.dart`
- Create: `lib/features/ai_coach/domain/improvement_tip.dart`
- Create: `lib/features/setup_advisor/domain/setup_recommendation.dart`
- Create: `lib/features/setup_advisor/domain/setup_context.dart`
- Test: `test/features/ai_coach/domain/coaching_report_test.dart`

**Commit:** `feat(ai_coach): add CoachingReport, ImprovementTip, SetupRecommendation models`

---

### Task 2.6: Profile domain model

**Files:**
- Create: `lib/features/profile/domain/driver_profile.dart`
- Test: `test/features/profile/domain/driver_profile_test.dart`

**Commit:** `feat(profile): add DriverProfile model`

---

## Phase 3: Circuit 3D Renderer

### Task 3.1: Projection utilities

**Files:**
- Create: `lib/features/circuit_3d/utils/projection.dart`
- Create: `lib/features/circuit_3d/utils/color_map.dart`
- Test: `test/features/circuit_3d/utils/projection_test.dart`
- Test: `test/features/circuit_3d/utils/color_map_test.dart`

Implement `project3D` function, `makeOrbitViewMatrix`, and speed-to-color heatmap mapping. Pure math, easily testable.

**Commit:** `feat(circuit_3d): add 3D projection and heatmap color utilities`

---

### Task 3.2: Orbit camera controller

**Files:**
- Create: `lib/features/circuit_3d/controllers/orbit_camera.dart`
- Test: `test/features/circuit_3d/controllers/orbit_camera_test.dart`

StateNotifier that converts drag/scroll gestures into CameraState updates (azimuth, elevation, distance). Clamp elevation to avoid gimbal lock.

**Commit:** `feat(circuit_3d): add OrbitCameraController with gesture handling`

---

### Task 3.3: Circuit & trajectory painters

**Files:**
- Create: `lib/features/circuit_3d/presentation/painters/circuit_painter.dart`
- Create: `lib/features/circuit_3d/presentation/painters/trajectory_painter.dart`
- Test: `test/features/circuit_3d/presentation/painters/circuit_painter_test.dart`

CustomPainter implementations using projection utils. CircuitPainter draws track edges, TrajectoryPainter draws colored lines.

**Commit:** `feat(circuit_3d): add CircuitPainter and TrajectoryPainter`

---

### Task 3.4: Circuit3DView widget

**Files:**
- Create: `lib/features/circuit_3d/presentation/circuit_3d_view.dart`
- Update: `lib/routing/app_router.dart` (replace /circuit Placeholder)

Assembles GestureDetector + CustomPaint + OrbitCamera. Wire into app router.

**Commit:** `feat(circuit_3d): add interactive Circuit3DView with orbit camera`

---

## Phase 4: Shared Memory Capture (Windows FFI)

### Task 4.1: FFI bindings for rF2 shared memory

**Files:**
- Create: `lib/features/capture/services/ffi/rf2_structs.dart`
- Create: `lib/features/capture/services/ffi/shared_memory_reader.dart`
- Test: `test/features/capture/services/ffi/shared_memory_reader_test.dart`

Define ctypes-equivalent Dart structs using `dart:ffi`. Map `$rFactor2SMMP_Telemetry$` and `$rFactor2SMMP_Scoring$` memory-mapped files. Use `win32` package for `OpenFileMappingW` and `MapViewOfFile`.

**Commit:** `feat(capture): add FFI bindings for rF2 shared memory`

---

### Task 4.2: SharedMemoryService

**Files:**
- Create: `lib/features/capture/services/shared_memory_service.dart`
- Test: `test/features/capture/services/shared_memory_service_test.dart`

Service that reads frames from shared memory, converts to domain models (TelemetryFrame, ScoringFrame). Returns Result<T>.

**Commit:** `feat(capture): add SharedMemoryService with frame reading`

---

### Task 4.3: CaptureService with recording

**Files:**
- Create: `lib/features/capture/services/capture_service.dart`
- Create: `lib/features/capture/services/session_recorder.dart`
- Create: `lib/features/capture/providers/capture_providers.dart`
- Test: `test/features/capture/services/capture_service_test.dart`

Orchestrates periodic reads (50 FPS telemetry, 5 FPS scoring). SessionRecorder writes frames to SQLite. Providers expose live streams.

**Commit:** `feat(capture): add CaptureService with live streaming and session recording`

---

## Phase 5: Session Management

### Task 5.1: Session SQLite repository

**Files:**
- Create: `lib/features/session/services/session_repository.dart`
- Create: `lib/shared/services/database_manager.dart`
- Test: `test/features/session/services/session_repository_test.dart`

Implement DatabaseManager (SgDatabaseManagerPort) with tables for sessions, laps, telemetry_points, circuit_cache. SessionRepository for CRUD.

**Commit:** `feat(session): add DatabaseManager and SessionRepository`

---

### Task 5.2: Session providers

**Files:**
- Create: `lib/features/session/providers/session_providers.dart`
- Test: `test/features/session/providers/session_providers_test.dart`

Riverpod providers: sessionListProvider, sessionDetailProvider, lapDataProvider.

**Commit:** `feat(session): add Riverpod providers`

---

## Phase 6: Dashboard Screens

### Task 6.1: Live Dashboard screen

**Files:**
- Create: `lib/features/dashboard/presentation/live_dashboard_screen.dart`
- Create: `lib/features/dashboard/presentation/widgets/gauge_cluster.dart`
- Create: `lib/features/dashboard/presentation/widgets/tire_monitor.dart`
- Create: `lib/features/dashboard/presentation/widgets/input_trace.dart`
- Update: `lib/routing/app_router.dart`

Compose SgGauge (RPM, speed, fuel), SgTireWidget, SgTelemetryBar (throttle/brake), SgLapDelta, SgSectorIndicator. ConsumerWidget watching capture providers.

**Commit:** `feat(dashboard): add LiveDashboard with gauges, tires, and inputs`

---

### Task 6.2: Post-Session screen

**Files:**
- Create: `lib/features/dashboard/presentation/post_session_screen.dart`
- Create: `lib/features/dashboard/presentation/widgets/lap_table.dart`
- Create: `lib/features/dashboard/presentation/widgets/sector_comparison.dart`

Lap table with times, delta vs best, tire wear progression. Sector comparison chart.

**Commit:** `feat(dashboard): add PostSession screen with lap table and sector analysis`

---

## Phase 7: Analysis Feature

### Task 7.1: Comparison service

**Files:**
- Create: `lib/features/analysis/domain/comparison.dart`
- Create: `lib/features/analysis/domain/delta_analysis.dart`
- Create: `lib/features/analysis/services/comparison_service.dart`
- Test: `test/features/analysis/services/comparison_service_test.dart`

Point-by-point delta calculation between two trajectories. Identify braking points, apex speeds, acceleration zones.

**Commit:** `feat(analysis): add ComparisonService with trajectory delta calculation`

---

### Task 7.2: Comparison screen

**Files:**
- Create: `lib/features/analysis/presentation/comparison_screen.dart`
- Create: `lib/features/analysis/presentation/widgets/speed_trace_overlay.dart`
- Create: `lib/features/analysis/presentation/widgets/braking_points_view.dart`

Superposed speed traces (player vs reference), braking point markers on circuit 3D.

**Commit:** `feat(analysis): add ComparisonScreen with speed traces and braking points`

---

## Phase 8: Firebase & Auth

### Task 8.1: Firebase setup

**Files:**
- Modify: `lib/main.dart`
- Create: `lib/config/firebase_options.dart` (via flutterfire configure)
- Create: `lib/features/auth/providers/auth_providers.dart`

Run `flutterfire configure`, add Firebase.initializeApp, wire sg_auth.

**Commit:** `feat(auth): configure Firebase and sg_auth integration`

---

### Task 8.2: Profile service

**Files:**
- Create: `lib/features/profile/services/profile_service.dart`
- Create: `lib/features/profile/providers/profile_providers.dart`
- Create: `lib/features/profile/presentation/profile_screen.dart`

Firestore-backed profile (driver name, favorite circuits, total laps, best times).

**Commit:** `feat(profile): add ProfileService and ProfileScreen`

---

## Phase 9: Credits & Stripe

### Task 9.1: Credits service (port from adriencoaching)

**Files:**
- Create: `lib/features/credits/services/credit_service.dart`
- Create: `lib/features/credits/providers/credit_providers.dart`
- Test: `test/features/credits/services/credit_service_test.dart`

Direct port of adriencoaching CreditService. Firestore paths: `users/{uid}/credit_account/main`, `users/{uid}/credit_transactions/`.

**Commit:** `feat(credits): add CreditService and providers`

---

### Task 9.2: Credits dashboard screen

**Files:**
- Create: `lib/features/credits/presentation/credits_screen.dart`

Balance display, pack cards with prices, transaction history. Reuse SgGlassContainer, SgNeonBorder.

**Commit:** `feat(credits): add CreditsScreen with packs and transaction history`

---

### Task 9.3: Cloud Functions — credits backend

**Files:**
- Create: `backend/functions/package.json`
- Create: `backend/functions/tsconfig.json`
- Create: `backend/functions/src/index.ts`
- Create: `backend/functions/src/credits/create-checkout-session.ts`
- Create: `backend/functions/src/credits/consume-credits.ts`
- Create: `backend/functions/src/credits/stripe-webhook.ts`

Port from adriencoaching with minimal changes.

**Commit:** `feat(backend): add Cloud Functions for credits and Stripe integration`

---

## Phase 10: AI Coach & Setup Advisor

### Task 10.1: AI Coach Cloud Function

**Files:**
- Create: `backend/functions/src/ai/coach-session.ts`

Cloud Function that receives telemetry summary, calls Claude API, returns structured CoachingReport. Consumes credits on success.

**Commit:** `feat(backend): add AI coach Cloud Function with Claude integration`

---

### Task 10.2: AI Coach Flutter service

**Files:**
- Create: `lib/features/ai_coach/services/ai_coach_service.dart`
- Create: `lib/features/ai_coach/providers/ai_coach_providers.dart`
- Create: `lib/features/ai_coach/presentation/coaching_screen.dart`
- Create: `lib/features/ai_coach/presentation/widgets/tip_card.dart`

Service calls Cloud Function via sg_cloud. Screen displays coaching report with tips in SgGlassContainer cards with SgNeonBorder.

**Commit:** `feat(ai_coach): add CoachingScreen with AI-generated tips`

---

### Task 10.3: Setup Advisor

**Files:**
- Create: `backend/functions/src/ai/setup-advisor.ts`
- Create: `lib/features/setup_advisor/services/setup_service.dart`
- Create: `lib/features/setup_advisor/providers/setup_providers.dart`
- Create: `lib/features/setup_advisor/presentation/setup_screen.dart`

Same pattern as AI Coach. Sends track + conditions + driving style, gets setup recommendations.

**Commit:** `feat(setup_advisor): add SetupAdvisor with Claude-powered recommendations`

---

## Phase 11: CrewChief Integration (Extension)

### Task 11.1: IPC server for vocal tips

**Files:**
- Create: `lib/features/crewchief/services/ipc_server.dart`
- Create: `lib/features/crewchief/domain/vocal_tip.dart`
- Create: `lib/features/crewchief/providers/crewchief_providers.dart`

TCP server on localhost that sends JSON vocal tips. CrewChief plugin connects and reads tips to driver via TTS.

**Commit:** `feat(crewchief): add IPC server for real-time vocal AI tips`

---

## Phase 12: Polish & Release

### Task 12.1: Session sync to Firestore

**Files:**
- Create: `lib/features/session/services/session_sync_service.dart`

Upload best laps and session summaries to Firestore for web access.

**Commit:** `feat(session): add Firestore sync for cross-device access`

---

### Task 12.2: Circuit cache system

**Files:**
- Modify: `lib/features/circuit_3d/domain/circuit_model.dart`
- Create: `lib/features/circuit_3d/services/circuit_builder.dart`
- Create: `lib/features/circuit_3d/services/circuit_cache.dart`

Build circuit from first lap data, smooth with cubic spline, cache in SQLite.

**Commit:** `feat(circuit_3d): add circuit auto-builder and SQLite cache`

---

### Task 12.3: Feature flags & analytics

**Files:**
- Create: `lib/shared/providers/feature_flag_providers.dart`
- Create: `lib/shared/providers/analytics_providers.dart`

Wire sg_feature_flags and sg_analytics. Gate AI features behind flags.

**Commit:** `feat(shared): add feature flags and analytics integration`

---

### Task 12.4: Final polish & release build

- Run `flutter analyze` — zero issues
- Run `flutter test` — all pass
- Run `flutter build windows --release`
- Run `flutter build web --release`
- Tag `v0.1.0`

**Commit:** `release: v0.1.0 — initial release`

---

## Dependency Graph

```
Phase 0 (Bootstrap)
  └── Phase 1 (sg_ui Racing Theme)
  └── Phase 2 (Domain Models)
        ├── Phase 3 (Circuit 3D) ← depends on 2.3
        ├── Phase 4 (Capture FFI) ← depends on 2.1
        ├── Phase 5 (Session) ← depends on 2.2
        │     └── Phase 6 (Dashboard) ← depends on 3, 4, 5
        │           └── Phase 7 (Analysis) ← depends on 6
        ├── Phase 8 (Firebase/Auth) ← independent
        │     └── Phase 9 (Credits) ← depends on 2.4, 8
        │           └── Phase 10 (AI Coach) ← depends on 7, 9
        │                 └── Phase 11 (CrewChief) ← depends on 10
        └── Phase 12 (Polish) ← depends on all
```

**Parallelisable:**
- Phase 1 + Phase 2 (independent)
- Phase 3 + Phase 4 + Phase 8 (independent after Phase 2)
- Phase 5 + Phase 9 (independent tracks)
