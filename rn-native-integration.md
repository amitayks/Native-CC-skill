# React Native Native Integration Guide

**Complete guide to building native modules and UI components for React Native**

## Table of Contents

1. [When to Use Native Modules](#when-to-use-native-modules)
2. [Architecture Overview](#architecture-overview)
3. [Creating Native UI Components](#creating-native-ui-components)
4. [Creating Native Modules](#creating-native-modules)
5. [TurboModules (New Architecture)](#turbomodules-new-architecture)
6. [Fabric Native Components](#fabric-native-components)
7. [Best Practices](#best-practices)

---

## When to Use Native Modules

### ✅ Use Native When:

1. **Performance Critical**: Image processing, encryption, complex animations (120fps)
2. **Platform APIs**: Biometrics, health data, NFC, Bluetooth
3. **Existing Native Code**: Integrating legacy iOS/Android code
4. **Third-party SDKs**: Payment processors, analytics without RN support
5. **Custom Animations**: Physics-based motion requiring GPU acceleration

### ❌ Don't Use Native When:

1. **Simple UI**: Standard components work fine
2. **Network Requests**: JavaScript fetch/axios is sufficient
3. **State Management**: React state/Redux/MobX handle this
4. **Simple Animations**: Reanimated 3 handles 95% of cases
5. **Time Constraints**: Native development is slower than RN

---

## Architecture Overview

### Legacy Bridge Architecture

```
JavaScript Thread    Bridge (JSON serialization)    Native Thread
─────────────────────────────────────────────────────────────────
React Component  <──────────────────────────>  Native Module
                         Async                     
                     Message Passing            iOS/Android Code
```

**Limitations**:
- JSON serialization overhead
- Asynchronous only
- No direct memory access

### New Architecture (Fabric + TurboModules)

```
JavaScript Thread       JSI (Direct C++ calls)      Native Thread
─────────────────────────────────────────────────────────────────
React Component  <─────────────────────────────>  TurboModule
                     Synchronous possible          
                  Direct memory sharing         iOS/Android Code
```

**Benefits**:
- Zero serialization overhead
- Synchronous calls supported
- Type-safe with CodeGen
- Better performance

---

## Creating Native UI Components

### iOS Native UI Component

#### Step 1: Create View Manager

```swift
// RNCustomViewManager.swift
import UIKit
import React

@objc(RNCustomViewManager)
class RNCustomViewManager: RCTViewManager {
    
    override func view() -> UIView! {
        return RNCustomView()
    }
    
    override static func requiresMainQueueSetup() -> Bool {
        return true  // UI updates need main thread
    }
}
```

#### Step 2: Create Custom View

```swift
// RNCustomView.swift
import UIKit

class RNCustomView: UIView {
    
    private let label = UILabel()
    private var animator: UIViewPropertyAnimator?
    
    // React prop
    @objc var text: String = "" {
        didSet {
            label.text = text
        }
    }
    
    // React prop with animation
    @objc var progress: CGFloat = 0 {
        didSet {
            animateProgress()
        }
    }
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        setupView()
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    private func setupView() {
        addSubview(label)
        label.translatesAutoresizingMaskIntoConstraints = false
        NSLayoutConstraint.activate([
            label.centerXAnchor.constraint(equalTo: centerXAnchor),
            label.centerYAnchor.constraint(equalTo: centerYAnchor)
        ])
    }
    
    private func animateProgress() {
        animator?.stopAnimation(true)
        
        let targetScale = 1.0 + (progress * 0.5)
        
        animator = UIViewPropertyAnimator(
            duration: 0.3,
            dampingRatio: 0.7
        ) {
            self.transform = CGAffineTransform(scaleX: targetScale, y: targetScale)
        }
        
        animator?.startAnimation()
    }
}
```

#### Step 3: Bridge Exports

```objc
// RNCustomViewManager.m
#import <React/RCTViewManager.h>

@interface RCT_EXTERN_MODULE(RNCustomViewManager, RCTViewManager)

// Export props
RCT_EXPORT_VIEW_PROPERTY(text, NSString)
RCT_EXPORT_VIEW_PROPERTY(progress, CGFloat)

// Export methods (if needed)
RCT_EXTERN_METHOD(resetView:(nonnull NSNumber *)reactTag)

@end
```

#### Step 4: Export Methods (Optional)

```swift
// In RNCustomViewManager
@objc
func resetView(_ reactTag: NSNumber) {
    DispatchQueue.main.async {
        guard let view = self.bridge.uiManager.view(forReactTag: reactTag) as? RNCustomView else {
            return
        }
        
        UIView.animate(withDuration: 0.3) {
            view.transform = .identity
        }
    }
}
```

### Android Native UI Component

#### Step 1: Create View Manager

```kotlin
// RNCustomViewManager.kt
package com.myapp.customview

import android.view.View
import com.facebook.react.uimanager.SimpleViewManager
import com.facebook.react.uimanager.ThemedReactContext
import com.facebook.react.uimanager.annotations.ReactProp

class RNCustomViewManager : SimpleViewManager<RNCustomView>() {
    
    override fun getName() = "RNCustomView"
    
    override fun createViewInstance(reactContext: ThemedReactContext): RNCustomView {
        return RNCustomView(reactContext)
    }
    
    @ReactProp(name = "text")
    fun setText(view: RNCustomView, text: String?) {
        view.setText(text ?: "")
    }
    
    @ReactProp(name = "progress")
    fun setProgress(view: RNCustomView, progress: Float) {
        view.setProgress(progress)
    }
    
    // Export methods
    override fun getCommandsMap(): Map<String, Int> {
        return mapOf("resetView" to COMMAND_RESET)
    }
    
    override fun receiveCommand(root: RNCustomView, commandId: Int, args: ReadableArray?) {
        when (commandId) {
            COMMAND_RESET -> root.reset()
        }
    }
    
    companion object {
        private const val COMMAND_RESET = 1
    }
}
```

#### Step 2: Create Custom View

```kotlin
// RNCustomView.kt
package com.myapp.customview

import android.content.Context
import android.view.Gravity
import android.widget.FrameLayout
import android.widget.TextView
import androidx.dynamicanimation.animation.SpringAnimation
import androidx.dynamicanimation.animation.SpringForce

class RNCustomView(context: Context) : FrameLayout(context) {
    
    private val textView: TextView = TextView(context).apply {
        gravity = Gravity.CENTER
    }
    
    private var scaleXAnimation: SpringAnimation? = null
    private var scaleYAnimation: SpringAnimation? = null
    
    init {
        addView(textView, LayoutParams(
            LayoutParams.WRAP_CONTENT,
            LayoutParams.WRAP_CONTENT,
            Gravity.CENTER
        ))
    }
    
    fun setText(text: String) {
        textView.text = text
    }
    
    fun setProgress(progress: Float) {
        val targetScale = 1f + (progress * 0.5f)
        animateScale(targetScale)
    }
    
    private fun animateScale(targetScale: Float) {
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
    
    fun reset() {
        animateScale(1f)
    }
}
```

#### Step 3: Register in Package

```kotlin
// MyAppPackage.kt
package com.myapp

import com.facebook.react.ReactPackage
import com.facebook.react.bridge.NativeModule
import com.facebook.react.bridge.ReactApplicationContext
import com.facebook.react.uimanager.ViewManager
import com.myapp.customview.RNCustomViewManager

class MyAppPackage : ReactPackage {
    override fun createViewManagers(
        reactContext: ReactApplicationContext
    ): List<ViewManager<*, *>> {
        return listOf(RNCustomViewManager())
    }
    
    override fun createNativeModules(
        reactContext: ReactApplicationContext
    ): List<NativeModule> {
        return emptyList()
    }
}
```

### TypeScript Interface

```typescript
// CustomView.tsx
import { requireNativeComponent, UIManager, findNodeHandle, ViewProps } from 'react-native';
import { useRef } from 'react';

interface CustomViewProps extends ViewProps {
  text: string;
  progress: number;
}

const NativeCustomView = requireNativeComponent<CustomViewProps>('RNCustomView');

export function CustomView({ text, progress, ...props }: CustomViewProps) {
  const viewRef = useRef(null);
  
  const resetView = () => {
    const handle = findNodeHandle(viewRef.current);
    if (handle) {
      UIManager.dispatchViewManagerCommand(
        handle,
        'resetView',
        []
      );
    }
  };
  
  return (
    <NativeCustomView
      ref={viewRef}
      text={text}
      progress={progress}
      {...props}
    />
  );
}
```

---

## Creating Native Modules

### iOS Native Module

```swift
// RNAnimationUtils.swift
import Foundation
import UIKit

@objc(RNAnimationUtils)
class RNAnimationUtils: NSObject {
    
    // Synchronous method
    @objc
    func calculateSpringDuration(
        _ damping: Double,
        stiffness: Double
    ) -> Double {
        return 4.0 * sqrt(1.0 / stiffness) * damping
    }
    
    // Asynchronous method
    @objc
    func generateLottieFrame(
        _ config: NSDictionary,
        resolver: @escaping RCTPromiseResolveBlock,
        rejecter: @escaping RCTPromiseRejectBlock
    ) {
        DispatchQueue.global(qos: .userInitiated).async {
            do {
                // Complex computation
                let result = try self.processLottieFrame(config)
                resolver(result)
            } catch {
                rejecter("GENERATION_ERROR", error.localizedDescription, error)
            }
        }
    }
    
    // Method with callback
    @objc
    func watchSpringAnimation(
        _ callback: @escaping RCTResponseSenderBlock
    ) {
        var progress: Double = 0
        Timer.scheduledTimer(withTimeInterval: 0.016, repeats: true) { timer in
            progress += 0.016
            callback([progress])
            
            if progress >= 1.0 {
                timer.invalidate()
            }
        }
    }
    
    @objc
    static func requiresMainQueueSetup() -> Bool {
        return false  // Can initialize on background thread
    }
    
    private func processLottieFrame(_ config: NSDictionary) throws -> [String: Any] {
        // Implementation
        return ["frame": "data"]
    }
}
```

```objc
// RNAnimationUtils.m
#import <React/RCTBridgeModule.h>

@interface RCT_EXTERN_MODULE(RNAnimationUtils, NSObject)

RCT_EXTERN_METHOD(calculateSpringDuration:(double)damping
                  stiffness:(double)stiffness)

RCT_EXTERN_METHOD(generateLottieFrame:(NSDictionary *)config
                  resolver:(RCTPromiseResolveBlock)resolve
                  rejecter:(RCTPromiseRejectBlock)reject)

RCT_EXTERN_METHOD(watchSpringAnimation:(RCTResponseSenderBlock)callback)

@end
```

### Android Native Module

```kotlin
// RNAnimationUtilsModule.kt
package com.myapp.animation

import com.facebook.react.bridge.*
import kotlin.math.sqrt

class RNAnimationUtilsModule(reactContext: ReactApplicationContext) : 
    ReactContextBaseJavaModule(reactContext) {
    
    override fun getName() = "RNAnimationUtils"
    
    // Synchronous method (use sparingly!)
    @ReactMethod(isBlockingSynchronousMethod = true)
    fun calculateSpringDuration(damping: Double, stiffness: Double): Double {
        return 4.0 * sqrt(1.0 / stiffness) * damping
    }
    
    // Asynchronous method with Promise
    @ReactMethod
    fun generateLottieFrame(config: ReadableMap, promise: Promise) {
        try {
            // Do work on background thread
            Thread {
                try {
                    val result = processLottieFrame(config)
                    promise.resolve(result)
                } catch (e: Exception) {
                    promise.reject("GENERATION_ERROR", e.message, e)
                }
            }.start()
        } catch (e: Exception) {
            promise.reject("GENERATION_ERROR", e.message, e)
        }
    }
    
    // Method with callback
    @ReactMethod
    fun watchSpringAnimation(callback: Callback) {
        var progress = 0.0
        val timer = android.os.Handler(android.os.Looper.getMainLooper())
        
        val runnable = object : Runnable {
            override fun run() {
                progress += 0.016
                callback.invoke(progress)
                
                if (progress < 1.0) {
                    timer.postDelayed(this, 16)
                }
            }
        }
        
        timer.post(runnable)
    }
    
    private fun processLottieFrame(config: ReadableMap): WritableMap {
        val result = Arguments.createMap()
        result.putString("frame", "data")
        return result
    }
}
```

```kotlin
// Register in package
override fun createNativeModules(
    reactContext: ReactApplicationContext
): List<NativeModule> {
    return listOf(RNAnimationUtilsModule(reactContext))
}
```

### TypeScript Interface

```typescript
// AnimationUtils.ts
import { NativeModules } from 'react-native';

interface AnimationUtilsModule {
  calculateSpringDuration(damping: number, stiffness: number): number;
  generateLottieFrame(config: object): Promise<{ frame: string }>;
  watchSpringAnimation(callback: (progress: number) => void): void;
}

const { RNAnimationUtils } = NativeModules;

export const AnimationUtils = RNAnimationUtils as AnimationUtilsModule;

// Usage
const duration = AnimationUtils.calculateSpringDuration(20, 300);
const frame = await AnimationUtils.generateLottieFrame({ width: 100 });
AnimationUtils.watchSpringAnimation((progress) => {
  console.log('Progress:', progress);
});
```

---

## TurboModules (New Architecture)

### Benefits

- **Lazy loading**: Modules loaded only when needed
- **Type safety**: CodeGen generates type-safe bindings
- **Synchronous methods**: Direct JSI calls
- **Better performance**: No JSON serialization

### Creating a TurboModule

#### Step 1: Define Spec

```typescript
// specs/NativeAnimationUtils.ts
import { TurboModule, TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  calculateSpringDuration(damping: number, stiffness: number): number;
  generateLottieFrame(config: Object): Promise<Object>;
}

export default TurboModuleRegistry.getEnforcing<Spec>('AnimationUtils');
```

#### Step 2: iOS Implementation

```swift
// AnimationUtils.swift - TurboModule
import Foundation

@objc(AnimationUtils)
class AnimationUtils: NSObject, RCTTurboModule {
    
    @objc
    func calculateSpringDuration(_ damping: Double, stiffness: Double) -> Double {
        return 4.0 * sqrt(1.0 / stiffness) * damping
    }
    
    @objc
    func generateLottieFrame(
        _ config: NSDictionary,
        resolve: @escaping RCTPromiseResolveBlock,
        reject: @escaping RCTPromiseRejectBlock
    ) {
        // Implementation
    }
    
    @objc
    static func requiresMainQueueSetup() -> Bool {
        return false
    }
}
```

#### Step 3: Android Implementation

```kotlin
// AnimationUtilsTurboModule.kt
class AnimationUtilsModule(reactContext: ReactApplicationContext) : 
    NativeAnimationUtilsSpec(reactContext) {
    
    override fun getName() = "AnimationUtils"
    
    override fun calculateSpringDuration(damping: Double, stiffness: Double): Double {
        return 4.0 * sqrt(1.0 / stiffness) * damping
    }
    
    override fun generateLottieFrame(config: ReadableMap, promise: Promise) {
        // Implementation
    }
}
```

---

## Fabric Native Components

### Benefits Over Legacy

- **Direct UI updates**: No bridge latency
- **Concurrent rendering**: React 18 support
- **Type safety**: CodeGen-generated props
- **Better performance**: C++ shared core

### Creating Fabric Component

#### Step 1: Define Spec

```typescript
// CustomViewNativeComponent.ts
import codegenNativeComponent from 'react-native/Libraries/Utilities/codegenNativeComponent';
import { ViewProps } from 'react-native';

interface NativeProps extends ViewProps {
  text: string;
  progress: number;
}

export default codegenNativeComponent<NativeProps>('CustomView');
```

#### Step 2: Implementation (iOS/Android)

Similar to regular native components but with Fabric-specific APIs.

---

## Best Practices

### 1. Thread Safety

```swift
// BAD - Not thread-safe
@objc
func updateData(_ newData: NSDictionary) {
    self.data = newData  // ❌ May be called from any thread
}

// GOOD - Ensure main thread for UI updates
@objc
func updateData(_ newData: NSDictionary) {
    DispatchQueue.main.async {
        self.data = newData
    }
}
```

### 2. Memory Management

```swift
// BAD - Retain cycle
class MyModule: NSObject {
    var callback: RCTResponseSenderBlock?
    
    func doWork() {
        heavy Computation { result in
            self.callback?([result])  // ❌ Retains self
        }
    }
}

// GOOD - Weak reference
func doWork() {
    heavyComputation { [weak self] result in
        self?.callback?([result])
    }
}
```

### 3. Error Handling

```kotlin
// GOOD - Comprehensive error handling
@ReactMethod
fun riskyOperation(promise: Promise) {
    try {
        val result = doSomething()
        promise.resolve(result)
    } catch (e: IllegalArgumentException) {
        promise.reject("INVALID_ARGUMENT", "Invalid input: ${e.message}", e)
    } catch (e: IOException) {
        promise.reject("IO_ERROR", "Failed to read file: ${e.message}", e)
    } catch (e: Exception) {
        promise.reject("UNKNOWN_ERROR", "Unexpected error: ${e.message}", e)
    }
}
```

### 4. Performance

```typescript
// BAD - Frequent bridge calls
for (let i = 0; i < 1000; i++) {
  await NativeModule.processItem(items[i]);  // ❌ 1000 bridge crossings!
}

// GOOD - Batch operations
await NativeModule.processItems(items);  // ✅ Single bridge crossing
```

### 5. Type Safety

```typescript
// Use TypeScript interfaces for all native modules
interface MyNativeModule {
  doSomething(param: string): Promise<Result>;
  getValue(): number;
}

const { MyModule } = NativeModules;
export default MyModule as MyNativeModule;
```

---

## Testing Native Modules

### iOS Tests

```swift
// AnimationUtilsTests.swift
import XCTest
@testable import MyApp

class AnimationUtilsTests: XCTestCase {
    var module: RNAnimationUtils!
    
    override func setUp() {
        super.setUp()
        module = RNAnimationUtils()
    }
    
    func testSpringDurationCalculation() {
        let duration = module.calculateSpringDuration(20, stiffness: 300)
        XCTAssertEqual(duration, 0.461, accuracy: 0.001)
    }
    
    func testAsyncOperation() {
        let expectation = XCTestExpectation(description: "Async operation")
        
        module.generateLottieFrame([:], resolver: { result in
            XCTAssertNotNil(result)
            expectation.fulfill()
        }, rejecter: { _, _, _ in
            XCTFail("Should not reject")
        })
        
        wait(for: [expectation], timeout: 5.0)
    }
}
```

### Android Tests

```kotlin
// AnimationUtilsModuleTest.kt
class AnimationUtilsModuleTest {
    private lateinit var module: RNAnimationUtilsModule
    
    @Before
    fun setup() {
        val context = mock(ReactApplicationContext::class.java)
        module = RNAnimationUtilsModule(context)
    }
    
    @Test
    fun testSpringDurationCalculation() {
        val duration = module.calculateSpringDuration(20.0, 300.0)
        assertEquals(0.461, duration, 0.001)
    }
    
    @Test
    fun testAsyncOperation() = runBlocking {
        val promise = mock(Promise::class.java)
        val config = Arguments.createMap()
        
        module.generateLottieFrame(config, promise)
        
        verify(promise).resolve(any())
    }
}
```

---

## Debugging

### iOS Debugging

```swift
// Add logging
import os.log

let logger = OSLog(subsystem: "com.myapp", category: "NativeModule")

os_log("Method called with param: %@", log: logger, type: .info, param)
```

### Android Debugging

```kotlin
// Add logging
import android.util.Log

private val TAG = "RNAnimationUtils"

Log.d(TAG, "Method called with param: $param")
```

### React Native Debugger

```typescript
// In JavaScript, test native calls
console.log('Calling native module...');
const result = await NativeModule.doSomething();
console.log('Native result:', result);
```

---

## Migration Path

### From Legacy to New Architecture

1. **Enable New Architecture**: Update `react-native.config.js`
2. **Create Specs**: Define TurboModule/Fabric specs
3. **Update Native Code**: Implement new interfaces
4. **Test Thoroughly**: Ensure backward compatibility
5. **Document Changes**: Update team on new patterns

### Backward Compatibility

```typescript
// Support both architectures
import { TurboModuleRegistry } from 'react-native';
import { NativeModules } from 'react-native';

export const MyModule = TurboModuleRegistry.get<Spec>('MyModule') || NativeModules.MyModule;
```

---

## Resources

- [React Native New Architecture](https://reactnative.dev/architecture/landing-page)
- [TurboModules Documentation](https://reactnative.dev/docs/next/the-new-architecture/pillars-turbomodules)
- [Fabric Native Components](https://reactnative.dev/docs/next/the-new-architecture/pillars-fabric-components)
- [Native Modules iOS](https://reactnative.dev/docs/native-modules-ios)
- [Native Modules Android](https://reactnative.dev/docs/native-modules-android)

---

## Conclusion

Native modules unlock the full power of iOS and Android platforms while maintaining React Native's development experience. Use them strategically for performance-critical features and platform-specific functionality.

**Key Takeaways**:
- Use native modules sparingly - only when JavaScript isn't sufficient
- Prefer New Architecture (TurboModules/Fabric) for new projects
- Always handle errors comprehensively
- Test on both iOS and Android thoroughly
- Document your native APIs clearly for JavaScript consumers
