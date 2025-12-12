---
name: native-app-builder
description: Comprehensive guide for building professional native applications with Telegram-level polish across iOS, Android, macOS, Windows, and Linux. Includes advanced animation systems, architecture patterns, performance optimization, and React Native native integration. Use when building native apps that require exceptional animation quality, complex architectures, or hybrid RN+native implementations.
version: 1.0.0
platforms: [iOS, Android, macOS, Windows, Linux, React Native]
---

# Native App Builder: Professional Cross-Platform Development

## Overview

Build production-quality native applications with Telegram-level polish across all major platforms. This skill provides battle-tested patterns for smooth 60-120fps animations, scalable architectures, performance optimization, and seamless React Native native integration.

**When to use this skill:**
- Building native iOS, Android, macOS, Windows, or Linux applications
- Implementing complex, physics-based animations
- Architecting large-scale native projects
- Creating React Native apps with native module integration
- Optimizing app performance to eliminate jank
- Designing motion that feels natural and responsive

---

# 🎯 Quick Start Decision Tree

## Platform Selection

```
Need cross-platform?
├─ YES → Consider these options:
│  ├─ Web background? → React Native + Native modules (this guide covers both)
│  ├─ Desktop focus? → Qt 6 (covers all platforms including mobile)
│  └─ Maximum performance? → Pure native with shared C++ business logic
│
└─ NO → Go pure native:
   ├─ iOS/macOS → SwiftUI + UIKit (Core Animation)
   ├─ Android → Jetpack Compose + Kotlin
   ├─ Windows → WinUI 3
   └─ Linux → GTK 4 or Qt 6
```

## Animation Approach Selection

```
What's your animation complexity?
├─ Simple transitions (opacity, scale)
│  → Use built-in declarative APIs (SwiftUI, Compose, CSS-like)
│
├─ Gesture-driven interactions
│  └─ iOS → UIViewPropertyAnimator
│  └─ Android → Animatable with gestures
│  └─ RN → Reanimated 3 + Gesture Handler
│
├─ Complex physics (springs, momentum)
│  └─ All platforms → Spring-based animation systems (detailed in Animation section)
│
└─ Custom rendering (particles, game-like)
   └─ iOS/macOS → Metal + Core Animation
   └─ Android → Custom Canvas + Compose
   └─ Cross-platform → Use Skia (RN Skia, Flutter)
```

---

# 📱 Phase 1: Project Setup and Architecture

## Step 1.1: Choose Your Architecture Pattern

### For Single-Feature Apps (<5 screens)
**Use: MV (Model-View) or MVVM Lite**

```swift
// iOS - Simple ViewModel
@Observable class ProfileViewModel {
    var user: User?
    var isLoading = false
    
    func loadProfile() async {
        isLoading = true
        defer { isLoading = false }
        user = try? await api.fetchProfile()
    }
}
```

```kotlin
// Android - Simple ViewModel
class ProfileViewModel(private val repository: UserRepository) : ViewModel() {
    private val _user = MutableStateFlow<User?>(null)
    val user = _user.asStateFlow()
    
    fun loadProfile() = viewModelScope.launch {
        _user.value = repository.getProfile()
    }
}
```

### For Medium Apps (5-20 screens)
**Use: MVVM + Repository Pattern**

```
app/
├── data/
│   ├── repository/
│   └── api/
├── domain/
│   └── models/
└── presentation/
    ├── home/
    │   ├── HomeView
    │   └── HomeViewModel
    └── profile/
```

### For Large Apps (20+ screens, multiple teams)
**Use: Clean Architecture + Modularization**

[📋 View Complete Clean Architecture Guide](./reference/clean-architecture.md)

```
:app (navigation, DI setup)
:feature-home
:feature-profile
:feature-settings
:core-network
:core-database
:core-ui (design system)
:domain (use cases, entities)
```

**Impact**: Uber reduced build times by **60%** and enabled 200+ concurrent engineers using this pattern.

## Step 1.2: Set Up Dependency Injection

### iOS: Factory or Resolver

```swift
// Factory pattern - compile-time safe
extension Container {
    var userRepository: Factory<UserRepository> {
        Factory(self) { UserRepositoryImpl() }
    }
}

// Usage in View
@Injected(\.userRepository) private var repository
```

### Android: Hilt

```kotlin
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel()

@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    @Provides
    @Singleton
    fun provideUserRepository(): UserRepository = UserRepositoryImpl()
}
```

### React Native: Context + Custom Hooks

```typescript
const RepositoryContext = createContext<UserRepository>(null!);

export function RepositoryProvider({ children }) {
  const repository = useMemo(() => new UserRepositoryImpl(), []);
  return <RepositoryContext.Provider value={repository}>{children}</RepositoryContext.Provider>;
}

export const useUserRepository = () => useContext(RepositoryContext);
```

## Step 1.3: Initial Project Structure

Run these commands to set up your project:

