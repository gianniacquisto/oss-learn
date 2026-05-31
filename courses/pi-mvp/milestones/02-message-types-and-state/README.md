# Milestone 2: Message Types & Agent State

## What You'll Build

A type-safe message system and an `Agent` class that manages conversation context. The agent can store messages, track which tools are available, and report its current state (idle, streaming, etc.). No loop yet — just the data model and state management.

## Why This Matters

Before you can build a loop, you need to understand what the loop operates on: **messages** and **state**. In pi.dev, messages are a union of LLM-compatible types (`user`, `assistant`, `toolResult`) plus custom application-specific message types. The Agent class owns this state and provides controlled access through getters/setters that copy arrays before exposing them (preventing accidental mutation).

## Prerequisites for This Milestone

- Complete Milestone 1
- Comfortable with TypeScript interfaces, unions, and generics
