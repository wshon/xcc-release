## Release v0.3.0

### New Features

#### Plugin System
- Integrated Python runtime with embedded support and virtual environment management
- Added plugin marketplace for discovering and installing plugins from GitHub
- Implemented async plugin discovery and dependency management
- Added plugin enable/disable configuration system
- Added environment variable configuration support for plugins
- Implemented server provider plugin system with BandwagonHost example
- Added plugin password storage management

#### Terminal Enhancements
- Implemented comprehensive terminal completion system with multiple providers
- Added scrollbar support with scroll metrics tracking
- Added alternate screen mode detection for completions
- Aligned terminal content to bottom of canvas
- Improved terminal selection modes

#### UI Improvements
- Migrated plugin marketplace from dialog to dedicated page
- Added card-based layout for plugin settings
- Implemented collapsible section headers
- Added pin icon and tab pinning functionality
- Improved connections page layout with absolute positioning
- Sort tabs with pinned items displayed first

#### Internationalization
- Added i18n support for plugin marketplace and settings pages
- Made locale lists dynamic from YAML files
- Dynamically display application version in about dialog

#### Logging System
- Implemented dual-output logging system with file rotation
- Optimized log formatting for better console readability
- Adjusted log levels to reduce spam

### Bug Fixes
- Suppress console windows for Python subprocess calls on Windows (CREATE_NO_WINDOW flag)
- Fixed window closure handling in async update callbacks
- Prevented duplicate marketplace windows
- Disabled GPUI logging to eliminate window handle errors
- Improved error handling in plugin marketplace installation

### Configuration & Packaging
- Added cargo-packager configuration for NSIS installer
- Enabled plugin feature by default in build configuration
- Simplified cargo-packager resource configuration
- Optimized binary size with aggressive compiler settings
- Added Windows packaging scripts

### Documentation
- Reorganized documentation structure with category-based hierarchy
- Consolidated plugin system documentation
- Added comprehensive plugin development guides
- Documented embedded Python environment and deployment
- Established documentation standards

### Architecture & Refactoring
- Restructured plugin system architecture with dedicated runtime crate
- Migrated to global plugin manager pattern
- Extracted common plugin action logic
- Centralized directory management with dedicated dirs module
- Removed pyo3 dependency and refactored build system
- Integrated font-kit for cross-platform font management

### Performance Optimizations
- Optimized tokio features across crates
- Removed unused dependencies
- Added server discovery caching
- Moved plugin initialization to blocking task to prevent UI thread blocking

### Security
- Migrated Bandwagon to environment-based credentials
- Improved password handling in plugin system