### iOS (SwiftUI + UIKit)
```bash
# Create Xcode project with recommended structure
xcodebuild -project MyApp.xcodeproj -scheme MyApp clean build

# Add Swift Package dependencies via Package.swift
dependencies: [
    .package(url: "https://github.com/Alamofire/Alamofire.git", from: "5.8.0"),
    .package(url: "https://github.com/hmlongco/Factory.git", from: "2.3.0")
]
```

### Android (Jetpack Compose + Kotlin)
```bash
# In build.gradle.kts (app level)
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("com.google.dagger.hilt.android")
    kotlin("kapt")
}

dependencies {
    // Compose BOM for version management
    implementation(platform("androidx.compose:compose-bom:2024.10.00"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
    
    // Hilt DI
    implementation("com.google.dagger:hilt-android:2.50")
    kapt("com.google.dagger:hilt-compiler:2.50")
}
```

### React Native with Native Modules
```bash
npx react-native init MyApp --template react-native-template-typescript

# Add animation libraries
npm install react-native-reanimated react-native-gesture-handler

# Enable New Architecture (RN 0.76+)
# Already enabled by default in new projects
```

[📱 View Complete Setup Guide for Each Platform](./reference/project-setup.md)

---

# 🎨 Phase 2: Animation System Implementation

## The Telegram Animation Formula

**Telegram's secret**: Spring physics + Interruptible gestures + Off-thread rendering

### Step 2.1: Understand Spring Physics

All modern animation systems use springs defined by three parameters:

| Parameter | Effect | Telegram Typical Values |
|-----------|--------|-------------------------|
| **Damping** | Friction/oscillation decay | 15-25 |
| **Stiffness** | Spring tensile strength | 200-400 |
| **Mass** | Object weight | 1.0 |

**Duration calculation** (critically damped spring):
```
duration ≈ 4 * sqrt(mass / stiffness) * damping
```

For a Telegram-style button press:
- Damping: 20, Stiffness: 300, Mass: 1
- Results in ~0.46s duration with smooth settle

### Step 2.2: Platform-Specific Spring Implementation

#### iOS: Core Animation Springs

```swift
// Modern SwiftUI approach (iOS 17+)
Button("Press Me") {
    withAnimation(.spring(duration: 0.5, bounce: 0.2)) {
        scale = pressed ? 1.2 : 1.0
    }
}

// UIKit approach for complex interactions
let spring = CASpringAnimation(keyPath: "transform.scale")
spring.damping = 20
spring.stiffness = 300
spring.mass = 1
spring.initialVelocity = 0
spring.duration = spring.settlingDuration
layer.add(spring, forKey: "bounce")
```

**UIViewPropertyAnimator for Interruptible Animations:**

```swift
class InteractiveViewController: UIViewController {
    private var animator: UIViewPropertyAnimator?
    
    func setupGesture() {
        let pan = UIPanGestureRecognizer(target: self, action: #selector(handlePan))
        view.addGestureRecognizer(pan)
    }
    
    @objc func handlePan(_ gesture: UIPanGestureRecognizer) {
        let translation = gesture.translation(in: view)
        let progress = translation.x / view.bounds.width
        
        switch gesture.state {
        case .began:
            animator = UIViewPropertyAnimator(duration: 0.6, dampingRatio: 0.75) {
                self.cardView.transform = CGAffineTransform(translationX: self.view.bounds.width, y: 0)
            }
            animator?.pauseAnimation()
            
        case .changed:
            animator?.fractionComplete = abs(progress)
            
        case .ended:
            let velocity = gesture.velocity(in: view).x
            if abs(progress) > 0.5 || abs(velocity) > 500 {
                // Continue animation to completion
                animator?.continueAnimation(withTimingParameters: nil, durationFactor: 0)
            } else {
                // Reverse animation
                animator?.isReversed = true
                animator?.continueAnimation(withTimingParameters: nil, durationFactor: 0)
            }
            
        default:
            break
        }
    }
}
```

**🎯 Key Insight**: UIViewPropertyAnimator allows users to grab animations mid-flight, creating the responsive feel that defines Telegram.

#### Android: Jetpack Compose Springs

```kotlin
// Compose approach - interruption handled automatically
var expanded by remember { mutableStateOf(false) }

val scale by animateFloatAsState(
    targetValue = if (expanded) 1.5f else 1f,
    animationSpec = spring(
        dampingRatio = Spring.DampingRatioMediumBouncy,
        stiffness = Spring.StiffnessLow,
        visibilityThreshold = 0.001f
    )
)

Box(
    modifier = Modifier
        .scale(scale)
        .clickable { expanded = !expanded }
)
```

**For Gesture-Driven Animations:**

```kotlin
val offsetX = remember { Animatable(0f) }

Box(
    modifier = Modifier
        .offset { IntOffset(offsetX.value.roundToInt(), 0) }
        .pointerInput(Unit) {
            detectDragGestures(
                onDrag = { change, dragAmount ->
                    change.consume()
                    launch {
                        offsetX.snapTo(offsetX.value + dragAmount.x)
                    }
                },
                onDragEnd = {
                    launch {
                        offsetX.animateTo(
                            targetValue = 0f,
                            animationSpec = spring(
                                dampingRatio = Spring.DampingRatioMediumBouncy,
                                stiffness = Spring.StiffnessMedium
                            )
                        )
                    }
                }
            )
        }
)
```

