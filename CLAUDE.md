# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a personal blog built with AstroPaper, an Astro-based blog theme. The site is deployed at https://obedmacallums.com/ and focuses on web development, programming, and technology content.

## Architecture

### Framework & Technologies
- **Main Framework**: Astro v4.16.3 with experimental content layer enabled
- **Styling**: TailwindCSS with custom base styles disabled
- **Components**: React (for interactive components like Search)
- **Content Management**: Astro Content Collections with markdown files
- **Build Tools**: TypeScript, ESLint, Prettier
- **Deployment**: Static site generation

### Key Directories
- `src/content/blog/` - All blog posts as markdown files
- `src/components/` - Reusable Astro and React components
- `src/layouts/` - Page layout templates
- `src/pages/` - Route definitions and special pages (RSS, sitemap, OG images)
- `src/utils/` - Utility functions for post processing, OG image generation
- `public/` - Static assets (images, favicon, etc.)

### Content Architecture
- Blog posts use Astro Content Collections with strict schema validation
- Posts support frontmatter with: title, description, pubDatetime, tags, featured status, draft status
- Dynamic OG image generation for blog posts using Satori
- RSS feed and sitemap automatically generated
- Fuzzy search implemented with Fuse.js

### Theme Configuration
- Site configuration in `src/config.ts` (SITE, LOCALE, LOGO_IMAGE, SOCIALS)
- Content schema defined in `src/content/config.ts`
- Astro configuration in `astro.config.ts` with integrations and markdown processing

## Development Commands

All commands run from project root:

### Core Development
- `npm run dev` - Start development server at localhost:4321
- `npm run build` - Build production site to ./dist/
- `npm run preview` - Preview production build locally

### Code Quality
- `npm run lint` - Run ESLint
- `npm run format:check` - Check code formatting with Prettier
- `npm run format` - Format code with Prettier
- `npm run sync` - Generate TypeScript types for Astro modules

### Docker (Alternative)
- `docker build -t astropaper .` - Build Docker image
- `docker run -p 4321:80 astropaper` - Run in Docker container
- `docker compose up -d` - Run with docker-compose

## Content Management

### Adding Blog Posts
1. Create new `.md` file in `src/content/blog/`
2. Include required frontmatter: title, description, pubDatetime, tags
3. Optional: featured (boolean), draft (boolean), ogImage, canonicalURL
4. Content supports standard markdown with remark plugins (TOC, collapse sections)

### Post Schema Requirements
- `pubDatetime` must be a valid Date
- `tags` defaults to ["others"] if not specified
- `ogImage` must be at least 1200x630 pixels if provided
- `author` defaults to site author from config

### Featured Posts
Set `featured: true` in frontmatter to display on homepage featured section.

## Styling & Customization

### Theme System
- Light/dark mode supported (configurable in src/config.ts)
- Uses Tailwind with custom color schemes
- Syntax highlighting with Shiki (light: min-light, dark: night-owl)

### Component Architecture
- Astro components for static content (.astro files)
- React components for interactive features (.tsx files)
- Shared utilities in src/utils/ for common operations

### Social Integration
Configure social links in `src/config.ts` SOCIALS array. Set `active: true` to display.

## Build & Deployment Notes

- Site generates static files suitable for any static host
- Uses Astro's experimental content layer for improved content handling
- Optimized for performance with Vite build optimizations
- Excludes @resvg/resvg-js from Vite optimization due to binary dependencies