---
layout: page
title: Verba – Privacy Policy
permalink: /verba/privacy/
nav_exclude: true
---

# Verba – Privacy Policy

**Last updated: June 15, 2026**

## Overview

Verba is a translation app for iOS and macOS developed by S4Y Solutions. This policy explains what data the app accesses and how it is used.

## Data We Collect

### Text You Translate

When you request a translation, the source text is sent to our translation backend over HTTPS. We do not store translated text or associate it with any personal identity.

### Anonymous Device Identity

Verba authenticates requests using an RSA key pair generated and stored in your device's Keychain. We receive only the public key hash — no name, email, Apple ID, or device identifier is ever transmitted. This identity is anonymous and cannot be used to track you across devices or apps.

### App Preferences

Mode, quality, provider, and language preferences are stored locally in `UserDefaults` on your device. They are never transmitted to our servers.

## Clipboard Access

If "Monitor Clipboard" is enabled (on by default), Verba reads your clipboard when the app becomes active and uses the content as translation input. Clipboard content is processed locally and sent to the translation backend only when a translation is requested. Verba never stores or forwards clipboard data for any other purpose.

You can disable clipboard monitoring at any time via the app menu.

## Data Sharing

We do not sell, rent, or share your data with third parties. Translation requests are processed by our own backend. No analytics SDKs or advertising frameworks are included in the app.

## Data Retention

Translation requests are not stored on our servers beyond the time needed to produce and return the translation response.

## Security

All communication between the app and our backend uses HTTPS. Authentication uses RSA-SHA256 signed tokens — no passwords or API keys are stored on your device or transmitted.

## Children's Privacy

Verba is not directed at children under 13. We do not knowingly collect data from children.

## Changes to This Policy

If this policy changes materially, we will update the "Last updated" date above. Continued use of the app after changes constitutes acceptance.

## Contact

Questions about this privacy policy:

**S4Y Solutions** — [sergey@s4y.solutions](mailto:sergey@s4y.solutions)
