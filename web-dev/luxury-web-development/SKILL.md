---
name: luxury-web-development
description: Framework for building high-visual, premium e-commerce experiences with 3D interactions and glassmorphism.
---

# Luxury Web Development Framework

Guidelines for creating "Next-Gen" luxury storefronts (e.g., Luxe Aura) that prioritize high-visual impact, spatial transitions, and a premium user experience.

## Updated for Termux Sub-Agent Architecture (May 2026)

Includes guidelines for deploying secure monitoring layers alongside UI with asynchronous Flask backends and periodic defense scanning via Hermes-native cronjobs.

## Visual Design Language
- **Glassmorphism:** Use `backdrop-filter: blur()`, semi-transparent white/dark backgrounds, and thin, light borders (`rgba(255, 255, 255, 0.2)`) to create a layered, crystalline feel.
- **Depth & Space:** Implement a z-axis depth system. Elements should not just slide; they should move "towards" or "away from" the viewer.
- **Typography:** Use a high-contrast pairing—Serif for luxury headings (e.g., Playfair Display) and clean Sans-Serif for readability (e.g., Inter, Plus Jakarta Sans).
- **Color Palette:** Start with a neutral base (white/off-white/deep black) and use a single, vibrant accent color (e.g., Indigo, Gold) for primary CTAs and highlights.

## 3D & Animation Standards
- **Three.js Integration:** Use Three.js for hero elements. Prefer `MeshPhysicalMaterial` for crystalline or metallic luxury looks.
- **Spatial Transitions:** 
    - Avoid standard linear slides. Use 3D transforms (`translateZ`, `rotateX`) and `perspective` on the parent container.
    - **Easing:** Always use high-end cubic-bezier curves (e.g., `cubic-bezier(0.2, 0.8, 0.2, 1)`) to mimic organic, weighted movement.
    - **UX Caution:** Complex 3D swipe-navigation across entire page sections can be buggy and feel unintuitive to users. Only use for focused micro-interactions; for primary page navigation, prefer smooth scrolling or standard transitions to ensure a polished, stable experience.
- **Performance:** 
    - Apply `will-change: transform` to elements undergoing spatial shifts to leverage the GPU and prevent flickering.
    - Ensure 3D anmations are disabled or simplified for low-power devices.

## Interaction Design
- **Touch-Sensing:** When implementing swipes for navigation, use a dual-threshold system:
    - `SWIPE_THRESHOLD`: Minimum distance (e.g., 80px) to distinguish a swipe from a scroll.
    - `HORIZONTAL_THRESHOLD`: Filter out horizontal noise to prevent accidental vertical triggers.
- **Micro-interactions:** Add 3D tilt effects to cards using `rotateX` and `rotateY` based on mouse/touch position relative to the card center.

## Implementation Pitfalls
- **Z-Fighting/Flickering:** When using `translateZ`, ensure the parent has a defined `perspective` value (e.g., `2000px`) and `transform-style: preserve-3d`.
- **Layout Shift:** Avoid changing `display: none` during 3D transitions; use `opacity: 0` and `pointer-events: none` to maintain the spatial position.
- **Cache Issues:** When updating styles in a live-synced environment, use version queries (`style.css?v=1.1`) to force browser cache clears.
