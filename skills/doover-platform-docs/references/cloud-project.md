# Cloud Project Setup

## Project Structure

### Processor

```
my-processor/
├── src/my_processor/
│   ├── __init__.py       # Handler entry point
│   ├── application.py    # Main application logic
│   ├── app_config.py     # Configuration schema
│   └── app_ui.py         # UI components (optional)
├── doover_config.json    # App metadata
├── build.sh              # Build script for deployment package
└── pyproject.toml
```

### Integration

```
my-integration/
├── src/my_integration/
│   ├── __init__.py       # Handler entry point
│   ├── application.py    # Main application logic
│   └── app_config.py     # Configuration schema
├── doover_config.json    # App metadata
├── build.sh              # Build script for deployment package
└── pyproject.toml
```

## Build Script (Deployment Package)

Processors and integrations are deployed as a zip package. Add a build script to the project root:

```sh
#!/bin/sh

uv export --frozen --no-dev --no-editable --quiet -o requirements.txt

uv pip install \
   --no-deps \
   --no-installer-metadata \
   --no-compile-bytecode \
   --python-platform x86_64-manylinux2014 \
   --python 3.13 \
   --quiet \
   --target packages_export \
   --refresh \
   -r requirements.txt

rm -f package.zip
mkdir -p packages_export
cd packages_export
zip -rq ../package.zip .
cd ..

zip -rq package.zip src

echo "OK"
```

This exports locked dependencies, installs them for the Lambda runtime platform, zips the installed packages, then adds your `src` tree.

Make the script executable:

```bash
chmod +x build.sh
```

Add the build outputs to `.gitignore`:

```
packages_export/
package.zip
requirements.txt
```

Run the script before publishing so the correct `package.zip` is used.

## Removing Docker Files

If the project was created from the device app template, remove Docker-specific files:
- Remove `Dockerfile` (processors/integrations deploy as zip packages, not containers)
- Remove `.github/workflows/build-image.yml` if present
- Remove `simulators/` directory if present

## Best Practices

### Keep Invocations Short

Lambda has timeout limits. For long operations:
- Break into multiple scheduled invocations
- Use tags to track progress
- Consider device apps for continuous processing

### Handle Cold Starts

Each invocation may be a cold start:
- Initialize efficiently in `setup()`
- Don't rely on in-memory caching
- Use tags for persistent state

### Idempotent Handlers

Messages may be delivered multiple times:
- Track processed message IDs in tags
- Design handlers to be safely re-runnable

### Error Handling

```python
async def on_message_create(self, event):
    try:
        await self.process(event)
    except Exception as e:
        # Log error
        await self.set_tag("last_error", str(e))
        # Don't re-raise unless you want retry
        raise
```

## Exporting Config

Generate doover_config.json:

```python
# In app_config.py
from pathlib import Path

def export():
    MyAppConfig().export(
        Path(__file__).parents[2] / "doover_config.json",
        "my_app"
    )

if __name__ == "__main__":
    export()
```

In pyproject.toml:

```toml
[project.scripts]
export-config = "my_app.app_config:export"
```
