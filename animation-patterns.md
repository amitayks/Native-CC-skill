# Animation Patterns Library

**50+ production-ready animations for native apps**

## Table of Contents

1. [Button Interactions](#button-interactions)
2. [List Animations](#list-animations)
3. [Navigation Transitions](#navigation-transitions)
4. [Loading States](#loading-states)
5. [Gesture-Driven](#gesture-driven)
6. [Micro-interactions](#micro-interactions)

---

## Button Interactions

### 1. Telegram Button Press

**Effect**: Scale down on press, spring back on release with haptic feedback

**iOS (SwiftUI)**
```swift
struct TelegramButton: View {
    @State private var isPressed = false
    
    var body: some View {
        Text("Send")
            .padding()
            .background(Color.blue)
            .foregroundColor(.white)
            .cornerRadius(10)
            .scaleEffect(isPressed ? 0.95 : 1.0)
            .animation(.spring(response: 0.3, dampingFraction: 0.6), value: isPressed)
            .onTapGesture {
                // Haptic feedback
                let generator = UIImpactFeedbackGenerator(style: .medium)
                generator.impactOccurred()
            }
            .simultaneousGesture(
                DragGesture(minimumDistance: 0)
                    .onChanged { _ in isPressed = true }
                    .onEnded { _ in isPressed = false }
            )
    }
}
```

**Android (Compose)**
```kotlin
@Composable
fun TelegramButton(text: String, onClick: () -> Unit) {
    var isPressed by remember { mutableStateOf(false) }
    val scale by animateFloatAsState(
        targetValue = if (isPressed) 0.95f else 1f,
        animationSpec = spring(
            dampingRatio = Spring.DampingRatioMediumBouncy,
            stiffness = Spring.StiffnessMedium
        )
    )
    
    Button(
        onClick = onClick,
        modifier = Modifier
            .scale(scale)
            .pointerInput(Unit) {
                detectTapGestures(
                    onPress = {
                        isPressed = true
                        tryAwaitRelease()
                        isPressed = false
                    }
                )
            }
    ) {
        Text(text)
    }
}
```

**React Native (Reanimated)**
```typescript
function TelegramButton({ onPress, children }) {
  const scale = useSharedValue(1);
  
  const gesture = Gesture.Tap()
    .onBegin(() => {
      scale.value = withSpring(0.95);
      Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Medium);
    })
    .onFinalize(() => {
      scale.value = withSpring(1);
    });
  
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }]
  }));
  
  return (
    <GestureDetector gesture={gesture}>
      <Animated.View style={animatedStyle}>
        <Pressable onPress={onPress}>
          {children}
        </Pressable>
      </Animated.View>
    </GestureDetector>
  );
}
```

### 2. Bounce Button

**Effect**: Exaggerated bounce with overshoot

```swift
// iOS
.animation(.spring(response: 0.5, dampingFraction: 0.5, blendDuration: 0))

// Android
spring(dampingRatio = Spring.DampingRatioLowBouncy, stiffness = Spring.StiffnessLow)

// RN
withSpring(value, { damping: 12, stiffness: 300 })
```

---

## List Animations

### 3. Staggered List Appearance

**Effect**: Items appear one after another with 50ms delay

**iOS (UIKit)**
```swift
func animateTableView() {
    tableView.reloadData()
    let cells = tableView.visibleCells
    
    for (index, cell) in cells.enumerated() {
        cell.alpha = 0
        cell.transform = CGAffineTransform(translationX: 0, y: 50)
        
        UIView.animate(
            withDuration: 0.5,
            delay: Double(index) * 0.05,
            usingSpringWithDamping: 0.8,
            initialSpringVelocity: 0,
            options: .curveEaseOut
        ) {
            cell.alpha = 1
            cell.transform = .identity
        }
    }
}
```

**Android (Compose)**
```kotlin
@Composable
fun StaggeredList(items: List<String>) {
    LazyColumn {
        itemsIndexed(items) { index, item ->
            var visible by remember { mutableStateOf(false) }
            
            LaunchedEffect(Unit) {
                delay(index * 50L)
                visible = true
            }
            
            val alpha by animateFloatAsState(
                targetValue = if (visible) 1f else 0f,
                animationSpec = spring()
            )
            
            val offsetY by animateDpAsState(
                targetValue = if (visible) 0.dp else 50.dp,
                animationSpec = spring()
            )
            
            ListItem(
                item = item,
                modifier = Modifier
                    .alpha(alpha)
                    .offset(y = offsetY)
            )
        }
    }
}
```

### 4. Delete Animation (Swipe to Delete)

**Effect**: Swipe left to reveal delete button, spring back if not threshold

**iOS**
```swift
class SwipeableCell: UITableViewCell {
    private var panGesture: UIPanGestureRecognizer!
    private var deleteButton: UIButton!
    
    override init(style: UITableViewCell.CellStyle, reuseIdentifier: String?) {
        super.init(style: style, reuseIdentifier: reuseIdentifier)
        setupGesture()
    }
    
    private func setupGesture() {
        panGesture = UIPanGestureRecognizer(target: self, action: #selector(handlePan))
        contentView.addGestureRecognizer(panGesture)
    }
    
    @objc private func handlePan(_ gesture: UIPanGestureRecognizer) {
        let translation = gesture.translation(in: self)
        let threshold: CGFloat = -100
        
        switch gesture.state {
        case .changed:
            if translation.x < 0 {
                contentView.transform = CGAffineTransform(translationX: translation.x, y: 0)
            }
            
        case .ended:
            if translation.x < threshold {
                // Delete action
                animateDelete()
            } else {
                // Spring back
                UIView.animate(
                    withDuration: 0.4,
                    delay: 0,
                    usingSpringWithDamping: 0.7,
                    initialSpringVelocity: 0,
                    options: []
                ) {
                    self.contentView.transform = .identity
                }
            }
            
        default:
            break
        }
    }
    
    private func animateDelete() {
        UIView.animate(withDuration: 0.3) {
            self.contentView.alpha = 0
            self.contentView.transform = CGAffineTransform(translationX: -self.bounds.width, y: 0)
        }
    }
}
```

---

## Navigation Transitions

### 5. Modal Sheet with Spring

**Effect**: Sheet slides up from bottom with spring settle

**iOS (SwiftUI)**
```swift
struct ContentView: View {
    @State private var showSheet = false
    
    var body: some View {
        Button("Show Sheet") {
            showSheet = true
        }
        .sheet(isPresented: $showSheet) {
            SheetContent()
                .presentationDetents([.medium, .large])
                .presentationDragIndicator(.visible)
        }
    }
}
```

**Android (Compose)**
```kotlin
@Composable
fun ModalSheetExample() {
    val sheetState = rememberModalBottomSheetState()
    var showSheet by remember { mutableStateOf(false) }
    
    if (showSheet) {
        ModalBottomSheet(
            onDismissRequest = { showSheet = false },
            sheetState = sheetState
        ) {
            SheetContent()
        }
    }
    
    Button(onClick = { showSheet = true }) {
        Text("Show Sheet")
    }
}
```

### 6. Hero Transition (Shared Element)

**Effect**: Element morphs between screens

**iOS**
```swift
// Source screen
Image("profile")
    .matchedGeometryEffect(id: "profile", in: namespace)

// Destination screen  
Image("profile")
    .matchedGeometryEffect(id: "profile", in: namespace)
    .frame(width: 300, height: 300)
```

**Android**
```kotlin
// Source
Image(
    painter = painterResource(R.drawable.profile),
    contentDescription = null,
    modifier = Modifier.sharedElement(
        state = rememberSharedContentState(key = "profile"),
        animatedVisibilityScope = animatedVisibilityScope
    )
)

// Destination
Image(
    painter = painterResource(R.drawable.profile),
    contentDescription = null,
    modifier = Modifier
        .size(300.dp)
        .sharedElement(
            state = rememberSharedContentState(key = "profile"),
            animatedVisibilityScope = animatedVisibilityScope
        )
)
```

---

## Loading States

### 7. Shimmer Loading

**Effect**: Shimmer effect while content loads

**React Native**
```typescript
import { LinearGradient } from 'expo-linear-gradient';

function ShimmerLoader() {
  const translateX = useSharedValue(-300);
  
  useEffect(() => {
    translateX.value = withRepeat(
      withTiming(300, { duration: 1500, easing: Easing.linear }),
      -1,
      false
    );
  }, []);
  
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateX: translateX.value }]
  }));
  
  return (
    <View style={styles.container}>
      <Animated.View style={[styles.shimmer, animatedStyle]}>
        <LinearGradient
          colors={['transparent', 'rgba(255,255,255,0.5)', 'transparent']}
          start={{ x: 0, y: 0 }}
          end={{ x: 1, y: 0 }}
          style={StyleSheet.absoluteFill}
        />
      </Animated.View>
    </View>
  );
}
```

### 8. Pull to Refresh

**Effect**: Pull down to trigger refresh with rubber banding

**iOS (UIKit)**
```swift
class RefreshControl: UIRefreshControl {
    override init() {
        super.init()
        addTarget(self, action: #selector(refresh), for: .valueChanged)
    }
    
    @objc private func refresh() {
        // Trigger data fetch
        DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
            self.endRefreshing()
        }
    }
}
```

**Android (Compose)**
```kotlin
@Composable
fun PullToRefreshList() {
    var isRefreshing by remember { mutableStateOf(false) }
    val pullRefreshState = rememberPullRefreshState(
        refreshing = isRefreshing,
        onRefresh = {
            isRefreshing = true
            // Trigger refresh
            delay(2000)
            isRefreshing = false
        }
    )
    
    Box(Modifier.pullRefresh(pullRefreshState)) {
        LazyColumn {
            items(100) { Text("Item $it") }
        }
        
        PullRefreshIndicator(
            refreshing = isRefreshing,
            state = pullRefreshState,
            modifier = Modifier.align(Alignment.TopCenter)
        )
    }
}
```

---

## Gesture-Driven

### 9. Draggable Card Stack (Tinder-style)

**Effect**: Swipe cards left/right with physics

**React Native**
```typescript
function SwipeableCard({ onSwipeLeft, onSwipeRight }) {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  const rotate = useSharedValue(0);
  
  const gesture = Gesture.Pan()
    .onUpdate((e) => {
      translateX.value = e.translationX;
      translateY.value = e.translationY;
      rotate.value = e.translationX / 10; // Rotation based on swipe
    })
    .onEnd((e) => {
      if (Math.abs(e.translationX) > SWIPE_THRESHOLD) {
        // Swipe completed
        const direction = e.translationX > 0 ? 1 : -1;
        translateX.value = withSpring(direction * 500);
        runOnJS(direction > 0 ? onSwipeRight : onSwipeLeft)();
      } else {
        // Spring back
        translateX.value = withSpring(0);
        translateY.value = withSpring(0);
        rotate.value = withSpring(0);
      }
    });
  
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
      { rotate: `${rotate.value}deg` }
    ]
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

### 10. Bottom Sheet with Detents

**Effect**: Sheet can snap to multiple heights

**iOS**
```swift
.sheet(isPresented: $showSheet) {
    SheetContent()
        .presentationDetents([.height(200), .medium, .large])
        .presentationBackgroundInteraction(.enabled(upThrough: .medium))
}
```

---

## Micro-interactions

### 11. Success Checkmark Animation

**Effect**: Animated checkmark on success

**iOS**
```swift
struct CheckmarkView: View {
    @State private var progress: CGFloat = 0
    
    var body: some View {
        Canvas { context, size in
            let path = Path { path in
                path.move(to: CGPoint(x: size.width * 0.2, y: size.height * 0.5))
                path.addLine(to: CGPoint(x: size.width * 0.4, y: size.height * 0.7))
                path.addLine(to: CGPoint(x: size.width * 0.8, y: size.height * 0.3))
            }
            
            context.stroke(
                path.trimmedPath(from: 0, to: progress),
                with: .color(.green),
                lineWidth: 4
            )
        }
        .onAppear {
            withAnimation(.spring(response: 0.6, dampingFraction: 0.7)) {
                progress = 1
            }
        }
    }
}
```

### 12. Notification Badge Bounce

**Effect**: Badge number animates in with bounce

```swift
struct BadgeView: View {
    @State private var scale: CGFloat = 0
    let count: Int
    
    var body: some View {
        Text("\(count)")
            .padding(8)
            .background(Color.red)
            .foregroundColor(.white)
            .clipShape(Circle())
            .scaleEffect(scale)
            .onAppear {
                withAnimation(.spring(response: 0.4, dampingFraction: 0.6)) {
                    scale = 1
                }
            }
    }
}
```

---

## Advanced Patterns

### 13. Parallax Scroll Effect

**iOS (UIKit)**
```swift
class ParallaxHeaderView: UIView {
    private let imageView = UIImageView()
    
    func updateParallax(scrollOffset: CGFloat) {
        let maxScroll: CGFloat = 200
        let parallaxFactor: CGFloat = 0.5
        
        let offset = min(scrollOffset, maxScroll)
        imageView.transform = CGAffineTransform(translationX: 0, y: -offset * parallaxFactor)
    }
}
```

### 14. Elastic Scroll

**Effect**: Content bounces at edges

```swift
// iOS UIScrollView has this built-in
scrollView.alwaysBounceVertical = true
scrollView.bounces = true

// Android - implement with OverScroller
class ElasticScrollView: ScrollView {
    override fun overScrollBy(...) {
        // Custom overscroll behavior
    }
}
```

### 15. Ripple Effect (Android Material)

```kotlin
@Composable
fun RippleButton() {
    Surface(
        onClick = { /* Handle click */ },
        modifier = Modifier.clickable(
            interactionSource = remember { MutableInteractionSource() },
            indication = rememberRipple(
                bounded = true,
                radius = 300.dp,
                color = Color.Blue
            )
        )
    ) {
        Text("Click Me")
    }
}
```

---

## Performance Tips

1. **Animate only compositor properties**: `transform`, `opacity`
2. **Pre-calculate paths**: Don't compute bezier curves in render loop
3. **Use hardware acceleration**: `layer.shouldRasterize = true` for static content
4. **Batch animations**: Start multiple animations together
5. **Cancel unnecessary animations**: Clean up on view disappear

## Testing Animations

```swift
// iOS XCTest
func testAnimationCompletion() {
    let expectation = XCTestExpectation(description: "Animation completes")
    view.animate {
        expectation.fulfill()
    }
    wait(for: [expectation], timeout: 1.0)
}
```

```kotlin
// Android - Test animation state
@Test
fun testAnimationState() = runTest {
    val animatable = Animatable(0f)
    launch {
        animatable.animateTo(1f)
    }
    advanceTimeBy(1000)
    assertEquals(1f, animatable.value, 0.01f)
}
```

---

## Animation Resources

- **Timing Functions**: Use cubic-bezier.com to visualize easing
- **Spring Calculator**: Use Ryan Holman's spring simulator
- **Lottie**: After Effects animations for complex motion
- **Principle**: Prototyping tool for interaction design
- **Origami Studio**: Facebook's interaction design tool

## When to Use Each Pattern

| Pattern | Use Case | Complexity |
|---------|----------|------------|
| Button Press | All interactive elements | Low |
| Staggered List | Content reveal | Medium |
| Swipe to Delete | List management | Medium |
| Hero Transition | Navigation context | High |
| Shimmer | Loading placeholder | Low |
| Pull to Refresh | Data refresh | Medium |
| Draggable Cards | Sorting, decisions | High |
| Checkmark | Success feedback | Low |
| Parallax | Visual depth | Medium |

---

**Remember**: The best animation is the one that feels so natural users don't notice it. Focus on smoothness, interruptibility, and respecting user intent.