#### React Native: Reanimated 3 with Worklets

```typescript
import { useSharedValue, useAnimatedStyle, withSpring } from 'react-native-reanimated';
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

// Telegram-style spring configuration
const SPRING_CONFIG = {
  damping: 20,
  stiffness: 300,
  mass: 1,
  overshootClamping: false,
  restDisplacementThreshold: 0.01,
  restSpeedThreshold: 0.01,
};

function SwipeableCard() {
  const translateX = useSharedValue(0);
  const context = useSharedValue({ x: 0 });

  const gesture = Gesture.Pan()
    .onStart(() => {
      context.value = { x: translateX.value };
    })
    .onUpdate((e) => {
      translateX.value = e.translationX + context.value.x;
    })
    .onEnd((e) => {
      // Check if swipe threshold exceeded
      if (Math.abs(translateX.value) > SWIPE_THRESHOLD) {
        translateX.value = withSpring(
          Math.sign(translateX.value) * 500,
          SPRING_CONFIG
        );
      } else {
        translateX.value = withSpring(0, SPRING_CONFIG);
      }
    });

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }],
  }));

  return (
    <GestureDetector gesture={gesture}>
      <Animated.View style={[styles.card, animatedStyle]}>
        {/* Card content */}
      </Animated.View>
    </GestureDetector>
  );
}
```

**🔑 Critical**: The `'worklet'` directive runs this code on the UI thread, enabling 120fps.

### Step 2.3: Animation Performance Checklist

Before shipping any animation, verify:

- [ ] **Frame rate**: Consistent 60fps on low-end devices (test on 3-year-old hardware)
- [ ] **Interruptibility**: User can interrupt and reverse mid-animation
- [ ] **Velocity preservation**: Releasing during motion maintains momentum
- [ ] **No jank**: Profile with Instruments/Android Profiler, zero dropped frames
- [ ] **Battery impact**: Animations don't cause excessive CPU/GPU usage
- [ ] **Animate compositor properties only**: `transform`, `opacity` (avoid `width`, `height`, `color`)

[🎬 View Advanced Animation Patterns](./reference/animation-patterns.md)

---

# 🏗️ Phase 3: Performance Optimization

## Step 3.1: Rendering Performance

### iOS: Core Animation Optimization

**The Golden Rules:**
1. **Keep layer tree shallow** (<10 levels deep)
2. **Avoid offscreen rendering during animations**
3. **Use `shouldRasterize` for static complex views**
4. **Pre-render first frame before animation starts**

```swift
// BAD - Triggers offscreen rendering
layer.cornerRadius = 10
layer.masksToBounds = true  // ❌ Expensive!

// GOOD - Use UIBezierPath mask instead
let maskLayer = CAShapeLayer()
maskLayer.path = UIBezierPath(roundedRect: bounds, cornerRadius: 10).cgPath
layer.mask = maskLayer
```

**Telegram's Technique: AsyncDisplayKit (Texture)**

Telegram uses a forked AsyncDisplayKit that moves rendering off the main thread:

```swift
// Conceptual example - Telegram's approach
class TextNode: ASDisplayNode {
    override func layout() {
        // This runs on background thread
        let textLayout = layoutText(attributedString, constrainedSize: bounds.size)
        self.textLayout = textLayout
    }
    
    override func drawParameters(forAsyncLayer layer: _ASDisplayLayer) -> NSObjectProtocol? {
        return TextDrawParameters(layout: textLayout)
    }
    
    // Actual drawing happens off main thread
    class func draw(_ bounds: CGRect, withParameters parameters: NSObjectProtocol?, 
                    isCancelled: () -> Bool, isRasterizing: Bool) {
        // CoreText drawing here
    }
}
```

**For most apps**: Don't build your own Texture fork. Use standard UIKit/SwiftUI with proper profiling.

### Android: Jetpack Compose Optimization

```kotlin
// BAD - Recomposes entire screen
@Composable
fun HomeScreen(viewModel: HomeViewModel) {
    val state by viewModel.state.collectAsState()
    
    Column {
        Header(state.title)  // ❌ Recomposes when unrelated state changes
        Content(state.items)
    }
}

// GOOD - Isolated recomposition
@Composable
fun HomeScreen(viewModel: HomeViewModel) {
    val state by viewModel.state.collectAsState()
    
    Column {
        Header(title = state.title)  // ✅ Only recomposes when title changes
        Content(items = state.items) // ✅ Only recomposes when items change
    }
}

@Composable
fun Header(title: String) {  // Stable parameter = skippable
    Text(title)
}
```

**derivedStateOf for Expensive Computations:**

