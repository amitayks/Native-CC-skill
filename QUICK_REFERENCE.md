# Native App Builder - Quick Reference Card

**Fast lookup for critical information**

## 🎯 Platform Selection

```
Single Platform?
├─ iOS only → SwiftUI + UIKit
├─ Android only → Jetpack Compose + Kotlin
├─ macOS only → SwiftUI + AppKit
├─ Windows only → WinUI 3
└─ Linux only → GTK 4 or Qt 6

Cross-Platform?
├─ Web developer → React Native + Native modules
├─ Desktop focus → Qt 6 (covers all platforms)
└─ Maximum control → Native with shared C++ core
```

## ⚡ Performance Targets

| Metric | Target | Maximum | Critical |
|--------|--------|---------|----------|
| Frame Rate | 60 FPS | 55 FPS | 60 FPS |
| Frame Time (60Hz) | <16ms | <18ms | <16ms |
| Frame Time (120Hz) | <8ms | <10ms | <8ms |
| Cold Start | <1.5s | <2.5s | <2s |
| Memory (idle) | <100MB | <150MB | <200MB |
| Battery (1hr) | <8% | <12% | <10% |

## 🎨 Spring Animation Presets

```swift
// Quick & snappy (buttons)
damping: 25, stiffness: 400, mass: 1

// Smooth navigation
damping: 20, stiffness: 300, mass: 1

// Playful bounce
damping: 12, stiffness: 300, mass: 1

// Gentle settle (modals)
damping: 30, stiffness: 300, mass: 1
```

## 📱 Animation Properties

### ✅ ANIMATE (compositor properties)
- `transform` (translate, scale, rotate)
- `opacity`
- `backgroundColor` (with layer)

### ❌ DON'T ANIMATE (layout properties)
- `frame` / `bounds`
- `constraints`
- `backgroundColor` (on view directly)
- `cornerRadius` (without optimization)

## 🔧 Profiling Commands

### iOS
```bash
# Launch Instruments
instruments -t "Time Profiler" MyApp.app

# Key templates
- Time Profiler → CPU bottlenecks
- Allocations → Memory leaks
- Core Animation → Rendering issues
- Leaks → Automatic leak detection
```

### Android
```bash
# Enable GPU profiling
adb shell setprop debug.hwui.profile visual_bars

# Capture Perfetto trace
adb shell perfetto -o /data/misc/trace -t 10s

# Pull trace
adb pull /data/misc/trace .
```

## 🏗️ Architecture Decision Tree

```
App Size?
├─ <5 screens → MV or MVVM Lite
├─ 5-20 screens → MVVM + Repository
└─ 20+ screens → Clean Architecture + Modules

Team Size?
├─ Solo → Keep it simple, focus on features
├─ 2-5 devs → MVVM with clear boundaries
└─ 5+ devs → Full modularization
```

## 💉 Dependency Injection

### iOS
```swift
// Factory (recommended)
extension Container {
    var userRepo: Factory<UserRepository> {
        Factory(self) { UserRepositoryImpl() }
    }
}

@Injected(\.userRepo) var repository
```

### Android
```kotlin
// Hilt (recommended)
@HiltViewModel
class MyViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel()
```

## 🎬 Common Animation Patterns

### 1. Button Press
```
Scale: 1.0 → 0.95 → 1.0
Duration: 150-200ms
Spring: medium bounce
```

### 2. List Item Appear
```
Opacity: 0 → 1
TranslateY: 50 → 0
Stagger: 50ms between items
Spring: low bounce
```

### 3. Modal Present
```
TranslateY: screen height → 0
Opacity: 0 → 1 (background)
Duration: 300-400ms
Spring: gentle settle
```

### 4. Swipe Gesture
```
Follow finger position
Spring back if <threshold
Animate out if >threshold
Preserve velocity on release
```

## 🐛 Common Performance Issues

| Issue | Symptom | Fix |
|-------|---------|-----|
| Main thread block | UI freezes | Move work to background |
| Overdraw | Battery drain | Remove redundant backgrounds |
| Memory leak | Increasing RAM | Use weak references |
| Dropped frames | Stuttering | Profile & optimize render |
| Slow startup | Long launch time | Lazy init, baseline profiles |

## 📊 Memory Leak Patterns

### iOS
```swift
// ❌ Leak
closure = { self.update() }

// ✅ Fixed  
closure = { [weak self] in self?.update() }

// ❌ Leak
timer = Timer.scheduledTimer { self.tick() }

// ✅ Fixed
timer = Timer.scheduledTimer { [weak self] _ in 
    self?.tick() 
}
```

### Android
```kotlin
// ❌ Leak
button.setOnClickListener { startAnimation() }

// ✅ Fixed
lifecycleScope.launch {
    lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
        // Work here
    }
}
```

## 🔌 React Native Native Integration

### When to Go Native
- ✅ 120fps animations needed
- ✅ Platform-specific APIs
- ✅ Performance-critical operations
- ❌ Simple UI or logic
- ❌ Standard animations

### Quick Native Module

```typescript
// iOS
@objc(MyModule)
class MyModule: NSObject {
    @objc
    func doWork() { }
}

// Android
class MyModule(context: ReactApplicationContext) : 
    ReactContextBaseJavaModule(context) {
    
    @ReactMethod
    fun doWork() { }
}

// TypeScript
const { MyModule } = NativeModules;
MyModule.doWork();
```

## 🎯 Testing Checklist

- [ ] 60fps on 3-year-old device
- [ ] No memory leaks (profiler confirms)
- [ ] Animations interruptible
- [ ] Velocity preserved
- [ ] Battery impact acceptable
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Accessibility works

## 🚀 Pre-Launch Optimization

### iOS
```bash
# Enable optimizations
SWIFT_OPTIMIZATION_LEVEL = -O
DEAD_CODE_STRIPPING = YES
```

### Android
```kotlin
// Enable R8 full mode
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
        }
    }
}

// Add Baseline Profiles
```

## 📚 Quick Links

| Need | Go To |
|------|-------|
| Start project | SKILL.md Phase 1 |
| Animation code | animation-patterns.md |
| Fix performance | profiling.md |
| Native module | rn-native-integration.md |
| Architecture help | SKILL.md Phase 1.1 |

## 🔑 Golden Rules

1. **Profile first, optimize second**
2. **Animate only compositor properties**
3. **Use springs for all interactions**
4. **Test on low-end devices**
5. **Keep main thread free**
6. **Make gestures interruptible**
7. **Use weak references for closures**
8. **Measure everything**

## 📞 Emergency Fixes

### App is slow
1. Profile with Instruments/Android Profiler
2. Check main thread CPU usage
3. Look for >16ms methods
4. Move heavy work to background

### Memory growing
1. Profile with Allocations/Memory Profiler
2. Check for retain cycles
3. Use weak references in closures
4. Cancel timers/animations on cleanup

### Animations janky
1. Check frame rate in profiler
2. Ensure animating compositor properties
3. Reduce view hierarchy depth
4. Disable expensive effects during animation

---

**Print this card and keep it handy!**

For full details, see [SKILL.md](./SKILL.md)
