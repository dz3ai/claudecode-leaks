# Part 5: Supply Chain Security Lessons

## 5.1 Source Map Risks Explained

### 5.1.1 What is a Source Map

**Source Map Definition:**

A Source Map is a file format used to map compiled/minified code back to original source code. It allows developers to debug original source code in production environments instead of dealing with hard-to-read transformed code.

**Technical Specification:**
- Standard: [Source Map Revision 3 Proposal](https://sourcemaps.info/spec.html)
- File extension: `.map`
- Format: JSON
- Key fields: `version`, `sources`, `mappings`, `names`

**Source Map Structure Example:**

```json
{
  "version": 3,
  "sources": [
    "file:///path/to/src/main.tsx",
    "file:///path/to/src/components/App.tsx"
  ],
  "sourcesContent": [
    "import React from 'react'\nexport const App = () => {...}",
    "import { Box } from 'ink'\n..."
  ],
  "mappings": "AAAA,SAASA,IAAI,GAAG...",
  "names": ["React", "App", "Box"]
}
```

**Key Findings:**
- `sourcesContent` field contains **complete original source code**
- If `sourcesContent` exists, source map file is **self-contained**
- No additional source code file access required

### 5.1.2 Leak Mechanism Analysis

**Specific Path of Claude Code Leak:**

```
1. Anthropic Build Process
   ├── TypeScript source code compilation
   ├── Bun bundling generates cli.js
   ├── Generate source map (cli.js.map)
   └── Publish to npm (@anthropic-ai/claude-code v2.1.88)

2. Source Map Content
   ├── Contains sourcesContent field
   ├── Complete TypeScript source embedded
   └── Paths pointing to R2 storage bucket

3. Leak Path
   ├── User installs npm package
   ├── Downloads node_modules/@anthropic-ai/claude-code/
   ├── Discovers cli.js.map file (59.8MB)
   ├── Parses source map JSON
   └── Extracts source code from sourcesContent
```

**Key Factors of the Leak:**

1. **Source Map Included `sourcesContent`**
   - This is the **root cause** of the leak
   - Without `sourcesContent`, only file paths are visible
   - With `sourcesContent`, complete source code can be extracted

2. **Source Map File Too Large (59.8MB)**
   - Normal source maps are usually smaller (a few MB)
   - 60MB size indicates it contains a lot of source code
   - Easy for security researchers to discover

3. **Public npm Package**
   - Anyone can `npm install`
   - No authentication required
   - Globally accessible

### 5.1.3 Why Include sourcesContent

**Development Convenience vs Security Trade-off:**

| Aspect | Include sourcesContent | Exclude sourcesContent |
|--------|----------------------|----------------------|
| **Debugging Experience** | ✅ Can view source directly in browser/tools | ⚠️ Need source server or local files |
| **Error Stacks** | ✅ Complete original files and line numbers | ⚠️ Only see transformed code |
| **Deployment Complexity** | ✅ No additional source access config needed | ❌ Need to set up source server |
| **Security** | ❌ Source code directly exposed | ✅ Source code stays in controlled environment |

**Common Scenarios:**

1. **Frontend Web Applications** (reasonable)
   - Browser needs source map for debugging
   - Code will be on client side anyway
   - sourcesContent convenient for development

2. **Open Source Projects** (acceptable)
   - Source code is public anyway
   - Source map adds no additional risk

3. **Commercial CLI Tools** (dangerous!) ← Claude Code's case
   - Source code is proprietary intellectual property
   - Should NOT include sourcesContent
   - Should use internal source map server

### 5.1.4 Source Map Security Risk Matrix

**Risk Level Assessment:**

| Deployment Environment | Include sourcesContent | No sourcesContent |
|----------------------|----------------------|-------------------|
| **Public npm (closed source)** | 🔴 Extreme Risk - Full source leak | 🟡 Medium Risk - Can infer structure |
| **Public npm (open source)** | 🟢 Low Risk - Code already public | 🟢 Low Risk - Code already public |
| **Enterprise internal npm** | 🟡 Medium Risk - Internal exposure | 🟢 Low Risk - Controlled access |
| **Frontend App (CDN)** | 🟡 Medium Risk - Client visible | 🟢 Low Risk - Minified code |
| **Mobile App** | 🔴 High Risk - Reverse engineering | 🟡 Medium Risk - Still decompilable |

**Claude Code's Risk Position:**
- 🔴 **Extreme Risk**
- Closed-source commercial software
- Public npm package
- Contains complete sourcesContent
- 512,000+ lines of code leaked

### 5.1.5 Detecting Source Map Leaks

**Automatic Detection Script:**

```bash
#!/bin/bash
# check-source-maps.sh - Detect source maps in npm packages

PACKAGE_DIR="node_modules"
THRESHOLD_MB=5

echo "Scanning for source maps in $PACKAGE_DIR..."
echo "Threshold: ${THRESHOLD_MB}MB"
echo ""

find "$PACKAGE_DIR" -name "*.map" -type f | while read -r mapfile; do
  size=$(du -h "$mapfile" | cut -f1)
  size_bytes=$(du -b "$mapfile" | cut -f1)
  size_mb=$((size_bytes / 1024 / 1024))

  if [ $size_mb -gt $THRESHOLD_MB ]; then
    echo "⚠️  LARGE SOURCE MAP: $mapfile (${size})"
    echo "   Checking for sourcesContent..."
    if grep -q '"sourcesContent"' "$mapfile"; then
      echo "   🔴 CONTAINS sourcesContent - POTENTIAL LEAK"
      # Extract source file count
      sources=$(jq -r '.sources | length' "$mapfile" 2>/dev/null || echo "?")
      echo "   Sources: $sources files"
    fi
    echo ""
  fi
done
```

**npm audit Integration:**

```javascript
// audit-source-maps.js
const fs = require('fs')
const path = require('path')

function auditSourceMaps(packageDir = 'node_modules') {
  const results = []

  function scanDir(dir) {
    const files = fs.readdirSync(dir)

    files.forEach(file => {
      const fullPath = path.join(dir, file)
      const stat = fs.statSync(fullPath)

      if (stat.isDirectory()) {
        scanDir(fullPath)
      } else if (file.endsWith('.map')) {
        const size = stat.size
        const sizeMB = (size / 1024 / 1024).toFixed(2)

        const content = fs.readFileSync(fullPath, 'utf-8')
        const hasSourcesContent = content.includes('"sourcesContent"')

        if (sizeMB > 5 || hasSourcesContent) {
          results.push({
            file: fullPath,
            sizeMB,
            hasSourcesContent,
            risk: hasSourcesContent ? 'HIGH' : 'MEDIUM'
          })
        }
      }
    })
  }

  scanDir(packageDir)
  return results
}

const findings = auditSourceMaps()

console.log('Source Map Audit Results:')
console.log('='.repeat(60))

findings.forEach(finding => {
  console.log(`\n${finding.file}`)
  console.log(`  Size: ${finding.sizeMB}MB`)
  console.log(`  Has sourcesContent: ${finding.hasSourcesContent}`)
  console.log(`  Risk Level: ${finding.risk}`)
})

if (findings.some(f => f.risk === 'HIGH')) {
  console.log('\n🔴 HIGH RISK: Source maps may contain source code!')
  process.exit(1)
}
```

### 5.1.6 Source Map Best Practices

**✅ Recommended Practices:**

1. **Never Include sourcesContent in Production Builds**
   ```javascript
   // webpack.config.js
   module.exports = {
     devtool: process.env.NODE_ENV === 'production'
       ? 'source-map'  // Does not include sourcesContent
       : 'eval-source-map'  // Can include in development
   }
   ```

2. **Separate Source Map Storage**
   ```bash
   # Upload source maps to internal service during build
   npm run build
   npm run upload-sourcemaps  # Upload to Sentry/internal service
   rm dist/*.map  # Delete local source maps
   ```

3. **Use Internal Source Map Server**
   ```javascript
   // Configure source map URL
   const sourceMapUrl = 'https://sourcemaps.internal.com/project/hash/'
   ```

4. **Check Source Maps in CI/CD**
   ```yaml
   # .github/workflows/build.yml
   - name: Check for source maps
     run: |
       npm run audit-sourcemaps
   ```

**❌ Practices to Avoid:**

1. **Include sourcesContent in public packages**
   ```javascript
   // ❌ Dangerous configuration
   devtool: 'eval-cheap-module-source-map'  // Inline sourcesContent
   ```

2. **Publish source maps to npm**
   ```json
   // package.json
   {
     "files": [  // ❌ Include .map files
       "dist/",
       "dist/**/*.map"
     ]
   }
   ```

3. **Skip Validating Source Map Content**
   ```bash
   # ❌ Missing validation step
   npm publish  # Publish directly without checking source maps
   ```

---

## 5.2 Build Configuration Best Practices

### 5.2.1 Bun Build Security Configuration

**Secure Production Build Configuration:**

```typescript
// bun.config.ts
import { defineConfig } from 'bun'

export default defineConfig({
  build: {
    // Entry points
    entrypoints: ['./src/main.tsx'],

    // Output configuration
    outdir: './dist',
    target: 'bun',
    format: 'esm',

    // Source map configuration (critical!)
    sourcemap: process.env.NODE_ENV === 'development' ? 'external' : false,

    // Or use internal source map server
    // sourcemap: 'inline',  // ❌ Disable in production
    // sourcemap: 'external',  // ✅ Generate separate .map file

    // Code minification
    minify: true,

    // Exclude sensitive files
    external: [
      '*.test.ts',
      '*.spec.ts',
      '*.mock.ts',
      'secrets/',
      'credentials/'
    ],

    // Environment variable injection (allowed prefixes only)
    define: {
      'process.env.NODE_ENV': JSON.stringify(process.env.NODE_ENV),
      // ❌ Don't inject all environment variables
      // ...process.env  // Dangerous!
    },

    // Dead code elimination
    // treeShaking: true,  // Bun enables by default

    // Build validation hooks
    onSuccess: [
      'echo "Build complete"',
      'node scripts/validate-build.js',  // Custom validation
      'node scripts/check-sourcemaps.js'  // Source map check
    ]
  }
})
```

### 5.2.2 Webpack/Vite Configuration

**Secure Webpack Configuration:**

```javascript
// webpack.config.js
module.exports = (env, argv) => {
  const isProduction = argv.mode === 'production'

  return {
    // ... other configuration

    // Source map configuration
    devtool: isProduction
      ? 'hidden-source-map'  // Generate but don't reference
      // : 'source-map'  // Or completely disable
      : 'eval-cheap-module-source-map',  // Development

    // Production environment plugins
    plugins: isProduction ? [
      // Remove source map references
      new webpack.SourceMapDevToolPlugin({
        filename: '[file].map',
        noSources: true,  // ❌ Key: Don't include sourcesContent
        // Or completely remove source map URL
      }),

      // Environment variable validation
      new webpack.EnvironmentPlugin({
        NODE_ENV: 'production',
        // Only include required environment variables
      }),

      // Secret detection
      new SecretsPlugin({
        patterns: [
          /API_KEY/,
          /SECRET/,
          /PASSWORD/,
          /TOKEN/
        ]
      })
    ] : [],

    // Output configuration
    output: {
      // Don't include source map URL in production
      sourceMapFilename: isProduction ? false : '[file].map'
    }
  }
}
```

**Vite Configuration:**

```javascript
// vite.config.js
export default defineConfig({
  build: {
    // Source map configuration
    sourcemap: process.env.NODE_ENV === 'development'
      ? 'true'
      : false,  // Disable in production

    // Or use internal server
    // sourcemap: 'hidden',  // Generate but don't reference

    // Output separation
    rollupOptions: {
      output: {
        // Source map as separate file
        sourcemapPathTransform: (relativeSourcePath, sourcemapPath) => {
          // Redirect to internal server
          return `https://sourcemaps.internal.com/${sourcemapPath}`
        }
      }
    },

    // Build size warnings
    chunkSizeWarningLimit: 1000,  // 1MB
  }
})
```

### 5.2.3 TypeScript Compilation Configuration

**Secure tsconfig.json:**

```json
{
  "compilerOptions": {
    // Source mapping configuration
    "sourceMap": true,  // Generate .map files
    "inlineSourceMap": false,  // ❌ Disable inline source map
    "inlineSources": false,  // ❌ Disable inline source code

    // Declaration maps
    "declarationMap": false,  // Disable in production

    // Remove comments
    "removeComments": true,

    // Compilation target
    "target": "ES2020",
    "module": "ESNext",

    // Strict mode
    "strict": true,

    // Other security options
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  },
  "exclude": [
    "node_modules",
    "dist",
    "**/*.test.ts",
    "**/*.spec.ts",
    "secrets/",
    "credentials/"
  ]
}
```

### 5.2.4 NPM Package Publishing Configuration

**package.json Security Configuration:**

```json
{
  "name": "your-package",
  "version": "1.0.0",
  "private": false,  // Set to true for private packages

  // Control published files
  "files": [
    "dist/",
    "dist/**/*.js",
    "dist/**/*.d.ts",
    "README.md",
    "LICENSE"
    // ❌ Don't include .map files
    // "dist/**/*.map"
  ],

  // Scripts include validation steps
  "scripts": {
    "build": "bun build src/index.ts --outdir dist",
    "prebuild": "node scripts/check-secrets.js",
    "postbuild": "node scripts/validate-build.js",
    "prepublishOnly": "npm run build && npm run validate-package",
    "validate-package": "npm pack --dry-run && tar -tzf *.tgz | grep -E '\\.map$' && exit 1 || true"
  },

  // Dev dependencies (not published)
  "devDependencies": {
    "typescript": "^5.0.0",
    "@types/node": "^20.0.0"
  }
}
```

**.npmignore File:**

```bash
# Source code
src/
*.ts
*.tsx

