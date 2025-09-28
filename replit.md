# Overview

TailBliss is a modern Hugo static site generator theme designed for building fast, responsive websites and blogs. The project combines Hugo's powerful static site generation capabilities with modern frontend technologies including TailwindCSS 4.x for styling, Alpine.js for interactive components, and Vite for efficient asset bundling and optimization. The theme focuses on performance, developer experience, and provides both light and dark mode support out of the box.

# User Preferences

Preferred communication style: Simple, everyday language.

# System Architecture

## Static Site Generation
The project uses Hugo as the core static site generator, leveraging its powerful templating system and content management capabilities. Hugo processes Markdown content files from the `content/` directory and generates static HTML pages using Go templates. The architecture supports multiple content types including blog posts, pages, and categorized content with tag-based organization.

## Frontend Build System
The build system integrates Vite 7.x as the primary bundler for CSS and JavaScript assets. This choice provides fast development builds, hot module replacement, and optimized production bundles. PostCSS is configured to process TailwindCSS 4.x, enabling utility-first styling with automatic purging of unused CSS for minimal bundle sizes.

## Styling Architecture
TailwindCSS 4.x serves as the primary CSS framework, providing utility-first styling capabilities. The configuration includes the official Typography plugin (@tailwindcss/typography) for enhanced prose styling, particularly beneficial for blog content rendered from Markdown. The system supports both light and dark theme modes with automatic theme detection and manual toggle functionality.

## Interactive Components
Alpine.js 3.x handles client-side interactivity without requiring a complex JavaScript framework. This lightweight approach enables features like theme switching, mobile navigation, and other UI interactions while maintaining excellent performance characteristics.

## Development Workflow
The development environment uses concurrent processes to watch for changes in both CSS (via Vite) and content (via Hugo). This setup enables real-time preview during development with automatic rebuilding of assets and content. The build system includes separate development and production configurations with optimizations for each environment.

## Content Management
Content is organized using Hugo's content management system with support for front matter metadata, featured images, categories, and tags. The structure supports multilingual content and includes example content for demonstration purposes. The system processes Markdown files and applies appropriate templates based on content type and layout specifications.

# External Dependencies

## Build Tools and Frameworks
- **Hugo**: Static site generator (minimum version 0.148.2)
- **Node.js**: JavaScript runtime for build tools and package management
- **Vite**: Frontend build tool and development server
- **PostCSS**: CSS processing with autoprefixer support

## CSS and UI Libraries
- **TailwindCSS**: Utility-first CSS framework (version 4.x)
- **@tailwindcss/typography**: Official typography plugin for prose styling
- **Alpine.js**: Lightweight JavaScript framework for UI interactions

## Development Dependencies
- **Concurrently**: Tool for running multiple npm scripts simultaneously
- **Autoprefixer**: PostCSS plugin for vendor prefix management

## Deployment Platforms
- **Cloudflare Pages**: Recommended deployment platform with specific build configurations
- **Netlify**: Alternative deployment option with Hugo support

## External Services
- **FormSubmit**: Third-party form handling service for contact forms
- **CDN Services**: Content delivery for external assets and libraries
- **Google AdSense**: Advertisement integration (configured via ads.txt)

The architecture prioritizes performance, developer experience, and maintainability while providing a solid foundation for building modern static websites and blogs.