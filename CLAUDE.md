# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

A collection of self-contained browser tools deployed to GitHub Pages at https://jdcockrill.github.io/tools/. No build step, no framework, no package manager — plain HTML/CSS/JS only.

## Development

Open any `.html` file directly in a browser. There are no build commands, test suites, or dev servers.

## Architecture

Each tool is a **single self-contained `.html` file** — all styles, markup, and JS in one file. `index.html` is the landing page listing all tools.

**CSS conventions:**
- CSS custom properties (`--bg`, `--fg`, `--muted`, `--border`, `--surface`, `--link`) defined in `:root`
- Dark mode via `@media (prefers-color-scheme: dark)` immediately after `:root`

**When adding a new tool:** create a new self-contained `.html` file and add a link to it in `index.html`.
