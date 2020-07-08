---
layout: post
title:  "shader 101"
categories: techart
---

# introduction

practical approach to learning shaders, not how they work on a technical level

# how to get started in unity or shadertoy

# breakdown of a simple texture mapped shader, whats happening

```hlsl
uniform sampler2d _MainTex;

fixed4 frag(v2f i) {
    float4 col = tex2D(_MainTex, i.uv);

    return col;
}
```

# what can we fuck around with this

```hlsl
uniform sampler2d _MainTex;

fixed4 frag(v2f i) {
    float4 col = tex2D(_MainTex, i.uv);
    col.r = 0;
    return col;
}
```

```hlsl
uniform sampler2d _MainTex;

fixed4 frag(v2f i) {
    float4 col = tex2D(_MainTex, i.uv);
    col = 1 - col;
    return col;
}
```

# vertex shaders

# shader inputs with unity

# post processing?