```kotlin
val listState = rememberLazyListState()

// BAD - Recalculates on every scroll pixel
val showButton = listState.firstVisibleItemIndex > 0

// GOOD - Only updates when crossing threshold
val showButton by remember {
    derivedStateOf {
        listState.firstVisibleItemIndex > 0
    }
}
```

### React Native: Reanimated Performance

```typescript
// BAD - Crosses bridge on every frame
const animatedStyle = useAnimatedStyle(() => {
  console.log(translateX.value); // ❌ Logs break worklet optimization
  return {
    transform: [{ translateX: translateX.value }]
  };
});

// GOOD - Pure worklet stays on UI thread
const animatedStyle = useAnimatedStyle(() => {
  'worklet'; // Explicit marking (optional in newer versions)
  return {
    transform: [{ translateX: translateX.value }]
  };
});
```

## Step 3.2: Memory Management

### iOS: Weak References and Capture Lists

```swift
class ProfileViewController: UIViewController {
    private var task: Task<Void, Never>?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // BAD - Retains self, causes retain cycle
        task = Task {
            await loadData()
        }
        
        // GOOD - Weak self prevents retain cycle
        task = Task { [weak self] in
            await self?.loadData()
        }
    }
    
    deinit {
        task?.cancel()
    }
}
```

### Android: Lifecycle-Aware Components

```kotlin
class ProfileFragment : Fragment() {
    private val viewModel: ProfileViewModel by viewModels()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // BAD - Collects even when fragment not visible
        lifecycleScope.launch {
            viewModel.profile.collect { profile ->
                updateUI(profile)
            }
        }
        
        // GOOD - Only collects when fragment in STARTED state
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.profile.collect { profile ->
                    updateUI(profile)
                }
            }
        }
    }
}
```

## Step 3.3: Profiling Workflow

### iOS: Xcode Instruments

```bash
# Profile your app
1. Product → Profile (⌘I)
2. Choose template:
   - Time Profiler → CPU bottlenecks
   - Allocations → Memory leaks
   - Core Animation → FPS issues
3. Record while testing animations
4. Look for:
   - Methods taking >16ms (causes dropped frames)
   - Retain cycles in Allocations
   - Red bars in Core Animation FPS graph
```

**Key Metrics:**
- **Main thread usage**: Should stay <80% during animations
- **GPU usage**: <90% sustained
- **FPS**: Consistent 60 (or 120 on ProMotion)

### Android: Android Studio Profiler

```bash
# Profile your app
1. Run → Profile 'app'
2. Select profiler:
   - CPU → Method traces
   - Memory → Allocations
   - Energy → Battery impact
3. Record animation sequence
4. Look for:
   - Frames >16ms
   - Memory leaks (objects not GC'd)
   - Excessive allocations during animations
```

**Enable GPU Rendering Profile:**
```
Settings → Developer Options → Profile GPU Rendering → On screen as bars
```
Each bar represents a frame. Green line is 16ms threshold (60fps).

[🔬 View Complete Profiling Guide](./reference/profiling.md)

---

# 🔌 Phase 4: React Native Native Integration

## When to Use Native Modules

**Use native modules when you need:**
- Complex animations requiring 120fps on iOS ProMotion displays
- Platform-specific APIs not available in RN (biometrics, health data)
- Performance-critical operations (image processing, encryption)
- Access to native UI components (custom video players, maps)

**Don't use native modules for:**
- Simple animations (Reanimated 3 handles most cases)
- Network requests (use fetch or axios)
- State management (use React state/Redux)
- Simple UI (Flexbox and RN primitives are sufficient)

## Step 4.1: Create Native UI Component (New Architecture)

### iOS: Swift UI Component

```swift
// RNSmoothAnimatedView.swift
import UIKit

@objc(RNSmoothAnimatedViewManager)
class RNSmoothAnimatedViewManager: RCTViewManager {
    override func view() -> UIView! {
        return RNSmoothAnimatedView()
    }
    
    override static func requiresMainQueueSetup() -> Bool {
        return true
    }
}

class RNSmoothAnimatedView: UIView {
    private var animator: UIViewPropertyAnimator?
    
    @objc var progress: CGFloat = 0 {
        didSet {
            updateProgress()
        }
    }
    
    private func updateProgress() {
        // Smooth animation driven by RN
        let target = CGAffineTransform(scaleX: 1 + progress * 0.5, y: 1 + progress * 0.5)
        
        animator?.stopAnimation(true)
        animator = UIViewPropertyAnimator(duration: 0.3, dampingRatio: 0.8) {
            self.transform = target
        }
        animator?.startAnimation()
    }
}

// Expose to RN
@objc(RNSmoothAnimatedViewManager)
class RNSmoothAnimatedViewManager: RCTViewManager {
    @objc func setProgress(_ node: NSNumber, progress: NSNumber) {
        DispatchQueue.main.async {
            let view = self.bridge.uiManager.view(forReactTag: node) as! RNSmoothAnimatedView
            view.progress = CGFloat(truncating: progress)
        }
    }
}
```

