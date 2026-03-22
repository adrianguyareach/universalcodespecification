# Universal Code Specification

This repository contains language-agnostic [NLSpecs](#terminology) that define how software is written and how it is built. The specifications enforce SOLID principles, clean architecture, and testable design across any language, framework, or runtime.

The code specification governs implementation quality. The SDLC specification governs the development lifecycle — from planning through testing and release. Together they form a complete standard for producing robust software, whether by human engineers or autonomous AI pipelines.

## Specs

- [Universal Code Specification](./UNIVERSAL_CODE_SPECIFICATION.md)
- [SDLC Specification (Dark Factory)](./SDLC.md)

## Companion Projects

The SDLC specification is designed to integrate with [Attractor](https://github.com/strongdm/attractor), StrongDM's pipeline runner for AI-driven software factories. Attractor orchestrates multi-stage workflows; this repository defines the quality standards and lifecycle phases those workflows follow.

## Terminology

- **NLSpec** (Natural Language Spec): a human-readable specification intended to be directly usable by coding agents to implement and validate behavior.
- **Dark Factory**: an autonomous software development pipeline where AI agents plan, implement, test, and release software without human intervention. Quality is enforced through automated gates, not manual review.
- **Greenfield**: a new application built from scratch with no existing codebase.
- **Brownfield**: a modification to an application with an existing codebase, tests, and conventions.
