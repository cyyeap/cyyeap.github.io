# Hugo Site

This is a Hugo static site using the PaperMod theme.

## Prerequisites

- [Hugo](https://gohugo.io/installation/) (Extended version)

## Installation

1. Clone the repository:
```bash
git clone <repository-url>
cd cyyeap.github.io
```

2. Initialize and update the theme submodule:
```bash
git submodule update --init --recursive
```

## Running the Development Server

To run the site locally with live reload:

```bash
hugo server -D
```

The site will be available at `http://localhost:1313/`

The `-D` flag includes draft posts. To run without drafts:

```bash
hugo server
```

## Building for Production

To build the static site:

```bash
hugo
```

The generated files will be in the `public/` directory.

## Creating New Content

To create a new post:

```bash
hugo new posts/my-post-name.md
```

## Project Structure

- `content/` - Markdown content files
- `themes/` - Hugo themes (PaperMod)
- `static/` - Static assets
- `layouts/` - Custom layout overrides
- `hugo.yaml` - Site configuration