```objc
// RNSmoothAnimatedView.m - Bridge file
#import <React/RCTViewManager.h>

@interface RCT_EXTERN_MODULE(RNSmoothAnimatedViewManager, RCTViewManager)
RCT_EXPORT_VIEW_PROPERTY(progress, CGFloat)
@end
```

### Android: Kotlin UI Component

```kotlin
// RNSmoothAnimatedViewManager.kt
package com.myapp.animations

import android.view.View
import androidx.dynamicanimation.animation.SpringAnimation
import androidx.dynamicanimation.animation.SpringForce
import com.facebook.react.uimanager.SimpleViewManager
import com.facebook.react.uimanager.ThemedReactContext
import com.facebook.react.uimanager.annotations.ReactProp

class RNSmoothAnimatedViewManager : SimpleViewManager<RNSmoothAnimatedView>() {
    override fun getName() = "RNSmoothAnimatedView"
    
    override fun createViewInstance(reactContext: ThemedReactContext): RNSmoothAnimatedView {
        return RNSmoothAnimatedView(reactContext)
    }
    
    @ReactProp(name = "progress")
    fun setProgress(view: RNSmoothAnimatedView, progress: Float) {
        view.setProgress(progress)
    }
}

class RNSmoothAnimatedView(context: Context) : View(context) {
    private var scaleXAnimation: SpringAnimation? = null
    private var scaleYAnimation: SpringAnimation? = null
    
    fun setProgress(progress: Float) {
        val targetScale = 1f + progress * 0.5f
        
        scaleXAnimation?.cancel()
        scaleYAnimation?.cancel()
        
        scaleXAnimation = SpringAnimation(this, SpringAnimation.SCALE_X).apply {
            spring = SpringForce(targetScale).apply {
                dampingRatio = SpringForce.DAMPING_RATIO_MEDIUM_BOUNCY
                stiffness = SpringForce.STIFFNESS_LOW
            }
            start()
        }
        
        scaleYAnimation = SpringAnimation(this, SpringAnimation.SCALE_Y).apply {
            spring = SpringForce(targetScale).apply {
                dampingRatio = SpringForce.DAMPING_RATIO_MEDIUM_BOUNCY
                stiffness = SpringForce.STIFFNESS_LOW
            }
            start()
        }
    }
}

// Register in package
class MyAppPackage : ReactPackage {
    override fun createViewManagers(reactContext: ReactApplicationContext) =
        listOf(RNSmoothAnimatedViewManager())
}
```

### React Native: TypeScript Interface

```typescript
// SmoothAnimatedView.tsx
import { requireNativeComponent, ViewProps } from 'react-native';
import { useSharedValue, useAnimatedReaction, runOnUI } from 'react-native-reanimated';

interface SmoothAnimatedViewProps extends ViewProps {
  progress: number;
}

const NativeSmoothAnimatedView = requireNativeComponent<SmoothAnimatedViewProps>(
  'RNSmoothAnimatedView'
);

export function SmoothAnimatedView({ progress, ...props }: SmoothAnimatedViewProps) {
  const animatedProgress = useSharedValue(0);
  
  useEffect(() => {
    animatedProgress.value = withSpring(progress);
  }, [progress]);
  
  // Bridge to native only when value changes
  useAnimatedReaction(
    () => animatedProgress.value,
    (value) => {
      // This runs on UI thread, minimal overhead
      runOnUI(() => {
        // Call native method (implementation depends on TurboModules setup)
      })();
    }
  );
  
  return <NativeSmoothAnimatedView progress={animatedProgress.value} {...props} />;
}
```

## Step 4.2: Native Module for Business Logic

### iOS: Swift Module

```swift
// RNNativeAnimations.swift
@objc(RNNativeAnimations)
class RNNativeAnimations: NSObject {
    
    @objc
    func calculateSpringPath(
        _ startValue: Double,
        endValue: Double,
        velocity: Double,
        damping: Double,
        resolver: @escaping RCTPromiseResolveBlock,
        rejecter: @escaping RCTPromiseRejectBlock
    ) {
        // Complex spring calculation that benefits from native performance
        let path = performComplexSpringCalculation(
            start: startValue,
            end: endValue,
            velocity: velocity,
            damping: damping
        )
        
        resolver(path)
    }
    
    @objc
    static func requiresMainQueueSetup() -> Bool {
        return false  // Can initialize on background thread
    }
}

// Bridge
@objc(RNNativeAnimations)
class RNNativeAnimationsBridge: RCTEventEmitter {
    @objc
    func calculateSpringPath(
        _ startValue: Double,
        endValue: Double,
        velocity: Double,
        damping: Double,
        resolver: @escaping RCTPromiseResolveBlock,
        rejecter: @escaping RCTPromiseRejectBlock
    ) -> Void
}
```

### Android: Kotlin Module

