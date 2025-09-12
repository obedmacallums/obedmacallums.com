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
2. Include required frontmatter with ALL attributes from the template (see adding-new-post.md and las-format.md examples):
   - `author`: Author name (e.g., "Obed Macallums")
   - `pubDatetime`: Publication date in ISO 8601 format (e.g., "2025-01-10T10:21:00Z")
   - `modDatetime`: Last modification date in ISO 8601 format (optional, for updated posts)
   - `title`: Post title
   - `slug`: URL slug (optional, auto-generated from title if not provided)
   - `featured`: Boolean for featured status (true/false)
   - `draft`: Boolean for draft status (true/false)
   - `tags`: Array of relevant tags
   - `description`: Detailed description of the post content
3. Optional additional attributes: ogImage, canonicalURL
4. **IMPORTANT**: Frontmatter attributes must be in the exact order shown in `src/content/blog/adding-new-post.md`
5. Content supports standard markdown with remark plugins (TOC, collapse sections)

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

## Current Published Posts

The following blog posts are currently published on the site (not in draft status):

• Getting Started with PyTorch
• Advanced Git Commands: Power Tips for Developers
• Advanced Pandas Techniques for Data Analysis
• Advanced Techniques in PyTorch
• Understanding Docker Basics: A Comprehensive Guide
• Essential Git Commands with Examples
• Gmail Templates from Drive - Chrome Extension
• Privacy Policy - Gmail Templates from Drive
• Introduction to NumPy Library
• Introduction to FastAPI - Modern Python Web Framework
• Introduction to Kubernetes: Orchestrating Containers at Scale
• Introduction to Pandas
• Introduction to Streamlit
• Introduction to .LAS and .LAZ Point Cloud Formats
• Matplotlib Tutorial: A Comprehensive Guide
• Introduction to OpenCV: A Powerful Library for Computer Vision
• Understanding the .PLY Point Cloud Format
• Introduction to Django: Python's Powerful Web Framework
• Introduction to Seaborn: Statistical Data Visualization in Python

## Blog Post Creation Guidelines

### Example Templates
Use these files as templates when creating new blog posts:
- `src/content/blog/adding-new-post.md` - Shows proper frontmatter structure and markdown formatting, it also shows all the attributes of a post.
- `src/content/blog/las-format.md` - Example of a well-structured technical tutorial post

### Table of Contents
Simply add `## Table of Contents` anywhere in your markdown content to automatically generate a table of contents for that post.

### Date Requirements
When creating new blog posts, always use the current date in ISO 8601 format for the `pubDatetime` field in the frontmatter.

### Post Creation Workflow

**IMPORTANT**: When asked to create a new blog post, ALWAYS read the files `src/content/blog/adding-new-post.md` and `src/content/blog/las-format.md` first to understand the proper structure, frontmatter format, and content organization before creating any new post.

After creating a new blog post, always ask if the "Current Published Posts" list in this CLAUDE.md file should be updated to include the new post title.