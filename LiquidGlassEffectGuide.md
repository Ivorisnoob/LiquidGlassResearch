# Liquid Glass Effect Analysis (Telegram Android)

This document analyzes the "Liquid Glass" rendering effect found in the Telegram Android codebase and explains how to reproduce it in other Kotlin applications.

## Overview
Telegram's Liquid Glass effect provides a physically-based, realistic glass refraction over UI elements. 
It makes use of Android 13+ (`API 33`) `RuntimeShader` (AGSL - Android Graphics Shading Language) combined with a `RenderEffect`.

The core logic uses a **Signed Distance Field (SDF)** to model a 3D rounded shape (like a glass pill or rounded rectangle) and calculate surface normals. These normals are then used to refract the underlying content pixel coordinates.

---

## 1. The Shader Code (AGSL)

The actual visual magic is defined in `liquid_glass_shader.agsl`. AGSL uses syntax similar to GLSL.

```glsl
uniform shader img; // The content being blurred/distorted behind the effect

uniform float2 resolution;
uniform float2 center;
uniform float2 size;
uniform float4 radius; // Radii for the 4 corners
uniform float thickness; // The thickness/depth of the glass edge
uniform float refract_index; // Index of refraction (IoR)
uniform float refract_intensity;
uniform float4 foreground_color_premultiplied; // Overlay color (e.g., white with some alpha)

// Function to calculate the Signed Distance Field of a rounded rectangle
half sdfRect(half2 p, half4 r) {
  r.xy = (p.x > 0.0) ? r.xy : r.zw;
  r.x  = (p.y > 0.0) ? r.x  : r.y;
  half2 q = abs(p) - size + r.x;
  return length(max(q, 0.0)) + min(max(q.x, q.y), 0.0) - r.x;
}

// Alpha compositing function for a colored tint on top of the glass
half4 srcOver(half4 src, half4 dst) {
    half3 outRGB = (src.rgb + dst.rgb * (1.0 - src.a));
    float outA = src.a + (1.0 - src.a) * dst.a;
    return half4(outRGB, outA);
}

half4 main(in float2 fragCoord) {
  half2 p = fragCoord - center; // Translate to the center of the glass element
  half sd = sdfRect(p, radius); // Get distance to the edge of the shape
  half2 uv = fragCoord;
  
  // If inside the shape (sd < 0) or slightly near it
  if (sd < 0.0) {
    // Sample the SDF nearby to calculate derivatives (approximating normals)
    half sdX = sdfRect(p + half2(1.0, 0.0), radius);
    half sdY = sdfRect(p + half2(0.0, 1.0), radius);

    // Calculate the 3D Normal of the glass surface
    half n_cos = max(thickness + sd, 0.0) / thickness;
    half n_cos2 = n_cos * n_cos;
    half n_sin = sqrt(1.0 - n_cos2);
    half3 normal = normalize(half3((sdX - sd) * n_cos, (sdY - sd) * n_cos, n_sin));

    // Refract a ray coming straight into the screen (0, 0, -1)
    half3 refract_vec = refract(half3(0.0, 0.0, -1.0), normal, 1.0 / refract_index);
    
    // Determine the depth/height of the glass at this pixel
    half h = sd < -thickness ? thickness : sqrt(sd * (-2.0 * thickness - sd));
    half refract_length = (h + 8.0 * thickness) / -refract_vec.z;

    // Offset the sampled texture coordinate based on the refraction calculation
    uv += refract_vec.xy * refract_length * refract_intensity;
  }

  // Multiply the original image sample with the tint color
  return srcOver(half4(foreground_color_premultiplied), img.eval(uv));
}
```

---

## 2. The Kotlin Implementation Interface

To use this shader effect in your own application (Requires API 33+), you can wrap it in a custom class. In Telegram, they attach this onto a `RenderNode` and pass it to a background view. 

Here is how you can use it generically in a modern Kotlin app:

```kotlin
import android.graphics.Color
import android.graphics.RenderEffect
import android.graphics.RuntimeShader
import android.os.Build
import android.view.View
import androidx.annotation.RequiresApi

@RequiresApi(Build.VERSION_CODES.TIRAMISU)
class LiquidGlassHelper(private val code: String) {

    private val shader: RuntimeShader = RuntimeShader(code)
    
    // Call this specifically when the size, shape, or colors change
    fun updateEffect(
        view: View,
        width: Float, height: Float,
        centerX: Float, centerY: Float,
        radiusLT: Float, radiusRT: Float,
        radiusRB: Float, radiusLB: Float,
        thickness: Float = 15f,
        intensity: Float = 1.0f,
        refractIndex: Float = 1.5f, // Glass is typically 1.5 IoR
        foregroundColor: Int = Color.parseColor("#44FFFFFF") // Transparent white tint
    ) {
        // Validate corner radii so they don't break the SDF
        var rLT = radiusLT
        var rLB = radiusLB
        var rRT = radiusRT
        var rRB = radiusRB
        
        if (rLT + rLB > height) {
            val a = rLT / (rLT + rLB)
            rLT = height * a
            rLB = height * (1.0f - a)
        }
        if (rRT + rRB > height) {
            val a = rRT / (rRT + rRB)
            rRT = height * a
            rRB = height * (1.0f - a)
        }

        // Premultiply color components for the shader
        val a = Color.alpha(foregroundColor) / 255f
        val r = Color.red(foregroundColor) / 255f * a
        val g = Color.green(foregroundColor) / 255f * a
        val b = Color.blue(foregroundColor) / 255f * a

        // Send uniform data to the AGSL shader
        shader.setFloatUniform("resolution", width, height)
        shader.setFloatUniform("center", centerX, centerY)
        shader.setFloatUniform("size", width / 2f, height / 2f)
        shader.setFloatUniform("radius", rRB, rRT, rLB, rLT)
        shader.setFloatUniform("thickness", thickness)
        shader.setFloatUniform("refract_intensity", intensity)
        shader.setFloatUniform("refract_index", refractIndex)
        shader.setFloatUniform("foreground_color_premultiplied", r, g, b, a)

        // Create RenderEffect and apply it to the view
        val effect = RenderEffect.createRuntimeShaderEffect(shader, "img")
        view.setRenderEffect(effect)
    }
}
```

### Implementing in Jetpack Compose
To implement this cleanly in Jetpack Compose (again, Android 13+ only), you can utilize the `graphicsLayer` modifier:

```kotlin
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.RenderEffect
import androidx.compose.ui.graphics.RuntimeShader
import androidx.compose.ui.graphics.asComposeRenderEffect
import androidx.compose.ui.graphics.graphicsLayer

val LiquidGlassShaderCode = """ ... (insert AGSL code here) ... """

@RequiresApi(Build.VERSION_CODES.TIRAMISU)
fun Modifier.liquidGlassEffect(
    width: Float, height: Float,
    cornerRadius: Float = 0f,
    tintColor: Color = Color.White.copy(alpha = 0.2f)
) = this.then(Modifier.graphicsLayer {
    val shader = RuntimeShader(LiquidGlassShaderCode)
    
    // Size and Center
    shader.setFloatUniform("resolution", size.width, size.height)
    shader.setFloatUniform("center", size.width / 2f, size.height / 2f)
    shader.setFloatUniform("size", width / 2f, height / 2f)
    
    // All radii are equal in this simple compose wrapper
    shader.setFloatUniform("radius", cornerRadius, cornerRadius, cornerRadius, cornerRadius)
    
    // Physics constants
    shader.setFloatUniform("thickness", 20f)
    shader.setFloatUniform("refract_index", 1.5f)
    shader.setFloatUniform("refract_intensity", 1.0f)

    // Tint formatting
    val a = tintColor.alpha
    val r = tintColor.red * a
    val g = tintColor.green * a
    val b = tintColor.blue * a
    shader.setFloatUniform("foreground_color_premultiplied", r, g, b, a)
    
    // Apply blur optionally below the glass!
    // -> Blur is required if you want frosted glass, 
    // -> Telegram only runs this shader over ALREADY blurred backgrounds.

    renderEffect = android.graphics.RenderEffect
        .createRuntimeShaderEffect(shader, "img")
        .asComposeRenderEffect()
})
```

## Important Note
Telegram's `LiquidGlassEffect` only handles **refraction** (the warping of the image along the edges of the shape to simulate depth and glass bending). 

To achieve the full "Frosted UI" look, Telegram layers this *on top* of a heavy blur. In Android 13+, you can stack `RenderEffect` items. You should apply a `RenderEffect.createBlurEffect()` first, and then wrap that inside the runtime shader effect using `RenderEffect.createRuntimeShaderEffect(shader, "img")` if doing this on traditional views, or chaining `RenderEffects` in Jetpack Compose.