```kotlin
// RNNativeAnimationsModule.kt
class RNNativeAnimationsModule(reactContext: ReactApplicationContext) : 
    ReactContextBaseJavaModule(reactContext) {
    
    override fun getName() = "RNNativeAnimations"
    
    @ReactMethod
    fun calculateSpringPath(
        startValue: Double,
        endValue: Double,
        velocity: Double,
        damping: Double,
        promise: Promise
    ) {
        try {
            val path = performComplexSpringCalculation(
                startValue, endValue, velocity, damping
            )
            promise.resolve(path)
        } catch (e: Exception) {
            promise.reject("CALCULATION_ERROR", e)
        }
    }
}
```

### TypeScript: Using the Native Module

```typescript
// NativeAnimations.ts
import { NativeModules } from 'react-native';

interface NativeAnimationsModule {
  calculateSpringPath(
    startValue: number,
    endValue: number,
    velocity: number,
    damping: number
  ): Promise<number[]>;
}

const { RNNativeAnimations } = NativeModules;

export const NativeAnimations = RNNativeAnimations as NativeAnimationsModule;

// Usage
const path = await NativeAnimations.calculateSpringPath(0, 100, 50, 20);
```

[🔗 View Complete Native Integration Guide](./reference/rn-native-integration.md)

---

# 🎨 Phase 5: Animation Design Principles

## The Physics of Natural Motion

### Duration Guidelines by Animation Type

| Animation Type | Duration | Easing | Use Case |
|----------------|----------|--------|----------|
| Micro-interactions | 150-200ms | Ease-out | Button press, checkbox |
| Simple transitions | 200-300ms | Spring (low bounce) | Navigation, modal |
| Complex transitions | 400-500ms | Spring (medium bounce) | Screen swipes, reveals |
| Emphasis animations | 300-400ms | Spring (high bounce) | Highlighting, notifications |

### Spring Parameter Presets

```typescript
// Telegram-style presets
const ANIMATION_PRESETS = {
  // Quick, snappy response
  BUTTON_PRESS: {
    damping: 25,
    stiffness: 400,
    mass: 1
  },
  
  // Smooth navigation
  SCREEN_TRANSITION: {
    damping: 20,
    stiffness: 300,
    mass: 1
  },
  
  // Playful interaction
  BOUNCE: {
    damping: 12,
    stiffness: 300,
    mass: 1
  },
  
  // Gentle settle
  MODAL_PRESENT: {
    damping: 30,
    stiffness: 300,
    mass: 1
  }
};
```

### Choreography: Sequencing Multiple Animations

**The Telegram Pattern**: Stagger by 50-100ms for cascade effect

```swift
// iOS - Staggered list appearance
func animateListItems() {
    for (index, cell) in cells.enumerated() {
        UIView.animate(
            withDuration: 0.5,
            delay: Double(index) * 0.05,  // 50ms stagger
            usingSpringWithDamping: 0.8,
            initialSpringVelocity: 0,
            options: []
        ) {
            cell.alpha = 1
            cell.transform = .identity
        }
    }
}
```

```kotlin
// Android - Staggered Compose
items.forEachIndexed { index, item ->
    val alpha by animateFloatAsState(
        targetValue = 1f,
        animationSpec = spring(
            dampingRatio = Spring.DampingRatioMediumBouncy,
            stiffness = Spring.StiffnessLow,
            delayMillis = index * 50
        )
    )
    
    ListItem(item, modifier = Modifier.alpha(alpha))
}
```

## Disney's 12 Principles Applied to UI

1. **Squash and Stretch**: Button press scales down then bounces back
2. **Anticipation**: Pull-back before swipe gesture
3. **Staging**: Use motion to direct attention
4. **Follow Through**: Momentum continues after main action
5. **Ease In/Out**: Never use linear timing
6. **Arcs**: Curved paths feel more natural than straight lines
7. **Secondary Action**: Badge bounce when notification arrives
8. **Timing**: Vary speed for weight illusion
9. **Exaggeration**: Slightly overdo for playfulness
10. **Solid Drawing**: Maintain visual hierarchy during motion
11. **Appeal**: Animations should be pleasant to watch repeatedly
12. **Overlap**: Start next animation before previous completes

[🎬 View Animation Design Guide](./reference/animation-design.md)

---

# 📊 Phase 6: Testing and Quality Assurance

## Performance Benchmarks

### Required Targets

| Metric | Target | Platform | How to Measure |
|--------|--------|----------|----------------|
| Frame rate | 60 FPS | All | Profiler FPS graph |
| Frame render time | <16ms | 60Hz displays | Frame timing |
| Frame render time | <8ms | 120Hz displays | Frame timing |
| Cold start | <2s | Mobile | Launch to interactive |
| Memory usage | <200MB | Mobile (avg) | Memory profiler |
| Battery impact | <10%/hr | Mobile | Energy profiler |

### Testing Checklist

