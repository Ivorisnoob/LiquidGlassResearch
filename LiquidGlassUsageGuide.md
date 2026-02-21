# Liquid Glass Usage Guide (Telegram Android)

This document explains how the `LiquidGlassEffect` shader is practically wired up and used across the Telegram Android application's UI components.

## 1. Feature Toggling (LiteMode)
Because this effect requires Android 13+ (API 33) and relies on GPU shading (`RuntimeShader`), it is guarded by a "Lite Mode" flag. Throughout the app's components, you will see it being explicitly enabled or disabled based on user performance settings.

```java
// Example from TopicsFragment, ProfileActivity, ChatActivity, etc.
factory.setLiquidGlassEffectAllowed(LiteMode.isEnabled(LiteMode.FLAG_LIQUID_GLASS));
```

## 2. Abstraction via Factories
Telegram abstracts the complex blurring and glass functionalities behind factories (namely, `BlurredBackgroundDrawableViewFactory` or `BlurProvider`). 

UI elements like `ChatAttachAlert`, `ShareAlert`, `MainTabsActivity`, and `DialogsActivity` do not instantiate `LiquidGlassEffect` directly. Instead:
1. The UI component requests a blurred background drawable from a "Glass" or "Liquid Glass" factory.
2. The UI component tells the factory if the liquid glass effect is allowed (via `LiteMode`).
3. If allowed, the factory applies it to the background drawable.

## 3. The Core Wrapper: `BlurredBackgroundDrawableRenderNode`
The actual `LiquidGlassEffect` instance lives inside `BlurredBackgroundDrawableRenderNode`. This is a hardware-accelerated wrapper that manages drawing the glass effect on a background source.

### Setup
When `setLiquidGlassEffectAllowed()` is called on this node, it instantiates the effect attached to a dedicated `renderNodeFill`.

```java
@RequiresApi(api = Build.VERSION_CODES.TIRAMISU)
public void setLiquidGlassEffectAllowed() {
    liquidGlassEffect = new LiquidGlassEffect(renderNodeFill);
}
```

### The Render Pipeline (`updateDisplayList`)
The node operates using two `RenderNode`s:
1. **`renderNodeFill`**: This node is responsible for rendering the content *underneath* the glass (the `source`). The `LiquidGlassEffect` is attached *to this node*. It applies the refraction to anything drawn into it.
2. **`renderNode`**: The parent node that handles clipping, outlines, and border strokes.

When the display list is updated (e.g., on scrolling, moving, or resizing), the wrapper feeds the current spatial dimensions, corner radii, and theme colors into the effect:

```java
c = renderNodeFill.beginRecording();
c.save();
c.translate(-sL, -sT); // Shift relative to screen

// 1. Give the shader the latest sizing and physics parameters
if (liquidGlassEffect != null && Build.VERSION.SDK_INT >= 33) {
    liquidGlassEffect.update(
        0, 0, boundProps.boundsWithPadding.width(), boundProps.boundsWithPadding.height(),
        boundProps.shaderRadii[0], boundProps.shaderRadii[2], boundProps.shaderRadii[4], boundProps.shaderRadii[6],
        boundProps.liquidThickness <= 0 ? dp(11) : boundProps.liquidThickness,
        boundProps.liquidIntensity,
        boundProps.liquidIndex,
        backgroundColor // Used for tinting
    );
}

// 2. Draw the background under the view
source.draw(c, sL, sT, sR, sB);
c.restore();
renderNodeFill.endRecording();
```

After recording the refracted background into `renderNodeFill`, it draws `renderNodeFill` onto the main `renderNode`, alongside any strokes (borders) or solid colors fallback (if the glass effect isn't turned on).

## 4. Where is it used?
Based on the codebase analysis, Telegram applies the Liquid Glass factory in many premium, floating, or overlapping UI surfaces to create depth:

*   **Alerts & Sheets**: `ShareAlert`, `ChatAttachAlert`, `StarGiftPreviewSheet`
*   **Main Navigation Surfaces**: `MainTabsActivity`, `GlassTabsView` (The bottom navigation bar)
*   **Headers and Headers Overlays**: `TopicsFragment`, `ChatActivity`, `DialogsActivity`
*   **Popups**: `EmojiView` (The background behind custom emoji pickers)
*   **Profiles**: `ProfileActivity`, `CallLogActivity`

### Summary 
1. **Gatekeeping:** Checked against `LiteMode`.
2. **Factory Management:** Configured on Blur/Glass element factories.
3. **Execution:** Implemented as a hardware-level `RenderEffect` on a background-fetching `RenderNode` that continuously syncs the surface's corner radii, size, and tint with the Shader inputs.
