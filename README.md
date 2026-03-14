# Toona Documentation

This is the documentation site for Toona, built with [Hugo](https://gohugo.io/) and the [Hugo Book](https://github.com/alex-shpak/hugo-book) theme.

## Development

### Prerequisites

- Hugo Extended v0.157.0 or later

### Running Locally

```bash
# Start the development server
hugo server --buildDrafts

# Build for production
hugo
```

The site will be available at `http://localhost:1313/`

## Structure

```
content/
├── _index.md              # Homepage
└── docs/                  # Documentation
    ├── getting-started/   # Installation guides
    ├── user-guide/        # User documentation
    ├── developer-guide/   # Developer documentation
    └── api-reference/     # API documentation
```

## Contributing

1. Fork the repository
2. Create your feature branch
3. Make your changes
4. Submit a pull request

## License

Same as Toona project - MIT License