```markdown
## Animation Quality
- [ ] 60fps on 3-year-old devices
- [ ] 120fps on ProMotion/high-refresh displays
- [ ] Smooth during heavy computation
- [ ] Interruptible mid-animation
- [ ] Velocity preserved when interrupted
- [ ] No jank during transitions
- [ ] Battery impact <10% per hour of active use

## Architecture Quality
- [ ] Unit test coverage >70%
- [ ] Integration tests for critical flows
- [ ] UI tests for core user journeys
- [ ] No retain cycles (iOS)
- [ ] No memory leaks (all platforms)
- [ ] Proper lifecycle handling
- [ ] Clean architecture boundaries respected

## Code Quality
- [ ] No force unwraps (iOS)
- [ ] No non-null assertions (!!) (Kotlin)
- [ ] Proper error handling
- [ ] Consistent naming conventions
- [ ] DRY principle followed
- [ ] Comments explain "why" not "what"
- [ ] Type-safe where possible
```

## Automated Testing

### iOS: XCTest

```swift
class AnimationTests: XCTestCase {
    func testButtonAnimation() {
        let expectation = XCTestExpectation(description: "Animation completes")
        let button = AnimatedButton()
        
        button.animate {
            expectation.fulfill()
        }
        
        wait(for: [expectation], timeout: 1.0)
        XCTAssertEqual(button.transform.a, 1.2, accuracy: 0.01)
    }
}
```

### Android: Espresso + Compose UI Tests

```kotlin
@Test
fun testSwipeAnimation() {
    composeTestRule.setContent {
        SwipeableCard()
    }
    
    composeTestRule
        .onNodeWithTag("card")
        .performTouchInput {
            swipeRight()
        }
    
    composeTestRule
        .onNodeWithTag("card")
        .assertDoesNotExist()  // Should be dismissed
}
```

### React Native: Jest + React Native Testing Library

```typescript
import { render, fireEvent, waitFor } from '@testing-library/react-native';

test('animates on press', async () => {
  const { getByTestId } = render(<AnimatedButton />);
  const button = getByTestId('button');
  
  fireEvent.press(button);
  
  await waitFor(() => {
    expect(button.props.style.transform).toBeDefined();
  });
});
```

[🧪 View Complete Testing Guide](./reference/testing.md)

---

# 🚀 Phase 7: Optimization and Launch

## Pre-Launch Optimization Checklist

### iOS App Store Optimization

```bash
# 1. Enable bitcode for App Store optimization (if still supported)
# 2. Strip debug symbols from release builds
# 3. Enable aggressive optimization

# Build.xcconfig
ENABLE_BITCODE = YES
DEAD_CODE_STRIPPING = YES
SWIFT_OPTIMIZATION_LEVEL = -O
SWIFT_COMPILATION_MODE = wholemodule

# 4. App thinning automatically handled by App Store
# 5. Baseline compile for common device types
```

### Android App Bundle Optimization

```kotlin
// build.gradle.kts
android {
    bundle {
        language {
            enableSplit = true  // Split by language
        }
        density {
            enableSplit = true  // Split by screen density
        }
        abi {
            enableSplit = true  // Split by CPU architecture
        }
    }
    
    // Enable R8 full mode
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

### Baseline Profiles (Android)

**Impact**: 20-51% startup improvement

```kotlin
// In androidTest/
@RunWith(AndroidJUnit4::class)
class BaselineProfileGenerator {
    @get:Rule
    val baselineProfileRule = BaselineProfileRule()
    