# Test files
test/
tests/
**/*.test.ts
**/*.spec.ts

# Source maps (critical!)
*.map
**/*.map

# Development configuration
tsconfig.json
webpack.config.js
vite.config.js

# CI/CD
.github/
.gitlab-ci.yml

# Documentation and examples
docs/
examples/

# Secrets and credentials
.env
.env.local
secrets/
credentials/

# IDE
.vscode/
.idea/
*.swp
*.swo
```

### 5.2.5 Build Validation Scripts

**build-validator.js:**

```javascript
#!/usr/bin/env node
const fs = require('fs')
const path = require('path')
const { execSync } = require('child_process')

console.log('🔍 Validating build output...\n')

const distDir = 'dist'
const errors = []
const warnings = []

// Check 1: Source map file sizes
console.log('Checking source map sizes...')
const mapFiles = execSync(`find ${distDir} -name "*.map"`)
  .toString()
  .trim()
  .split('\n')
  .filter(Boolean)

mapFiles.forEach(file => {
  const stats = fs.statSync(file)
  const sizeMB = (stats.size / 1024 / 1024).toFixed(2)

  if (stats.size > 10 * 1024 * 1024) {  // 10MB
    errors.push(`Large source map: ${file} (${sizeMB}MB)`)
  } else if (stats.size > 5 * 1024 * 1024) {  // 5MB
    warnings.push(`Big source map: ${file} (${sizeMB}MB)`)
  }
})

// Check 2: Source map content
console.log('Checking source map content...')
mapFiles.forEach(file => {
  const content = fs.readFileSync(file, 'utf-8')

  if (content.includes('"sourcesContent"')) {
    // Check if empty array
    try {
      const map = JSON.parse(content)
      if (map.sourcesContent && map.sourcesContent.length > 0) {
        // Check if sourcesContent is null (no source code)
        const hasActualSource = map.sourcesContent.some(
          source => source !== null && source.length > 100
        )

        if (hasActualSource) {
          errors.push(`Source map contains sourcesContent: ${file}`)
        }
      }
    } catch (e) {
      warnings.push(`Failed to parse source map: ${file}`)
    }
  }
})

// Check 3: Unexpected files
console.log('Checking for unexpected files...')
const unexpectedPatterns = [
  /\.test\.js$/,
  /\.spec\.js$/,
  /\.mock\.js$/,
  /secrets/,
  /credentials/,
  /\.env$/
]

const allFiles = execSync(`find ${distDir} -type f`)
  .toString()
  .trim()
  .split('\n')
  .filter(Boolean)

allFiles.forEach(file => {
  unexpectedPatterns.forEach(pattern => {
    if (pattern.test(file)) {
      errors.push(`Unexpected file in dist: ${file}`)
    }
  })
})

// Check 4: Secrets in code
console.log('Checking for secrets...')
const secretPatterns = [
  /API_KEY\s*[:=]\s*['"]([a-zA-Z0-9-]{20,})['"]/,
  /SECRET\s*[:=]\s*['"]([a-zA-Z0-9-]{20,})['"]/,
  /PASSWORD\s*[:=]\s*['"]([^\s'"]{8,})['"]/,
  /TOKEN\s*[:=]\s*['"]([a-zA-Z0-9-._]{20,})['"]/
]

const jsFiles = allFiles.filter(f => f.endsWith('.js'))
jsFiles.forEach(file => {
  const content = fs.readFileSync(file, 'utf-8')
  secretPatterns.forEach(pattern => {
    if (pattern.test(content)) {
      errors.push(`Possible secret in ${file}`)
    }
  })
})

// Output results
console.log('\n' + '='.repeat(60))
console.log('Validation Results:')
console.log('='.repeat(60))

if (errors.length > 0) {
  console.log('\n❌ ERRORS:')
  errors.forEach(err => console.log(`  - ${err}`))
}

if (warnings.length > 0) {
  console.log('\n⚠️  WARNINGS:')
  warnings.forEach(warn => console.log(`  - ${warn}`))
}

if (errors.length === 0 && warnings.length === 0) {
  console.log('\n✅ All checks passed!')
  process.exit(0)
} else if (errors.length > 0) {
  console.log('\n❌ Validation failed!')
  process.exit(1)
} else {
  console.log('\n⚠️  Validation passed with warnings')
  process.exit(0)
}
```

---

## 5.3 CI/CD Security Checkpoints

### 5.3.1 GitHub Actions Workflow

**Secure CI Configuration:**

```yaml
# .github/workflows/build-and-test.yml
name: Build and Security Check

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  release:
    types: [created]

jobs:
  security-checks:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for vulnerability scanning

      # Check 1: Secret scanning
      - name: Scan for secrets
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD

      # Check 2: Source map validation
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Validate source maps
        run: |
          node scripts/validate-build.js
        continue-on-error: false

      # Check 3: Package content audit
      - name: Pack for dry run
        run: npm pack

      - name: Check package contents
        run: |
          echo "Checking package contents..."
          tar -tzf *.tgz | tee package-contents.txt

          # Check for .map files
          if grep -E '\.map$' package-contents.txt; then
            echo "❌ ERROR: Source maps found in package!"
            exit 1
          fi

          # Check for test files
          if grep -E '\.(test|spec)\.' package-contents.txt; then
            echo "❌ ERROR: Test files found in package!"
            exit 1
          fi

          # Check for source files (should only have compiled code)
          if grep -E '\.(ts|tsx)$' package-contents.txt; then
            echo "⚠️  WARNING: TypeScript files found in package"
          fi

          echo "✅ Package content check passed"

      # Check 4: Dependency vulnerability scan
      - name: Run npm audit
        run: npm audit --audit-level=moderate
        continue-on-error: true

      - name: Snyk test
        uses: snyk/actions/node@master
        continue-on-error: true
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

      # Check 5: License compliance
      - name: Check licenses
        run: npx license-checker --production --failOn "GPL"

      # Check 6: CodeQL analysis
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v2
        with:
          languages: javascript

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v2

  publish:
    needs: security-checks
    runs-on: ubuntu-latest
    # Only run on releases
    if: github.event_name == 'release'
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://registry.npmjs.org'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Validate build
        run: node scripts/validate-build.js

      - name: Publish to npm
        run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}

      # Post-publish validation
      - name: Verify published package
        run: |
          PACKAGE_NAME=$(node -p "require('./package.json').name")
          PACKAGE_VERSION=$(node -p "require('./package.json').version")

          echo "Installing published package..."
          npm install ${PACKAGE_NAME}@${PACKAGE_VERSION} --temp-dir=/tmp/verify

          echo "Checking for source maps in published package..."
          if find /tmp/verify -name "*.map" | grep -q .; then
            echo "❌ ERROR: Source maps found in published package!"
            exit 1
          fi

          echo "✅ Package verification passed"
```

### 5.3.2 Pre-commit Hook

**Pre-commit Hook Using Husky:**

```javascript
// .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

echo "Running pre-commit checks..."

# Check 1: Source map check
echo "Checking for source maps in package..."
if git diff --cached --name-only | grep -E '\.map$'; then
  echo "❌ ERROR: Attempting to commit source map files!"
  echo "Remove .map files from the commit:"
  git diff --cached --name-only | grep -E '\.map$' | xargs git reset HEAD
  exit 1
fi

# Check 2: Secret scanning
echo "Scanning for secrets..."
if git diff --cached | grep -E '(API_KEY|SECRET|PASSWORD|TOKEN)\s*[:=]\s*['\''"]' | grep -v 'example\|test'; then
  echo "❌ ERROR: Possible secret detected in staged changes!"
  echo "Review the changes and remove any secrets."
  exit 1
fi

# Check 3: Run tests
echo "Running tests..."
npm test -- --passWithNoTests

# Check 4: Lint
echo "Running linter..."
npm run lint

# Check 5: TypeScript check
echo "Running TypeScript check..."
npm run type-check

echo "✅ Pre-commit checks passed!"
```

### 5.3.3 Pre-publish Hook

**PrepublishOnly Hook in package.json:**

```json
{
  "scripts": {
    "prepublishOnly": "npm run validate:publish",
    "validate:publish": "npm-run-all validate:build validate:security validate:package",
    "validate:build": "node scripts/validate-build.js",
    "validate:security": "npm run security:check",
    "validate:package": "node scripts/validate-package.js",
    "security:check": "npm audit && npm run license-check"
  }
}
```

**validate-package.js:**

```javascript
#!/usr/bin/env node
const fs = require('fs')
const { execSync } = require('child_process')

console.log('🔍 Validating package for publish...\n')

const packageJson = JSON.parse(fs.readFileSync('package.json', 'utf-8'))
const errors = []

// Check 1: Is private package
if (packageJson.private) {
  console.log('⚠️  This is a private package. Skipping publish validation.')
  process.exit(0)
}

// Check 2: package.json files field
if (packageJson.files) {
  const hasMapFiles = packageJson.files.some(f => f.includes('.map'))
  if (hasMapFiles) {
    errors.push('package.json files field includes .map files')
  }
}

// Check 3: .npmignore configuration
if (fs.existsSync('.npmignore')) {
  const npmignore = fs.readFileSync('.npmignore', 'utf-8')
  if (!npmignore.includes('.map') && !npmignore.includes('*.map')) {
    errors.push('.npmignore does not exclude .map files')
  }
}

// Check 4: Package validation
console.log('Creating dry-run package...')
try {
  execSync('npm pack --dry-run', { stdio: 'inherit' })

  const tgzFiles = execSync('ls *.tgz').toString().trim().split('\n')
  const tgzFile = tgzFiles[0]

  console.log(`\nChecking contents of ${tgzFile}...`)
  const contents = execSync(`tar -tzf "${tgzFile}"`).toString()

  // Check for source maps
  if (/\.(js)?\.map$/.test(contents)) {
    errors.push('Source maps found in package archive')
  }

  // Check for test files
  if (/\.(test|spec)\.(js|ts)$/.test(contents)) {
    errors.push('Test files found in package archive')
  }

  // Cleanup
  execSync(`rm "${tgzFile}"`)
} catch (error) {
  errors.push('Failed to create package archive: ' + error.message)
}

// Output results
if (errors.length > 0) {
  console.log('\n❌ VALIDATION FAILED:')
  errors.forEach(err => console.log(`  - ${err}`))
  console.log('\nFix these issues before publishing.')
  process.exit(1)
}

console.log('✅ Package validation passed!')
```

### 5.3.4 Regular Security Scanning

**Periodic CI Job:**

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  schedule:
    # Run every Monday at 2 AM
    - cron: '0 2 * * 1'
  workflow_dispatch:  # Allow manual trigger

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Dependency vulnerability scan
      - name: npm audit
        run: npm audit --audit-level=low
        continue-on-error: true

      # Snyk scan
      - name: Snyk test
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

      # CodeQL scan
      - name: CodeQL Analysis
        uses: github/codeql-action/analyze@v2
        with:
          languages: javascript

      # Dependency review
      - name: Dependency Review
        uses: actions/dependency-review-action@v3
        with:
          fail-on-severity: moderate

      # Results report
      - name: Report results
        if: always()
        run: |
          echo "Security scan completed"
          echo "Review the results in the Actions tab"
```

---

## 5.4 Package Publishing Validation Process

### 5.4.1 Pre-Publish Checklist

**Complete Pre-Publish Checklist:**

```markdown
## 📋 Pre-Publish Checklist

### 🔒 Security Checks
- [ ] No source maps with `sourcesContent` in dist/
- [ ] No .map files included in package
- [ ] No test files (.test.js, .spec.js) in package
- [ ] No secrets (API keys, tokens) in code
- [ ] Environment variables properly validated
- [ ] `.npmignore` correctly configured
- [ ] `package.json` files field is minimal

### 🏗️ Build Verification
- [ ] Build completes without errors
- [ ] All tests pass (`npm test`)
- [ ] Linting passes (`npm run lint`)
- [ ] TypeScript checks pass
- [ ] Bundle size is reasonable
- [ ] No unexpected files in dist/

### 📦 Package Validation
- [ ] `npm pack --dry-run` succeeds
- [ ] Package contents verified
- [ ] Dependencies are up-to-date
- [ ] License is correct
- [ ] Version number follows semver
- [ ] Changelog updated

### 🧪 Functional Testing
- [ ] Install from tarball works: `npm install ./package-1.0.0.tgz`
- [ ] Basic functionality works
- [ ] No runtime errors
- [ ] Performance is acceptable

### 📝 Documentation
- [ ] README is up-to-date
- [ ] API documentation is current
- [ ] Examples work correctly
- [ ] Migration guide (if breaking changes)

### 🚀 Post-Publish Verification
- [ ] Package is visible on npm
- [ ] Can install from npm: `npm install package@version`
- [ ] Installed package works correctly
- [ ] No source maps in installed package
```

### 5.4.2 Automated Publishing Script

**publish.sh:**

```bash
#!/bin/bash
set -e  # Exit immediately on any error

VERSION=$(node -p "require('./package.json').version")
PACKAGE_NAME=$(node -p "require('./package.json').name")

echo "🚀 Preparing to publish ${PACKAGE_NAME}@${VERSION}"
echo ""

# Color output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Check function
check_step() {
  local description=$1
  local command=$2

  echo -n "Checking: $description... "

  if eval "$command" > /dev/null 2>&1; then
    echo -e "${GREEN}✓${NC}"
    return 0
  else
    echo -e "${RED}✗${NC}"
    return 1
  fi
}

# Checklist
echo "Running pre-publish checks..."
echo ""

# 1. Git status
check_step "Clean git status" "git diff --quiet"
check_step "On main branch" "[ $(git branch --show-current) = 'main' ]"

# 2. Build validation
check_step "Build succeeds" "npm run build"
check_step "Tests pass" "npm test -- --passWithNoTests"
check_step "Lint passes" "npm run lint"

# 3. Security checks
check_step "No source maps in dist" "! find dist -name '*.map' | grep -q ."
check_step "Build validation passes" "node scripts/validate-build.js"

# 4. Package validation
echo ""
echo "Creating dry-run package..."
npm pack --dry-run > /dev/null 2>&1
TGZ_FILE=$(ls *.tgz 2>/dev/null | head -1)

if [ -z "$TGZ_FILE" ]; then
  echo -e "${RED}Error: Failed to create package${NC}"
  exit 1
fi

echo "Package: $TGZ_FILE"

# Check package contents
echo ""
echo "Validating package contents..."

if tar -tzf "$TGZ_FILE" | grep -E '\.map$'; then
  echo -e "${RED}Error: Source maps found in package!${NC}"
  rm "$TGZ_FILE"
  exit 1
fi

if tar -tzf "$TGZ_FILE" | grep -E '\.(test|spec)\.'; then
  echo -e "${YELLOW}Warning: Test files found in package${NC}"
  read -p "Continue anyway? (y/N) " -n 1 -r
  echo
  if [[ ! $REPLY =~ ^[Yy]$ ]]; then
    rm "$TGZ_FILE"
    exit 1
  fi
fi

# 5. Version check
echo ""
CURRENT_VERSION=$(npm view "$PACKAGE_NAME" version 2>/dev/null || echo "0.0.0")

if [ "$VERSION" = "$CURRENT_VERSION" ]; then
  echo -e "${YELLOW}Warning: Version $VERSION already published${NC}"
  read -p "Publish anyway? (y/N) " -n 1 -r
  echo
  if [[ ! $REPLY =~ ^[Yy]$ ]]; then
    rm "$TGZ_FILE"
    exit 1
  fi
fi

# 6. Confirm publish
echo ""
echo "========================================"
echo "Ready to publish:"
echo "  Package: $PACKAGE_NAME"
echo "  Version: $VERSION"
echo "  Current: $CURRENT_VERSION"
echo "========================================"
echo ""
read -p "Publish to npm? (y/N) " -n 1 -r
echo

if [[ ! $REPLY =~ ^[Yy]$ ]]; then
  echo "Cancelled"
  rm "$TGZ_FILE"
  exit 0
fi

# 7. Publish
echo ""
echo "Publishing..."
npm publish

# 8. Post-publish validation
echo ""
echo "Verifying published package..."
TEST_DIR="/tmp/npm-verify-$PACKAGE_NAME-$VERSION"
rm -rf "$TEST_DIR"
mkdir -p "$TEST_DIR"
cd "$TEST_DIR"

npm install "$PACKAGE_NAME@$VERSION"

if find "$TEST_DIR/node_modules/$PACKAGE_NAME" -name '*.map' | grep -q .; then
  echo -e "${RED}Error: Source maps found in published package!${NC}"
  echo "Please unpublish immediately: npm unpublish $PACKAGE_NAME@$VERSION --force"
  exit 1
fi

echo -e "${GREEN}✓ Package verification passed${NC}"

# Cleanup
cd -
rm -rf "$TEST_DIR"
rm "$TGZ_FILE"

echo ""
echo -e "${GREEN}========================================${NC}"
echo -e "${GREEN}✓ Successfully published $PACKAGE_NAME@$VERSION${NC}"
echo -e "${GREEN}========================================${NC}"
```

### 5.4.3 Post-Publish Monitoring

**Script to Monitor Published Packages:**

```javascript
// scripts/post-publish-monitor.js
const https = require('https')

async function checkPackage(packageName, version) {
  console.log(`Checking ${packageName}@${version}...`)

  // 1. Check if package is visible on npm
  const metadata = await fetchNpmMetadata(packageName)

  if (!metadata.versions[version]) {
    console.log('❌ Package version not found on npm')
    return false
  }

  console.log('✓ Package is on npm')

  // 2. Check package tarball contents
  const tarballUrl = metadata.versions[version].dist.tarball
  console.log(`Tarball: ${tarballUrl}`)

  const tarballContent = await fetchUrl(tarballUrl)

  // 3. Download and validate
  const { execSync } = require('child_process')
  const fs = require('fs')
  const path = require('path')

  const tempDir = `/tmp/verify-${packageName}-${version}`
  const tgzPath = path.join(tempDir, 'package.tgz')

  fs.mkdirSync(tempDir, { recursive: true })
  fs.writeFileSync(tgzPath, tarballContent)

  // 4. Check contents
  execSync(`cd ${tempDir} && tar -xzf package.tgz`)

  const packageDir = path.join(tempDir, 'package')

  // 4.1 Check for source maps
  const mapFiles = execSync(`find ${packageDir} -name '*.map'`)
    .toString()
    .trim()
    .split('\n')
    .filter(Boolean)

  if (mapFiles.length > 0) {
    console.log('❌ Source maps found in published package:')
    mapFiles.forEach(f => console.log(`  - ${f}`))
    return false
  }

  console.log('✓ No source maps found')

  // 4.2 Check for sourcesContent
  const jsFiles = execSync(`find ${packageDir} -name '*.js'`)
    .toString()
    .trim()
    .split('\n')
    .filter(Boolean)

  let hasSourcesContent = false
  jsFiles.forEach(file => {
    const content = fs.readFileSync(file, 'utf-8')
    // Check if source map is referenced
    const sourceMappingURL = content.match(/\/\/# sourceMappingURL=(.+\.map)/)
    if (sourceMappingURL) {
      console.log(`  Found sourceMappingURL: ${sourceMappingURL[1]}`)
    }
  })

  // 4.3 Cleanup
  execSync(`rm -rf ${tempDir}`)

  return true
}

function fetchNpmMetadata(packageName) {
  return new Promise((resolve, reject) => {
    const url = `https://registry.npmjs.org/${packageName}`

    https.get(url, (res) => {
      let data = ''

      res.on('data', chunk => data += chunk)
      res.on('end', () => resolve(JSON.parse(data)))
    }).on('error', reject)
  })
}

function fetchUrl(url) {
  return new Promise((resolve, reject) => {
    https.get(url, (res) => {
      const chunks = []

      res.on('data', chunk => chunks.push(chunk))
      res.on('end', () => resolve(Buffer.concat(chunks)))
    }).on('error', reject)
  })
}

// Main function
async function main() {
  const packageJson = require('../package.json')
  const packageName = packageJson.name
  const version = packageJson.version

  console.log('Post-Publish Monitoring')
  console.log('========================\n')

  const success = await checkPackage(packageName, version)

  if (success) {
    console.log('\n✅ All checks passed!')
    process.exit(0)
  } else {
    console.log('\n❌ Checks failed!')
    console.log('Action required: Review and potentially unpublish')
    process.exit(1)
  }
}

main()
```

### 5.4.4 Incident Response Plan

**If a Problematic Package is Published:**

**Emergency Unpublish Steps:**

```bash
#!/bin/bash
# unpublish-package.sh - Emergency unpublish

PACKAGE_NAME=$1
VERSION=$2

if [ -z "$PACKAGE_NAME" ] || [ -z "$VERSION" ]; then
  echo "Usage: ./unpublish-package.sh PACKAGE_NAME VERSION"
  exit 1
fi

echo "⚠️  WARNING: You are about to UNPUBLISH a package!"
echo "Package: $PACKAGE_NAME"
echo "Version: $VERSION"
echo ""
echo "This action should only be taken in emergencies:"
echo "  - Source code exposure"
echo "  - Security vulnerabilities"
echo "  - Critical bugs"
echo ""
read -p "Continue? (type 'yes' to confirm): " confirm

if [ "$confirm" != "yes" ]; then
  echo "Cancelled"
  exit 0
fi

# Unpublish
npm unpublish "$PACKAGE_NAME@$VERSION" --force

# Verify
if npm view "$PACKAGE_NAME@$VERSION" 2>/dev/null; then
  echo "❌ Unpublish may have failed"
  echo "Verify at: https://www.npmjs.com/package/$PACKAGE_NAME"
else
  echo "✓ Package unpublished successfully"
fi

# Next steps
echo ""
echo "Next steps:"
echo "1. Fix the issue"
echo "2. Bump version (patch/minor/major)"
echo "3. Re-run pre-publish checks"
echo "4. Publish new version"
echo "5. Notify users (if public)"
```

**Notification Template:**

```markdown
## Security Notice: Package [PACKAGE]@[VERSION] Withdrawn

### Summary
We have withdrawn version [VERSION] of [PACKAGE] due to:
- [ ] Source code exposure
- [ ] Security vulnerability
- [ ] Critical bug

### Impact
If you have installed this version:
```bash
npm install [PACKAGE]@[VERSION]
```

You should:
1. Uninstall the affected version
2. Update to the latest version
3. Audit your projects for any issues

### Remediation
```bash
# Uninstall affected version
npm uninstall [PACKAGE]

# Install safe version
npm install [PACKAGE]@[SAFE_VERSION]
```

### Timeline
- Issue discovered: [DATE]
- Package withdrawn: [DATE]
- Fixed version released: [DATE]

### Contact
For questions: [SECURITY_EMAIL]
```

---

## Part 5 Summary

### Core Security Points

| Layer | Key Measures | Tools/Methods |
|-------|-------------|---------------|
| **Source Map** | Disable sourcesContent | build config, validation scripts |
| **Build Config** | Separate source maps, minification | Bun/Webpack/Vite config |
| **CI/CD** | Multi-layer checks, auto-blocking | GitHub Actions, pre-commit hooks |
| **Publish Process** | Checklists, validation scripts | publish.sh, post-publish monitor |

### Security Checklist

**At Build Time (Required):**
- ✅ Source map doesn't contain sourcesContent
- ✅ No .map files in dist/
- ✅ Code minified and obfuscated
- ✅ Secret scanning passes

**Before Publishing (Required):**
- ✅ npm pack --dry-run validation
- ✅ Package content check passes
- ✅ All tests pass
- ✅ Dependency vulnerability scan passes

**After Publishing (Recommended):**
- ✅ Verify published package
- ✅ Monitor security reports
- ✅ Prepare incident response

### Design Patterns Summary

**Security Patterns (5 types):**
1. **Defense-in-Depth Build** - Least privilege principle
2. **Multi-Layer Validation** - CI/CD checkpoints
3. **Automated Checks** - pre-commit/prepublish hooks
4. **Post-Publish Verification** - Monitoring and auditing
5. **Incident Response** - Rapid unpublish and fix

---

**Word Count Estimate:**
- Chinese: ~4,000 words
- English: ~2,500 words (corresponding English version)

**Next Part:** Part 6 - Code Reading Practice
