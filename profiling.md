# Performance Profiling and Optimization Guide

**Complete guide to identifying and fixing performance issues in native apps**

## Table of Contents

1. [Profiling Tools Overview](#profiling-tools-overview)
2. [iOS Profiling with Instruments](#ios-profiling-with-instruments)
3. [Android Profiling](#android-profiling)
4. [Performance Metrics](#performance-metrics)
5. [Common Performance Issues](#common-performance-issues)
6. [Optimization Strategies](#optimization-strategies)

---

## Profiling Tools Overview

### Platform-Specific Tools

| Platform | Primary Tool | Purpose |
|----------|--------------|---------|
| iOS | Xcode Instruments | CPU, Memory, GPU, Energy |
| Android | Android Studio Profiler | CPU, Memory, Network, Energy |
| Android | Perfetto | System-wide traces |
| All | Custom FPS counter | Real-time frame rate |

### Frame Budget Reference

| Refresh Rate | Frame Time Budget | Use Case |
|--------------|-------------------|----------|
| 60 Hz | 16.67ms | Standard displays |
| 90 Hz | 11.11ms | High-refresh Android |
| 120 Hz | 8.33ms | ProMotion iOS, flagship Android |

**Critical Rule**: If any single frame takes longer than the budget, you'll see a dropped frame (jank).

---

## iOS Profiling with Instruments

### Step 1: Launch Instruments

```bash
# From command line
instruments -t "Time Profiler" -D trace.trace MyApp.app

# Or from Xcode: Product → Profile (⌘I)
```

### Step 2: Choose the Right Template

#### Time Profiler
**Use for**: Finding CPU bottlenecks

**How to read**:
- Heaviest Stack Trace shows methods consuming most CPU time
- Call Tree shows hierarchical view
- Look for methods >16ms in main thread

**Example problematic trace**:
```
Main Thread (92% CPU)
├─ UIView.layoutSubviews: 18ms ❌ Too slow!
│  └─ CustomView.calculateLayout: 15ms
│     └─ Heavy computation in render loop
```

**Fix**:
```swift
// BAD - Computation in render
override func layoutSubviews() {
    super.layoutSubviews()
    let result = expensiveCalculation() // ❌ 15ms
    label.text = result
}

// GOOD - Cache result
private var cachedResult: String?
override func layoutSubviews() {
    super.layoutSubviews()
    label.text = cachedResult ?? ""
}

func updateContent() {
    DispatchQueue.global().async {
        let result = self.expensiveCalculation()
        DispatchQueue.main.async {
            self.cachedResult = result
            self.setNeedsLayout()
        }
    }
}
```

#### Allocations
**Use for**: Finding memory leaks

**Steps**:
1. Record baseline memory usage
2. Perform action (open/close screen)
3. Return to initial state
4. Check if memory returned to baseline

**Red flags**:
- Memory keeps growing with each iteration
- Objects not deallocated after use
- Retain cycles keeping objects alive

**Example leak**:
```swift
// BAD - Retain cycle
class ViewController: UIViewController {
    var completion: (() -> Void)?
    
    func loadData() {
        dataService.fetch { [self] data in
            self.updateUI(data) // ❌ Strong reference to self
            self.completion?()
        }
    }
}

// GOOD - Weak reference
func loadData() {
    dataService.fetch { [weak self] data in
        guard let self else { return }
        self.updateUI(data)
        self.completion?()
    }
}
```

#### Core Animation
**Use for**: Finding rendering bottlenecks

**Key metrics**:
- FPS graph (should be flat at 60)
- Color-coded rendering issues:
  - Red: Offscreen rendering
  - Yellow: Rasterized layers
  - Green: GPU-accelerated

**Common issues**:

1. **Offscreen rendering** (RED):
```swift
// BAD - Forces offscreen rendering
layer.cornerRadius = 10
layer.masksToBounds = true
layer.shadowOpacity = 0.5 // ❌ Shadow + mask = expensive!

// GOOD - Use shadow path
layer.cornerRadius = 10
layer.shadowOpacity = 0.5
layer.shadowPath = UIBezierPath(roundedRect: bounds, cornerRadius: 10).cgPath
```

2. **Blending** (expensive pixel compositing):
```swift
// BAD - Transparent background
view.backgroundColor = .clear // ❌ Requires blending

// GOOD - Opaque background
view.backgroundColor = .white
view.isOpaque = true
```

3. **View hierarchy too deep**:
```
UIViewController
└─ UIView (10 nested levels) ❌ Expensive!
   └─ UIView
      └─ UIView
         └─ ...
```

**Fix**: Flatten hierarchy, use `UIStackView` or SwiftUI

#### Leaks
**Use for**: Detecting memory leaks automatically

**How it works**: Instruments analyzes retain counts and flags objects that should be deallocated but aren't.

**Common leak patterns**:

1. **Closure capture**:
```swift
// Leak
timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { _ in
    self.update() // ❌ Captures self strongly
}

// Fixed
timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] _ in
    self?.update()
}
```

2. **Delegate retain cycle**:
```swift
// Protocol should be class-bound and weak
protocol MyDelegate: AnyObject {
    func didUpdate()
}

class MyClass {
    weak var delegate: MyDelegate? // ✅ Weak reference
}
```

### Step 3: Interpret Results

#### Time Profiler Analysis

**Look for**:
- Main thread CPU usage >80% sustained
- Methods taking >5ms in render loop
- Synchronous operations on main thread

**Example report**:
```
Main Thread: 1,234 samples (75% of total)
├─ URLSession.dataTask: 456 samples (37%) ❌ Network on main thread!
├─ JSONDecoder.decode: 234 samples (19%) ❌ Heavy parsing
└─ UIView.layoutSubviews: 123 samples (10%)

Background Thread: 321 samples (25% of total)
└─ Image processing: 321 samples ✅ Good!
```

**Fix strategy**:
1. Move URLSession to background queue
2. Use background queue for JSON parsing
3. Cache decoded results

---

## Android Profiling

### Android Studio Profiler

#### CPU Profiler

**Steps**:
1. Run app with profiling enabled
2. Click "Record" in CPU Profiler
3. Perform actions you want to profile
4. Click "Stop" and analyze

**Viewing options**:
- **Flame Chart**: Visualize call stacks over time
- **Top Down**: Shows methods sorted by CPU usage
- **Bottom Up**: Shows callees from method perspective
- **Call Chart**: Timeline view of method execution

**Example flame chart analysis**:
```
MainActivity.onCreate (800ms)
├─ RecyclerView.setAdapter (600ms) ❌ Slow!
│  └─ Adapter.onCreateViewHolder (580ms)
│     └─ LayoutInflater.inflate (570ms) ❌ Complex layouts
└─ Database query (150ms)
```

**Fix**:
```kotlin
// BAD - Complex layout inflated on main thread
override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
    val view = LayoutInflater.from(parent.context)
        .inflate(R.layout.complex_item, parent, false) // ❌ 570ms!
    return ViewHolder(view)
}

// GOOD - Simplified layout + ViewHolder pattern
override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
    // Use ConstraintLayout to flatten hierarchy
    // Use view binding for faster inflation
    return ItemViewHolder(ItemBinding.inflate(
        LayoutInflater.from(parent.context), parent, false
    ))
}
```

#### Memory Profiler

**Features**:
- Real-time memory graph
- Heap dump analysis
- Allocation tracking
- Garbage collection events

**Detecting leaks**:

1. **Baseline capture**:
   - Open screen → Capture heap dump
2. **Action**:
   - Navigate away → Force GC
3. **Verification**:
   - Capture heap dump → Compare

**Example leak**:
```
Instance View:
- MainActivity (1 instance) ❌ Should be 0 after back press
  - Retained by: lambda$onCreate$0
    - Retained by: AnimatorSet
      - Retained by: Global AnimatorPool
```

**Fix**:
```kotlin
// BAD - Animation holds reference to Activity
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        button.setOnClickListener {
            animateButton() // ❌ Implicit reference to Activity
        }
    }
    
    private fun animateButton() {
        ObjectAnimator.ofFloat(button, "alpha", 0f, 1f).apply {
            duration = 1000
            start() // ❌ Animator may outlive Activity
        }
    }
}

// GOOD - Cancel animations on destroy
class MainActivity : AppCompatActivity() {
    private val animators = mutableListOf<Animator>()
    
    private fun animateButton() {
        ObjectAnimator.ofFloat(button, "alpha", 0f, 1f).apply {
            duration = 1000
            animators.add(this)
            start()
        }
    }
    
    override fun onDestroy() {
        super.onDestroy()
        animators.forEach { it.cancel() }
        animators.clear()
    }
}
```

#### GPU Rendering Profiler

**Enable**: Developer Options → Profile GPU Rendering → "On screen as bars"

**Reading the graph**:
- Each vertical bar = one frame
- Green horizontal line = 16ms (60fps target)
- Colors show different phases:
  - Blue: Drawing commands
  - Orange: CPU work
  - Red: GPU work

**If bars exceed green line**:
1. **Simplify layouts** (reduce hierarchy depth)
2. **Reduce overdraw** (avoid drawing same pixels multiple times)
3. **Optimize custom drawing** (Canvas operations)

**Overdraw detection**:
```
Settings → Developer Options → Debug GPU Overdraw → Show overdraw areas
```

Colors mean:
- No color: 1x draw (ideal)
- Blue: 2x draw (acceptable)
- Green: 3x draw (concerning)
- Pink: 4x draw (problem)
- Red: 5x+ draw (critical issue)

**Fix overdraw**:
```kotlin
// BAD - Multiple background layers
<LinearLayout android:background="@color/white"> ❌
    <CardView android:background="@color/white"> ❌
        <TextView android:background="@color/white" /> ❌
    </CardView>
</LinearLayout>

// GOOD - Remove redundant backgrounds
<LinearLayout>
    <CardView>
        <TextView android:background="@color/white" />
    </CardView>
</LinearLayout>
```

### Perfetto (System-Wide Tracing)

**Use for**: Understanding system-level performance issues

**Capture trace**:
```bash
# Record 10-second trace
adb shell perfetto \
  -c - --txt \
  -o /data/misc/perfetto-traces/trace \
  <<EOF
buffers: {
    size_kb: 63488
}
data_sources: {
    config {
        name: "linux.ftrace"
        ftrace_config {
            ftrace_events: "sched/sched_switch"
            ftrace_events: "power/cpu_frequency"
            ftrace_events: "power/cpu_idle"
        }
    }
}
duration_ms: 10000
EOF

# Pull trace
adb pull /data/misc/perfetto-traces/trace .
```

**Open in**: https://ui.perfetto.dev

**What to look for**:
- Main thread blocked by other processes
- CPU frequency throttling
- Thread priority inversions
- I/O wait times

---

## Performance Metrics

### Key Performance Indicators

#### Startup Time

**iOS**:
```swift
// Measure via Instruments or Xcode Organizer
// Target: <1s cold start

// Log startup time
func application(_ application: UIApplication, 
                didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    let startupTime = ProcessInfo.processInfo.systemUptime
    print("App started in \(startupTime)s")
    return true
}
```

**Android**:
```kotlin
// Measure via Logcat
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        val startTime = SystemClock.elapsedRealtime()
        
        // App initialization
        
        val endTime = SystemClock.elapsedRealtime()
        Log.d("Startup", "Time: ${endTime - startTime}ms")
    }
}

// Or use App Startup library
```

**Optimization strategies**:
1. Lazy initialize non-critical components
2. Use baseline profiles (Android)
3. Defer heavy SDK initialization
4. Load initial data asynchronously

#### Frame Rate

**Target**: Consistent 60 FPS (or 120 FPS on high-refresh displays)

**Measuring**:

iOS:
```swift
// Use CADisplayLink for accurate FPS measurement
class FPSCounter {
    private var displayLink: CADisplayLink?
    private var lastTimestamp: CFTimeInterval = 0
    private var frameCount: Int = 0
    
    func start() {
        displayLink = CADisplayLink(target: self, selector: #selector(tick))
        displayLink?.add(to: .main, forMode: .common)
    }
    
    @objc private func tick(link: CADisplayLink) {
        if lastTimestamp == 0 {
            lastTimestamp = link.timestamp
        }
        
        frameCount += 1
        
        let elapsed = link.timestamp - lastTimestamp
        if elapsed >= 1.0 {
            let fps = Double(frameCount) / elapsed
            print("FPS: \(fps)")
            
            lastTimestamp = link.timestamp
            frameCount = 0
        }
    }
}
```

Android:
```kotlin
class FPSMonitor(private val view: View) {
    private var frameCount = 0
    private var lastTime = System.nanoTime()
    
    fun start() {
        view.choreographer.postFrameCallback(object : Choreographer.FrameCallback {
            override fun doFrame(frameTimeNanos: Long) {
                frameCount++
                
                val elapsed = (frameTimeNanos - lastTime) / 1_000_000_000.0
                if (elapsed >= 1.0) {
                    val fps = frameCount / elapsed
                    Log.d("FPS", "FPS: $fps")
                    
                    lastTime = frameTimeNanos
                    frameCount = 0
                }
                
                view.choreographer.postFrameCallback(this)
            }
        })
    }
}
```

#### Memory Usage

**Healthy ranges**:
- Small app: 50-100 MB
- Medium app: 100-200 MB
- Large app: 200-500 MB
- >500 MB: Investigate leaks

**Monitor**:
```swift
// iOS
let memoryUsage = reportMemory()
print("Memory: \(memoryUsage / 1024 / 1024) MB")

func reportMemory() -> UInt64 {
    var info = mach_task_basic_info()
    var count = mach_msg_type_number_t(MemoryLayout<mach_task_basic_info>.size)/4
    
    let kerr: kern_return_t = withUnsafeMutablePointer(to: &info) {
        $0.withMemoryRebound(to: integer_t.self, capacity: 1) {
            task_info(mach_task_self_, task_flavor_t(MACH_TASK_BASIC_INFO), $0, &count)
        }
    }
    
    return info.resident_size
}
```

```kotlin
// Android
val runtime = Runtime.getRuntime()
val usedMemory = (runtime.totalMemory() - runtime.freeMemory()) / 1024 / 1024
Log.d("Memory", "Used: ${usedMemory} MB")
```

---

## Common Performance Issues

### 1. Main Thread Block

**Symptom**: UI freezes during operations

**Cause**: Network, database, or heavy computation on main thread

**Fix**:
```swift
// BAD
func loadData() {
    let data = try! Data(contentsOf: url) // ❌ Blocks main thread
    updateUI(data)
}

// GOOD
func loadData() async {
    do {
        let data = try await URLSession.shared.data(from: url).0
        await MainActor.run {
            updateUI(data)
        }
    } catch {
        // Handle error
    }
}
```

### 2. Excessive Redraws

**Symptom**: High GPU usage, battery drain

**Cause**: Animating non-compositor properties or triggering unnecessary layouts

**Fix**:
```swift
// BAD - Animates frame (triggers layout)
UIView.animate(withDuration: 0.3) {
    view.frame.size.width = 200 // ❌ Layout recalculation
}

// GOOD - Animates transform (compositor)
UIView.animate(withDuration: 0.3) {
    view.transform = CGAffineTransform(scaleX: 2.0, y: 1.0)
}
```

### 3. Memory Leaks

**Common patterns**:

```swift
// 1. Strong reference cycles
class ViewController {
    var closure: (() -> Void)?
    
    func setup() {
        closure = {
            self.doSomething() // ❌ Captures self
        }
    }
}

// Fix
closure = { [weak self] in
    self?.doSomething()
}

// 2. Timer not invalidated
class TimerViewController {
    var timer: Timer?
    
    func start() {
        timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { _ in
            self.update() // ❌ Retains self
        }
    }
}

// Fix
func start() {
    timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] _ in
        self?.update()
    }
}

deinit {
    timer?.invalidate()
}
```

### 4. Inefficient List Rendering

**Symptom**: Scrolling jank in lists

**Cause**: Complex cell layouts, expensive cell preparation

**Fix**:
```kotlin
// BAD - Expensive work in onBindViewHolder
override fun onBindViewHolder(holder: ViewHolder, position: Int) {
    val item = items[position]
    holder.image.setImageBitmap(
        BitmapFactory.decodeFile(item.imagePath) // ❌ I/O on main thread
    )
}

// GOOD - Use image loading library
override fun onBindViewHolder(holder: ViewHolder, position: Int) {
    val item = items[position]
    Glide.with(holder.itemView.context)
        .load(item.imagePath)
        .into(holder.image) // ✅ Async loading with caching
}
```

---

## Optimization Strategies

### iOS Optimization Checklist

- [ ] Use `shouldRasterize` for static complex views
- [ ] Avoid `cornerRadius` + `masksToBounds` together
- [ ] Set `shadowPath` explicitly when using shadows
- [ ] Mark views as `opaque` when possible
- [ ] Use `UIStackView` to reduce view hierarchy
- [ ] Profile with Instruments before optimizing
- [ ] Test on oldest supported device
- [ ] Use `@MainActor` for UI updates
- [ ] Implement proper deinitialization

### Android Optimization Checklist

- [ ] Implement Baseline Profiles
- [ ] Use `ConstraintLayout` to flatten hierarchy
- [ ] Enable R8 full mode for release builds
- [ ] Use `RecyclerView` with proper ViewHolder recycling
- [ ] Implement `DiffUtil` for efficient list updates
- [ ] Use `Flow` instead of `LiveData` for better performance
- [ ] Profile with Android Studio Profiler
- [ ] Test on low-end devices (1-2GB RAM)
- [ ] Use Jetpack Compose with strong skipping mode

### React Native Optimization Checklist

- [ ] Use Reanimated 3+ for animations
- [ ] Enable New Architecture (Fabric + TurboModules)
- [ ] Use `FlatList` with proper memoization
- [ ] Implement native modules for heavy operations
- [ ] Use `useMemo` and `useCallback` appropriately
- [ ] Enable Hermes engine
- [ ] Profile with Flipper
- [ ] Test on mid-range Android devices

---

## Performance Budget Template

Create a performance budget for your app:

```
Metric                  | Target  | Maximum | Current
------------------------|---------|---------|--------
Cold start time         | 1.5s    | 2.5s    | _____
Frame rate (avg)        | 58 FPS  | 55 FPS  | _____
Frame rate (p95)        | 60 FPS  | 57 FPS  | _____
Memory (idle)           | 80 MB   | 150 MB  | _____
Memory (peak)           | 200 MB  | 350 MB  | _____
Battery (1hr active)    | 8%      | 12%     | _____
App size (iOS)          | 30 MB   | 50 MB   | _____
App size (Android)      | 25 MB   | 40 MB   | _____
Network usage (session) | 5 MB    | 10 MB   | _____
```

Monitor these metrics in CI/CD and alert on regressions.

---

## Conclusion

Performance optimization is an ongoing process:

1. **Measure** with profiling tools
2. **Identify** bottlenecks using data
3. **Optimize** the slowest parts first
4. **Verify** improvements with profiling
5. **Monitor** in production with analytics

**Remember**: Don't optimize prematurely. Profile first, then optimize based on real data.
