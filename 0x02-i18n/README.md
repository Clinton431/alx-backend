# python-i18n

A lightweight, easy-to-use internationalization (i18n) library for Python applications. This package simplifies the process of making your Python application available in multiple languages.

## Features

- Simple, intuitive API
- Support for multiple translation file formats (YAML, JSON, Python dictionaries)
- Locale fallbacks
- Interpolation support
- Pluralization support
- Lazy translation
- Thread-safe
- No external dependencies

## Installation

```bash
pip install python-i18n
```

## Quick Start

### 1. Create translation files

Create YAML files in a `translations` directory:

**translations/en.yaml**:
```yaml
greeting: "Hello, %{name}!"
items:
  zero: "No items"
  one: "One item"
  other: "%{count} items"
```

**translations/es.yaml**:
```yaml
greeting: "¡Hola, %{name}!"
items:
  zero: "Ningún artículo"
  one: "Un artículo"
  other: "%{count} artículos"
```

### 2. Configure and use translations

```python
import i18n

# Configure the library
i18n.load_path.append('translations')
i18n.set('filename_format', '{locale}.{format}')
i18n.set('locale', 'en')  # Default locale

# Use translations
print(i18n.t('greeting', name='World'))  # Output: Hello, World!

# Change locale
i18n.set('locale', 'es')
print(i18n.t('greeting', name='Mundo'))  # Output: ¡Hola, Mundo!

# Pluralization
print(i18n.t('items', count=0))  # Output: Ningún artículo
print(i18n.t('items', count=1))  # Output: Un artículo
print(i18n.t('items', count=5))  # Output: 5 artículos
```

## Configuration

The library can be configured with the following settings:

```python
# Set the current locale
i18n.set('locale', 'en')

# Set the fallback locale (used when a translation is missing)
i18n.set('fallback', 'en')

# Add directories to search for translation files
i18n.load_path.append('path/to/translations')

# Configure the format of translation files
i18n.set('filename_format', '{locale}.{format}')

# Set the file format (yaml, json, or dict)
i18n.set('file_format', 'yaml')

# Enable/disable available locales
i18n.set('available_locales', ['en', 'es', 'fr', 'ja'])

# Enable namespace separator (for nested translations)
i18n.set('namespace_separator', '.')

# Set error behavior - throw exceptions (default) or return missing key
i18n.set('error_on_missing_translation', True)
```

## Advanced Usage

### Namespaces

You can organize translations using namespaces:

**translations/en.yaml**:
```yaml
user:
  greeting: "Hello, %{name}!"
  messages:
    new: "You have %{count} new messages"
```

Access with:
```python
i18n.t('user.greeting', name='Alice')
i18n.t('user.messages.new', count=3)
```

### Lazy Translation

For cases where you need to define translations before the actual locale is determined:

```python
from i18n import lazy

greeting = lazy('greeting', name='World')
# Later, when locale is set:
print(greeting())  # Will use the current locale at the time of call
```

### Pluralization

Advanced pluralization rules:

```python
# Configuration for pluralization rules
i18n.set('plural_rules', {
    'en': lambda n: 'one' if n == 1 else 'other',
    'ru': lambda n: 'one' if n % 10 == 1 and n % 100 != 11 else 'few' if 2 <= n % 10 <= 4 
          and (n % 100 < 10 or n % 100 >= 20) else 'many'
})

# Then in translations
# ru.yaml
items:
  one: "%{count} предмет"
  few: "%{count} предмета"
  many: "%{count} предметов"
  other: "%{count} предметов"
```

### Loading Translations Programmatically

```python
translations = {
    'en': {
        'welcome': 'Welcome',
        'goodbye': 'Goodbye'
    },
    'fr': {
        'welcome': 'Bienvenue',
        'goodbye': 'Au revoir'
    }
}

i18n.set('translations', translations)
```

### Thread Safety

The library is thread-safe by default. Locale settings are stored per-thread.

```python
import threading

def worker(locale):
    i18n.set('locale', locale)
    print(i18n.t('greeting', name='Thread'))

threads = [
    threading.Thread(target=worker, args=('en',)),
    threading.Thread(target=worker, args=('es',))
]

for t in threads:
    t.start()
```

## Command-line Tools

Extract translatable strings from Python files:

```bash
python -m i18n.extract -o translations/messages.pot your_module/*.py
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Acknowledgments

- Inspired by Ruby's I18n gem
- Thanks to all contributors who have helped make this project better