    @Test
    fun startup() = baselineProfileRule.collect(
        packageName = "com.myapp",
        includeInStartupProfile = true
    ) {
        pressHome()
        startActivityAndWait()
        
        // Navigate through critical paths
        device.waitForIdle()
        device.findObject(By.text("Profile")).click()
        device.waitForIdle()
    }
}
```

## Launch Metrics to Monitor

### Week 1 Focus
- **Crash rate**: Target <0.5%
- **ANR rate** (Android): Target <0.1%
- **Slow renders**: <1% of frames
- **Cold start P95**: <2 seconds
- **Battery drain**: Monitor complaints

### Month 1 Focus
- **Session length**: Are animations encouraging engagement?
- **User retention**: Day 1, Day 7, Day 30
- **Feature adoption**: Are users discovering animated features?
- **Performance regressions**: Monitor with each update

[📈 View Analytics Setup Guide](./reference/analytics.md)

---

# 📚 Reference Documentation

## Platform-Specific Deep Dives

- [📱 iOS Development Guide](./reference/ios-guide.md) - SwiftUI, UIKit, Core Animation mastery
- [🤖 Android Development Guide](./reference/android-guide.md) - Jetpack Compose, Kotlin best practices
- [💻 macOS Development Guide](./reference/macos-guide.md) - AppKit and Mac Catalyst
- [🪟 Windows Development Guide](./reference/windows-guide.md) - WinUI 3 and Composition API
- [🐧 Linux Development Guide](./reference/linux-guide.md) - GTK 4 and Qt 6

## Architecture and Patterns

- [🏛️ Clean Architecture Guide](./reference/clean-architecture.md) - Scalable app structure
- [🎯 MVVM Implementation](./reference/mvvm-guide.md) - ViewModels done right
- [💉 Dependency Injection](./reference/dependency-injection.md) - Hilt, Factory, Resolver
- [📦 Modularization Strategy](./reference/modularization.md) - Feature modules

## Animation Deep Dives

- [🎨 Animation Design Principles](./reference/animation-design.md) - Motion design theory
- [🎬 Animation Patterns Library](./reference/animation-patterns.md) - 50+ ready-to-use animations
- [⚡ Performance Optimization](./reference/performance.md) - 60-120fps techniques
- [🎭 Lottie Integration](./reference/lottie-guide.md) - After Effects to code

## React Native Integration

- [🔗 Native Modules Guide](./reference/rn-native-integration.md) - TurboModules and Fabric
- [📲 Brownfield Integration](./reference/brownfield.md) - Adding RN to existing apps
- [🎪 Reanimated Advanced](./reference/reanimated-advanced.md) - Worklets and shared values

## Testing and Quality

- [🧪 Testing Strategy](./reference/testing.md) - Unit, integration, E2E
- [🔬 Profiling Guide](./reference/profiling.md) - Instruments, Android Profiler, Perfetto
- [📊 Analytics Setup](./reference/analytics.md) - Firebase, Mixpanel, custom events

## Tools and Build Systems

- [⚙️ Project Setup](./reference/project-setup.md) - Initial configuration for all platforms
- [🔨 Build Optimization](./reference/build-optimization.md) - Faster compile times
- [📦 Distribution](./reference/distribution.md) - App Store, Play Store, Flatpak

---

# 💡 Best Practices Summary

## Golden Rules for Native Apps

1. **Animate only compositor properties** (transform, opacity)
2. **Use springs for all interactive animations**
3. **Profile on low-end devices** (3-year-old hardware)
4. **Keep main thread free** (60fps = <16ms per frame)
5. **Make gestures interruptible** (users expect instant response)
6. **Test memory leaks** (weak references, lifecycle awareness)
7. **Measure first, optimize second** (don't guess bottlenecks)
8. **Use platform conventions** (iOS != Android != Windows)
9. **Progressive enhancement** (graceful degradation on old devices)
10. **Iterate on animation timing** (small changes matter)

## Common Pitfalls to Avoid

❌ **Don't**: Animate `frame`, `bounds`, `backgroundColor` directly
✅ **Do**: Animate `transform`, `opacity`, use `CALayer` properties

❌ **Don't**: Block main thread with heavy computation during animations
✅ **Do**: Move work to background threads, use async/await

❌ **Don't**: Use linear timing curves
✅ **Do**: Use ease-in-out or springs

❌ **Don't**: Create animations that can't be interrupted
✅ **Do**: Use UIViewPropertyAnimator, Animatable, Reanimated

❌ **Don't**: Guess what's slow
✅ **Do**: Profile with Instruments/Android Profiler

❌ **Don't**: Test only on flagship devices
✅ **Do**: Test on 3-year-old mid-range phones

❌ **Don't**: Copy-paste animation code without understanding
✅ **Do**: Understand spring physics fundamentals

## When to Reach for This Skill

Use this skill when:
- Starting a new native app project
- Implementing complex animations or interactions
- Performance optimization is critical
- Building React Native apps with native requirements
- Architecting for scale (large team, many features)
- Need to match Telegram/Instagram/Spotify polish level

Don't overthink it if:
- Building a simple CRUD app with standard UI
- Animation quality isn't a differentiator
- Timeline is extremely tight (ship first, polish later)
- Team lacks native experience (consider React Native or Flutter)

---

# 🎓 Learning Path for Mastery

## Week 1-2: Foundations
- Understand spring physics (damping, stiffness, mass)
- Complete official platform tutorials (Apple, Android)
- Build 5 simple animations from Animation Patterns Library

## Week 3-4: Architecture
- Implement MVVM in sample app
- Add dependency injection
- Write unit tests for ViewModels

## Week 5-6: Performance
- Profile sample app with Instruments/Android Profiler
- Optimize to 60fps on low-end device
- Implement memory leak detection

## Week 7-8: Advanced Animations
- Build gesture-driven interactions
- Create custom spring-based transitions
- Implement Telegram-style swipe animations

## Week 9-10: Production Ready
- Add analytics and crash reporting
- Set up CI/CD pipeline
- Create automated test suite

## Week 11-12: Polish
- Iterate on animation timing based on user feedback
- Add micro-interactions throughout app
- Optimize for 120fps on ProMotion displays

**Estimated time to Telegram-level proficiency**: 3-6 months of focused practice

---

# 🤝 Contributing and Updates

This skill is maintained based on:
- Official platform documentation (Apple, Google, Microsoft)
- Open-source projects (Telegram, Signal, Element)
- Conference talks (WWDC, Google I/O, FOSDEM)
- Production experience from companies building high-polish apps

**Version**: 1.0.0  
**Last Updated**: December 2024  
**Platforms Covered**: iOS 17+, Android 13+, macOS 14+, Windows 11, Linux (GTK 4, Qt 6), React Native 0.76+

For questions about specific implementation details, use the reference documentation or ask Claude to explain concepts in more depth.

---

Built with 💙 for developers who care about craft.
